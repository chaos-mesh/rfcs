# Physical Machine Auth

## Summary

PhysicalMachineChaos-based auth solution, including end-user authorization
and service-to-service authentication.

## Motivation

Chaos Dashboard's Authentication and Authorization scheme is only applicable
to single-cluster Kubernetes, and the Chaos Mesh platform needs to be adapted
to more scenarios, such as multi-cluster Kubernetes, physical machines, cloud
infrastructure, etc.

### Goals

- end-user authorization on physical machine chaos
- service-to-service authentication on physical machine chaos

### Non-Goals

- end-user authentication on physical machine chaos (same with other chaos
which is using token)

## Detailed design

### End-user authorization

#### New Custom Resource: `PhysicalMachine`

Because there is no concept of pod to filter the target physical machine
when performing chaos injection on a physical machine, a resource needs to
be introduced to represent the physical machine, which is the new custom
resource `PhysicalMachine`. Here is a sample of `PhysicalMachine`.

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PhysicalMachine
metadata:
  name: pm-172.16.112.130
  namespace: chaos-testing
  labels:
    chaos-mesh/physical-machine-group: abc
    kubernetes.io/arch: arm64
    kubernetes.io/os: linux
spec:
  address: https://172.16.112.130:31767
```

#### Changes to `PhysicalMachineChaos`

Remove the `address` field in `PhysicalMachineChaos`, and use the `selector`
field to filter the injection range of the experiment.

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PhysicalMachineChaos
metadata:
  name: physical-network-delay
  namespace: chaos-testing
spec:
  action: network-delay
  network-delay:
    device: “ens33”
    hostname: “baidu.com”
    duration: "5s"
  selector:
    namespaces: 
      - testA
    labelSelectors:
      chaos-mesh/physical-machine-group: abc
```

#### RBAC

According to the above design, there is no need to change the existing
`Role`/`ClusterRole` content, because the access to `PhysicalMachine` is
included in the api group of `chaos-mesh.org`.

We have two different ranges of access rights:

- clustered scope: permissions for `PhysicalMachines` in all namespaces,
so you can experiment with all physical machines.
- namespaced scope：permissions for the `PhysicalMachine` in the current
namespace, and you can experiment with multiple physical machines in that
namespace.

### service-to-service authentication

To secure the service-to-service communication, establish an mTLS connection
between chaos-controller-manager and chaosd. 

The following certificates are required:

- CA certificate: generated when deploying `Chaos Mesh` with `helm` or
`instal.sh`, and saved in the secret named
`chaos-mesh-controller-manager-client-certs` (and on each physical machine)
- certificate of chaos-controller-manager side: generated when deploying
`Chaos Mesh` with `helm` or `instal.sh`, and saved in the secret named
`chaos-mesh-controller-manager-client-certs`
- certificate of chaosd side: generated automatically or manually when adding
physical machine information to the cluster, saved on each physical machine

#### User Cases

Here is a description of three different usage scenarios of user cases.

#### Case1: Automatically generate certificates

Prerequisites:

1. User deployed `Chaos Mesh` in Kubernetes cluster with security mode
1. The node executing the `chaosctl` command can ssh to
the target physical machine
1. The node executing the `chaosctl` command can access the Kubernetes cluster

Steps:

1. User prepares the physical machine using `chaosctl`, the command might be
`chaosctl physical-machine init --server=127.0.0.1 --port=2333`, in this step,
`chaosctl` generates the certificates on the physical machine side and copies
all the required certificates to the target physical machine, then create the
`PhysicalMachine` CR in Kubernetes cluster (BTW, `chaosctl pm` can be used
instead of `chaosctl physical-machine`)
1. User starts chaosd service on the physical machine
1. User creates a physical machine experiment on the dashboard
1. Chaos-controller-manager establishes an mTLS connection when requesting
the chaosd service on the physical machine

#### Case2: Manually generate certificates

Prerequisites:

1. User deployed `Chaos Mesh` in Kubernetes cluster with security mode
1. The node executing the `chaosctl physical-machine create` command can
access the Kubernetes cluster

Steps:

1. User copies the CA certificate from the kubernetes cluster
to the physical machine
1. User uses `chaosctl` to generate the certificates on the physical machine,
the command might be `chaosctl physical-machine generate`
1. User uses `chaosctl` to create `PhysicalMachine` resource in Kubernetes
cluster, the command might be
`chaosctl physical-machine create --server=127.0.0.1 --port=2333`
1. User starts chaosd service on the physical machine
1. User creates a physical machine experiment on the dashboard
1. Chaos-controller-manager establishes an mTLS connection when requesting
the chaosd service on the physical machine

#### Case3: Without mTLS authentication (Not recommended)

Prerequisites:

1. User deployed `Chaos Mesh` in Kubernetes cluster without security mode

Steps:

1. User uses `chaosctl` to create `PhysicalMachine` resource in Kubernetes
cluster, the command might be 
`chaosctl physical-machine create --server=127.0.0.1 --port=2333 --protocol=http`
1. User starts chaosd service on the physical machine
1. User creates a physical machine experiment on the dashboard
1. Chaos-controller-manager will use HTTP to request the chaosd service

## Drawbacks

Because `PhysicalMachine` is a namespaced scope resource, users can create
the same physical machine information on different namespaces, which may
cause duplicate injection problems and  it requires the chaosd service
API to support idempotency.

## Alternatives

NA

## Unresolved questions

### Not support a role to access multiple specified `PhysicalMachines`

One option considered is to directly use the `resourceNames` field in
Kubernetes RBAC to control access by specifying the `name` field of the
PhysicalMachine resource. However, in practice, we found that `resourceNames`
only supports `GET` and `DELETE` APIs, not `LIST`, `WATCH`, `CREATE` and
`DELETECOLLECTION` APIs, which makes it impossible for users to select the
physical machine they want when operating on the dashboard. If there is a
practical need, we may use `OPA`, `gatekeeper` and other policy frameworks to
control the fine-grained permissions more easily.
