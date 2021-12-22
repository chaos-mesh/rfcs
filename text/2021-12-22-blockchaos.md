# Block schedule injection

## Background

Current `IOChaos` injection works on file system layer. It will setup a FUSE and
inject delay / corrupt into the request. This solution can have its own
advantages. Meanwhile, the disadvantages also bothers us a lot.

Advantages:

1. It will never have chance to break the host system, and is relatively safe.
2. Have knowledge about the file system, which means it can filter through path,
   owner, or anything which only occurs upon the file system.

Disadvantages:

1. The performance impact is really big even without injecting any delay, which
   makes it impossible to do quantitative analysis.
2. The target process will be paused at the beginning of the injecting. The time
   may differ from 1s to 4s according to the openned files.

In this RFC, I will propose a new mechanism to inject delay, IOPS limitation (or
corrupt, in the future) to a block device. The implementation is split into a
standalone repo:
[chaos-mesh/chaos-driver](https://github.com/chaos-mesh/chaos-driver). And the
Chaos Mesh will convert an injection request on Pod to an injection request on
block device to the chaos-driver.

## Proposal

The implementation can be saw in three parts:

1. Chaos Mesh. Reconcile the `BlockChaos` resource, select related PV and node,
   and send the request to the corresponding chaos-daemon.
2. Chaos Daemon. It reads the grpc request (for injection and recover), and
   then use the client provided by chaos-driver to talk with `/dev/chaos`.
3. Chaos Driver. It handles the operation on `/dev/chaos`, and add the injection
   on the injection list.

### Chaos Mesh

The scheme of `BlockChaos` is generally like:

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: BlockChaos
metadata:
  name: block-chaos
spec:
  mode: one
  selector:
    labelSelectors:
      "app.kubernetes.io/component": "tikv"
  volumeName: tikv
  action: limit
  iops: 5000
status:
  ...
  ids:
    default/tikv-0/tikv: 1
```

The record will be like: `default/tikv-0/tikv`. The first part of it is the
namespace, the second part of it is the pod name, and the third one is the
volume name.

To inject a chaos, the Chaos Mesh will read the `Pod`, find the corresponding
volume entry in the `Pod` spec, and then get the `PVC`. By reading
`PVC.spec.volumeName`, the controller can get the `PV`.

If the source of the `PV` is `LocalVolume` or `HostPathVolume` (which means
either `LocalVolumeSource` or `HostPathVolumeSource` is not empty), this PV can
be injected. The controller can get a path, which represents a directory on host
or a block device on host.

Then the controller can send this path and injecting spec to the chaos-daemon on
corresponding node, and will receive an id from the chaos-daemon.

To recover a chaos, the controller only needs to send the id to related
chaos-daemon.

#### Requirement

1. The volume entry of the `Pod` must be `PVC`.
2. The source of the `PV` must be `LocalVolume` or `HostPath`. `VolumeMode` must
   be `FileSystem`

### Chaos Daemon

The chaos daemon can use the go client provided by chaos-driver directly.

Before injecting the chaos, the chaos daemon should make sure the `chaos_driver`
module has been loaded.

#### Get device from path

The chaos daemon should get a block device from the path.

If the path is a directory, then we can read the `/proc/mounts` and find the
mount source of the path. If the mount source is still a directory, which means
this mount is a bind mount, we can handle this situation recursively until we
get a block device. If it's neither a block device nor a directory, it's an
invalid value.

This device can also be a partition: `sda1` or `nvme0n1p1`. If the device name
doesn't have a corresponding directory in `/sys/block`, it is a partition. We
can get the parent device of it through `basename /sys/class/block/%s/..`. For
example, the `/sys/class/block/nvm0n1p1` is a soft link to
`../../devices/pci0000:00/0000:00:01.1/0000:01:00.0/nvme/nvme0/nvme0n1/nvm0n1p1`,
then the `..` of it is `nvme0n1`.

#### Inject

After getting the block device, the chaos daemon should read
`/sys/block/%s/queue/scheduler` to make sure it is `ioem` or `ioem-mq`, then it
can be injected through `github.com/chaos-mesh/chaos-driver/pkg/client`:

```go
id, err := c.InjectIOEMDelay(dev_path, op, pidNs, delay, corr, jitter)
```

The id should be returned to the controller. The `dev_path` is `/dev/xxx`, the
`pidNs` is the pid of the container, and the `delay`, `corr`, `jitter` is the
corresponding value in the chaos spec.

#### Recover

Recover doesn't need the device path:

```go
err = c.Recover(id)
```

### Chaos Driver

Chaos Driver is designed for many various kinds of injection. In this RFC, only
IOEM (which is the io scheduler designed for injecting delay and limit) will be
discussed.

The IOEM scheduler has two main structure: `ioem_data` and `irl`. The
`ioem_data` contains two data structure: one linked list (called
"waiting_queue") and an rb_tree. The `irl` means "IOEM Request Limit", which is
used to limit the IOPS.

Every hardware queue will be binded by one `ioem_data`, every hardware will be
binded by one `irl`. For multiqueue scheduler, one `irl` may be shared by
multiple `ioem_data`.

#### IRL

The design of IRL is quire simple: it contains two fields, one for `quota` and
one for `counter`, and it has one method: "dispatch". Everytime this functions
is called, the counter will increase by one, and if the counter is still smaller
than the `quota`, it will return successfully, or it will return a false.

A hrtimer will be setuped to reset the `counter` to 0 for every period.

#### IOEM DATA

An IO scheduler has two main functions: insert request and dispatch request.

##### Insert Request

When a request reaches the scheduler, it will iterate through the registered
injections to see whether this request should be affected by the limit and
delay.

If this request should be injected with a delay, then the `time_to_send` of this
request is marked as `now + delay`, or it will be marked as `now`.

If  this request should be injected by the limit, the `ioem_limit_should_affect`
will be marked as `true`, otherwise `false`.

Then this request will be inserted into the rbtree, sorted with the
`time_to_send`.

##### Dispatch

The dispatch will be triggered by hardware driver, background task, the
scheduler itself, etc... When the dispatch is called, the scheduler can dispatch
one (or zero) request to the hardware.

The scheduler will check the first entry of `waiting_list`, to see whether there
are requests waiting for the limit quota, and whether the quota is enough to
send a request. If both quota and request is availabe, return the request.

If either the quota or request is unavailable, the scheduler will iterate the
`rbtree` until the `time_to_send` is greater than now. For any request whose
`time_to_send` is smaller than now, it can be tried to dispatch through `irl`
(or directly, if it's not affected by the limit).

If the quota is not enough to send the request in rbtree, pop it out and
reinsert into the waiting queue.

If no request is sent in one dispatch, but there are still request inside the
rb_tree or waiting list, the scheduler should calculate a earliest time when a
request could be sent. This time should be the next time when the counter in irl
is reset, or the smallest `time_to_send` in the rbtree. This time will be used
to setup a timer to trigger the dispatch.
