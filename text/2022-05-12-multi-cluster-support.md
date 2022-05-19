# Multi Cluster Support

## Summary

As the multi-cluster injection is a huge feature, and related with the full
lifecycle of Chaos Mesh, this RFC will be organized in a different way. It will
describe the design through the Deploy -> Usage -> Authentication ->
Architecture, and tell you the reason behind the design and other considerations
in every separated section in a Q&A form.

## Deploy

A new cluster scope resource RemoteCluster will be used to install and manage
the chaos-mesh installation on another cluster. This resource will be
responsible to install, configure, upgrade and uninstall the cluster.

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: RemoteCluster
metadata:
 name: cluster-xxxxxxx
spec:
 namespace: "chaos-mesh"
 kubeConfig:
   secretRef:
     name: “cluster-xxx-kubeconfig”
   key: xxx
 configOverride:
   chaosDaemon:
     runtime: containerd
status:
 currentVersion: "v2.1.0"
 conditions:
 - type: Installed
   status: True
 - type: Ready
   status: True
```

The `Spec` defines the cluster which we want to install the chaos mesh, and the
version of chaos mesh. It will install the same version of Chaos Mesh as the
controller’s, which means if the version of controller is v2.2.0, it will
install or upgrade the chaos mesh to v2.2.0 in the target cluster. The
`configOverride` allows to override the global configurations. The full
configuration is a combination of the default configuration, the global
configuration (provided through a constant configmap) and the override. We
should provide an eliminated default configuration with the controller, as there
are too many things not needed in the child chaos mesh.

The `Status` shows the current status of the target cluster. All of this field
can be read from the target cluster through the helm package or read the pods’
condition from the target cluster. The condition `Installed` is True iff the
release has been installed in the target cluster and namespace. The condition
`Ready` is True iff all pods are ready. The `currentVersion` in the status gives
us a chance to automatically migrate the configuration.

The total workflow can be described in the following graph:

```
User creates RemoteCluster -> RemoteCluster controller Reconcile

User removes the RemoteCluster -> RemoteCluster controller Reconcile

Pods in the target cluster/namespace changed -> RemoteCluster controller Reconcile (resources will be mapped to the `RemoteCluster`)

RemoteCluster controller Reconcile
-> If the resource is being deleted
   -> Helm list the installed release
   -> If the chaos-mesh chart is not installed, and there is no chaos using this cluster
      remove the finalizer
   -> Apply the chaos and return
-> If the finalizer doesn't exist, add a finalizer
-> Helm list the installed release/chart
   -> If the chaos-mesh chart is not installed, and the `RemoteCluster` itself is not being 
      deleted, install the Chaos Mesh through helm and list the helm release again
   -> If the chaos-mesh chart is installed, `Installed` conditions turn to true, else, turn to false
-> If all pods of target Chaos Mesh are running, `Ready` conditions turn to true, else, turn to false
```

### Q&A

1. Why do we need to manage the installation of Chaos Mesh?

   Consider a normal SaaS provider, which is a typical user of Chaos Mesh, will
   need to automatically create Kubernetes clusters for his users. To use Chaos
   Mesh, he will also need to install Chaos Mesh in the cluster which is created
   automatically by the software. It’s inconvenient to deploy Chaos Mesh
   manually in this situation.

   Instead of every such user writing their own program to manage the
   installation of Chaos Mesh, we could provide an official way to install /
   upgrade / uninstall the Chaos Mesh.

2. Why do we define a new API to manage the deployment, rather than a go package
   or cli tools?

   Because the k8s API is just intuitive and easy to integrate, and it’s only
   one step further than a go package or cli tools.

3. Why is the `RemoteCluster` resource cluster scoped?

   Different *Chaos (e.g. NetworkChaos) will want to inject chaos in the same
   cluster. If the `RemoteCluster` is namespace scoped, and we have to allow
   selecting a remote cluster across the namespaces, then the namespace itself
   is meaningless.

4. Where will the controller exist?

   In the chaos-controller-manager.

5. What will it install?

   A normal Chaos Mesh, with some controllers (like dashboard, workflow,
   schedule) disabled.

## Inject

To inject chaos in another cluster, add a `cluster` and `namespace` field to
every selector. For example

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
 name: burn-cpu
spec:
 remoteCluster:
   cluster: cluster-xxxxxxx
   namespace: remote-cluster-test
 mode: one
 selector:
   labelSelectors:
     "app.kubernetes.io/component": "tikv"
 stressors:
   cpu:
     workers: 1
     load: 100
     options: ["--cpu 2", "--timeout 600", "--hdd 1"]
 duration: "30s"
```
 
For a chaos whose remoteCluster is not nil, the controller will synchronize this
resource with the target namespace in the target cluster. “synchronize” is
executed in two-ways: distribute the spec from the parent to child, and sync the
status from the child to parent. As we don’t support modifying the spec yet,
half of the “synchronize” is executed only once. The child chaos resource will
be created with some annotations to help the controller find the parent (in
parent cluster).

For implementation, the chaos mesh will set up multiple “Managers”, one for each
cluster. These managers will set up controllers to synchronize the things.

The creation, deletion is straightforward. Another thing to consider is the GC.
As there is nothing like `ownerReference` across the clusters, we will need to
handle the unexpected situation, e.g. the parent is deleted without
notification… We should also provide a way to force remove both parent and child
(maybe propagating the annotation is enough).

The total workflow can be described in the following graph:

```
User creates chaos -> if this chaos contain a `remoteCluster` field, the remote chaos controller reconcile

User delete chaos -> if this chaos contain a `remoteCluster` field, the remote chaos controller reconcile

Chaos in child cluster changed -> the remote chaos controller reconcile (resources will be mapped to the chaos in parent cluster)

Controller Reconcile -> If the Target Cluster is not ready, do nothing and return
   -> If the chaos is being deleted
      -> If the chaos in the child cluster doesn't exist, remove the finalizer
      -> Apply and return
   -> If the finalizer doesn't exist, add a finalizer
   -> Get the chaos in child cluster
      -> If the chaos in the child cluster doesn't exist, create the chaos in the child cluster
      -> If the chaos in the child cluster exists, read the `Status` of the chaos in the child cluster
         and update the `Status` of the chaos in the parent cluster
```

### Q&A

1. Why should we install the chaos mesh controller in every cluster? Wouldn’t it
   be better to manage all chaos in a single cluster (even process)?

   It sounds great to listen and respond to every chaos in a single process,
   which means we will only need to install chaos daemon in the target cluster.
   However, the reality is that there is no common way to communicate with the
   chaos daemon in another cluster. I have thought about proxy, port-forward or
   other things, but nothing is reliable enough.

   I also don’t want to depend on the user to tell us the network topology: the
   clusters are in the same network / different network / proxy / ingress / … .
   It’s too hard for both of us and users. If we operate the chaos through
   another controller in the target cluster, we only need an available
   kubeconfig, which is much simpler than the network configuration.

2. Why is the `remoteCluster` an extra field? Would a standalone resource /
   annotation / … be better?

   I can’t say which one is better. It’s a blind choice. I don’t want to create
   a new embedded standalone resource because the existing two embedded
   resources: Schedule and Workflow are too huge (but it’s actually not a strong
   evidence). I prefer fields over annotations because fields sound more like an
   official API, but annotations do not (or maybe we could start from
   annotations? I’m not sure).

3. Can one chaos inject fault into different clusters?

   Obviously, no. I don’t think we can provide a general way to implement this
   in the near future.

4. Where will the webhook run?

   In the parent chaos mesh. Because we will not create chaos in the webhook
   (but in the reconcile), the validation should be done in the parent’s
   webhook.

5. What about chaos events?

   Thanks to @xlgao-zju , we have stored some events inside the chaos status, so
   we don’t need to synchronize the event between two clusters.

## Authentication / Authorization

The Chaos Mesh project has paid a lot of attention to Authentication and
Authorization. In most situations, we choose to follow the RBAC pattern. The
multi-cluster injection will also follow the RBAC pattern. Like what we have
done inside the validate_auth.go, if the remoteCluster is not nil, a webhook
will create a SubjectAccessReview to check whether the user is able to create
this object in the target cluster/namespace.

The only problem is that the username could be different among the clusters. To
solve this problem, we could provide a usermap configuration to allow users map
the usernames across different clusters. The usermap should be stored inside a
configmap. The cluster in userMap should allow globs.

```yaml
userMap:
- originalName: yangkeao
  cluster: *
  name: admin
```

We should also provide an option to stop the webhook (in helm), as it will truly
bring some inconvenience to the users. If the users don’t care about
Authentication and Authorization, they can turn off this feature.

### Q&A

1. Why don’t we build a standalone authentication solution?

   This sound never dimmed since the first day we were considering the
   authentication. From the subjective aspect, as a developer, I don’t want to
   build or ship a software with awful experience and strange configuration.
   From the objective aspect, as a user, I don’t want to learn or manage the
   authentication for every single application on my cluster. It would be a
   disaster to create users / teams / groups in every application. Obviously,
   the Chaos Mesh shouldn’t be the bad guy.

   If there are some needs which cannot be solved through RBAC, I don’t regard
   it as a common need (or a suitable need). e.g. you want to list every
   namespaces which you have privilege…

2. Why don't we store the user information inside the chaos?

   Yes. We could store the username inside the chaos annotation, and during the
   Reconcile, we impersonate the user. It is possible, but doesn’t solve the
   problem better. With webhook, we can tell the user that you are not
   privileged enough to create the resource at the creation time, which is
   really expected (like every resource in a single cluster).

3. What about the `Workflow` and `Schedule`?

   The `Workflow` can create chaos in different clusters, and it’s the only way
   to orchestrate the chaos in different clusters. We could get a list of
   affected clusters/namespaces, and evaluate the privilege.

4. What about creating a CRD for `userMap`?

   I think a ConfigMap is enough. A standalone CRD cannot bring more benefits.

5. `userMap` is ugly!

   Yes. It reminds me of uid_map in the linux user_namespace, which is also an
   ugly thing. If you have a better idea, please tell me.

## Architecture

The total architecture is really simple.

In the management (or so-called parent) cluster, the chaos-controller-manager
will be responsible for

1. deploying chaos mesh to other clusters
2. validating chaos
3. authorization
4. control workflow and schedule
5. synchronizing the chaos status
6. normal chaos injecting / recovering, as the user may want to inject to
   current management cluster

In the child cluster, the chaos-controller-manager and chaos-daemon will be
responsible for injecting / recovering chaos

These two chaos-controller-manager could share a single binary, so the
chaos-controller-manager could work in three different modes:

1. parent mode, like discussed in former section
2. child mode, like discussed in former section
3. single mode, just the normal one we are using now

(maybe these modes can be described through turning on/off some controllers /
webhooks)

## Dashboard

As the plan shown above, we don't need too much modification on dashboard. The
cluster installation status should be represented in `RemoteCluster`, and the
execution status of a standalone chaos should be recorded in the `.Status` of
every chaos. The `Workflow` and `Schedule` doesn't exist in the child cluster.

Given these design, all information about the execution of current chaos can be
read in the current (parent) cluster, and showed in the dashboard like the
normal chaos.

The only modification, is to use the events recorded in the chaos, rather than
the events in the kubernetes cluster. As the events in the chaos status and the
events in Kubernetes cluster are totally redundant now, we should peek one of
them.
