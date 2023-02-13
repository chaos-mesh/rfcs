# Extensible Chaosctl

## Summary

A tool to control the status of Chaos Mesh as much as possible in an extensible
way.

## Motivation

Currently, the chaosctl is a debug tool to collect logs and other information
from several kinds of chaos. However, some of its functions are implemented by
executing commands in the namespace of target pods through the chaos-daemon.

For example, in the iochaos debug part, the chaosctl executes `cat /proc/mounts`
[command](https://github.com/chaos-mesh/chaos-mesh/blob/4b8fb5ba1518fda0d144c8df9239dcb0381ff485/pkg/chaosctl/debug/iochaos/iochaos.go#L54)
straightly in the namepsace of target pods, which is dangerous and causes
develpment difficulties in the long time.

To implement more features for chaosctl, we must refactor it for more security
and extensibility.

## Detailed Design

For more security, functions like reading file or listing processes should be
implemented in the server side. In other words, we need an API server, and make
the chaosctl a pure client.

So, what APIs do we needs? To implement the features that current chaosctl
supports, maybe we can provide restful APIs like `/networkchaos/iptables` or
`/iochaos/mounts`. However, in some cases we may need `/networkchaos/iptables`
and in other cases we need `/networkchaos/ipset` or both of them, should we
provide API for each of them to avoid unsued information?

> I think the unsued information will reduce the debug efficiency very much.

On the other hand, there are too many kinds of resources related with each
other: the scheduler, the xxxchaos, the podcxxxchaos, the pod, the container and
the process, how can we provide all APIs with their arrangement?

> For example, how can we provide both of `/xxxchaos/podchaos` and
> `/podchaos/xxxchaos` conveniently?

To resolve these problems, we need structural APIs.

### Nested Resources

Nested resources, also called sub-resources, may accomplish our goals. For
example, if we register the struture of `networkchaos` resources, its
sub-resources like `iptables` and `ipset` will be registerd automatically. We
can access the networkchaos resources by path `/networkchaos` and access its
sub-resources by path `/networkchaos/iptables` or `/networkchaos/ipset`.

```go
type NetWorksChaos struct {
    Iptables []*IptablesRule
    Ipset    []*IpsetRule
}
```

However, there are two main drawbacks of this solution. Firstly, it can not
support cascade queries, we must regard `/networkchaos` and
`/networkchaos/<name>` as different resources. Secondly, there is almost no
library to conveniently build nested API servers in the ecosystem of golang.

### GraphQL

The GraphQL is one of the most famous solutions for structural APIs. In above
case, we can easy fetch iptables resources only by query `{ networkchaos
{iptables } }`. However, the main drawback of this solution is that queries are
not convenient to edit in cli, we need translation between resource paths and
GraphQL queries.

For example, we can translate resource path `/networkchaos/iptables` to query `{
networkchaos { iptables } }`.

### API Server

We can choose one of the nested resources and GraphQL as our API solution, I
prefer GraphQL as its ecosystem is better than nested resources.

#### Pod

The core resource we usually need is `Pod`. A `Pod` can represent both of the
target `Pod` to be injected and the `Pod` to run system components like
`chaos-daemon` or `chaos-controller-manager`. In the schema of GraphQL, we
should provide almost all sub-resources of `Pod`. Moreover, additional
sub-resources like `logs` and `processes` are also necessary.

Here is a schema of Pod:

```graphql
type Pod @goModel(model: "k8s.io/api/core/v1.Pod") {
    # metadata
    kind: String!
    apiVersion: String!
    name: String!
    generateName: String!
    namespace: String!
    selfLink: String!
    uid: String!
    resourceVersion: String!
    generation: Int!
    creationTimestamp: Time!
    deletionTimestamp: Time
    deletionGracePeriodSeconds: Int
    labels: Map
    annotations: Map
    ownerReferences: [OwnerReference!]
    finalizers: [String!]
    clusterName: String!

    # spec and status
    spec: PodSpec!
    status: PodStatus!

    # custom sub-resources
    logs: String!         @goField(forceResolver: true)
    daemon: Pod           @goField(forceResolver: true)
    processes: [Process!] @goField(forceResolver: true)
    mounts: [String!]     @goField(forceResolver: true)
    ipset: String!        @goField(forceResolver: true)
    tcQdisc: String!      @goField(forceResolver: true)
    iptables: String!     @goField(forceResolver: true)
}
```

The metadata, spec and status can be straightly mapped from `Pod` structure of
k8s api, but the custom sub-resources should be resolved by defined resolvers.

There is an example to resolve the `daemon`:

```go
func (r *podResolver) Daemon(ctx context.Context, obj *v1.Pod) (*v1.Pod, error) {
  var list v1.PodList
  if err := r.Client.List(ctx, &list, client.MatchingLabels(componentLabels(model.ComponentDaemon))); err != nil {
    return nil, err
  }
  for _, daemon := range list.Items {
    if obj.Spec.NodeName == daemon.Spec.NodeName {
      return &daemon, nil
    }
  }
  return nil, fmt.Errorf("daemon of pod(%s/%s) not found", obj.Namespace, obj.Name)
}
```

#### PodXXXChaos

`PodXXXChaos` is a relationship to describe one-to-many mapping between `Pod`
and `XXXChaos`, so we can just add sub-resource `pod` and `xxxChaos`:

```graphql
type PodIOChaos @goModel(model: "github.com/chaos-mesh/chaos-mesh/api/v1alpha1.PodIOChaos") {
    # metadata
    kind: String!
    apiVersion: String!
    name: String!
    generateName: String!
    namespace: String!
    selfLink: String!
    uid: String!
    resourceVersion: String!
    generation: Int!
    creationTimestamp: Time!
    deletionTimestamp: Time
    deletionGracePeriodSeconds: Int
    labels: Map
    annotations: Map
    ownerReferences: [OwnerReference!]
    finalizers: [String!]
    clusterName: String!

    # spec and status
    spec: PodIOChaosSpec!
    status: PodIOChaosStatus!

    # mappings
    pod: Pod!           @goField(forceResolver: true)
    ioChaos: [IOChaos!] @goField(forceResolver: true)
}
```

> There may be some virtual resources like `PodStressChaos`, you cannot find them
> in Chaos Mesh. Schemas of them do not contains metadata, spec or status:
>
> ```graphql
> type PodStressChaos {
>     stressChaos: [StressChaos!]
>
>     pod: Pod!
>     cgroups: Cgroups!               @goField(forceResolver: true)
>     processStress: [ProcessStress!] @goField(forceResolver: true)
> }
> ```

#### XXXChaos

`XXXChaos` contains metadata, spec, status and mapping to `PodXXXChaos`:

```graphql
type IOChaos @goModel(model: "github.com/chaos-mesh/chaos-mesh/api/v1alpha1.IOChaos") {
    kind: String!
    apiVersion: String!
    name: String!
    generateName: String!
    namespace: String!
    selfLink: String!
    uid: String!
    resourceVersion: String!
    generation: Int!
    creationTimestamp: Time!
    deletionTimestamp: Time
    deletionGracePeriodSeconds: Int
    labels: Map
    annotations: Map
    ownerReferences: [OwnerReference!]
    finalizers: [String!]
    clusterName: String!

    spec: IOChaosSpec!
    status: IOChaosStatus!

    podIOChaos: [PodIOChaos!] @goField(forceResolver: true)
}
```

### Client

#### Logs and Debug

The legacy chaosctl provides `logs` and `debug` subcommands, we should
re-implement them based on new ctrl server.

##### Subcommand `logs`

The `logs` subcommand is used to fetch and print logs of system
components(daemon, controller, dashboard and dns server). Here is the new
implementation:

```go
var query struct {
  Namespace []struct {
    Component []struct {
      Name string
      Spec struct {
        NodeName string
      }
      Logs string
    } `graphql:"component(component: $component)"`
  } `graphql:"namespace(ns: $namespace)"`
}

variables := map[string]interface{}{
  "namespace": graphql.String(managerNamespace),
  "component": name,
}

err := client.Client.Query(context.TODO(), &query, variables)
if err != nil {
  return err
}

if len(query.Namespace) == 0 {
  return fmt.Errorf("no namespace %s found", managerNamespace)
}

for _, component := range query.Namespace[0].Component {
  if component.Spec.NodeName != o.node {
    // ignore component on this node
    continue
  }
  PrettyPrint(component.Logs)
}
```

##### Subcommand `debug`

The `debug` subcommand is used to fetch chaos-related sub-resources from
injected pods. For example, we need `ip set`, `tc qdisc` and `iptables` rules
when debugging network chaos, here is the new implementation for network chaos:

```go
var query struct {
  Namespace []struct {
    NetworkChaos []struct {
      Name       string
      PodNetwork []struct {
        Spec      *v1alpha1.PodNetworkChaosSpec
        Namespace string
        Name      string
        Pod       struct {
          Ipset    string
          TcQdisc  string
          Iptables string
        }
      }
    } `graphql:"networkchaos(name: $name)"`
  } `graphql:"namespace(ns: $namespace)"`
}

variables := map[string]interface{}{
  "namespace": graphql.String(namespace),
  "name":      name,
}

err := client.Client.Query(ctx, &query, variables)
if err != nil {
  return nil, err
}

if len(query.Namespace) == 0 {
  return results, nil
}

for _, networkChaos := range query.Namespace[0].NetworkChaos {
  result := &common.ChaosResult{
    Name: networkChaos.Name,
  }

  for _, podNetworkChaos := range networkChaos.PodNetwork {
    podResult := common.PodResult{
      Name: podNetworkChaos.Name,
    }

    podResult.Items = append(podResult.Items, cm.ItemResult{Name: "ipset list", Value: podNetworkChaos.Pod.Ipset})
    podResult.Items = append(podResult.Items, cm.ItemResult{Name: "tc qdisc list", Value: podNetworkChaos.Pod.TcQdisc})
    podResult.Items = append(podResult.Items, cm.ItemResult{Name: "iptables list", Value: podNetworkChaos.Pod.Iptables})
    output, err := cm.MarshalChaos(podNetworkChaos.Spec)
    if err != nil {
      return nil, err
    }
    podResult.Items = append(podResult.Items, cm.ItemResult{Name: "podnetworkchaos", Value: output})
    result.Pods = append(result.Pods, podResult)
  }

  results = append(results, result)
}
return results, nil
```

#### Common Query

The most challenging feature of this RFC is the common query, which allow us to
query or delete any resources by special path.

##### identify resources

We can identify target resources by path. For example, the path
`/networkchaos:<name>/podio:<pod-name>` will identify the podnetworkchaos with
`id=<pod-name>` owned by networkchaos with `id=<name>`. You can also use `@all`
subfix like `/networkchaos@all/podio@all` to identify all resources.

##### Subcommand `query`

We provide `query` subcommand, you can query any resources by their path.

##### Subcommand `delete`

We will provide `delete` subcommand, witch will delete identified resources.

##### Translation

If we choose GraphQL as the API solution, we must translate paths to queries.
The `@all` subfix will be tanslated to GraphQL fragments without any parameters.
For example, resource paths `/networkchaos@all/iptables@all` will be translated
to query `{ networkchaos { iptables } }`.

And the cascade subpaths will be tanslate to fragments with paramenters. For
example, resource paths `/networkchaos:<name>/iptables:<podName>` will be
translated to following query.

```GraphQL
# { "name": "<name>", "podName": "<podName>" }
query GetIptables($name: String!, $podName: String!) {
  networkchaos(name: $name) {
    iptables(name: $podName)
  }
}
```

##### Auto-Completion

We can improve auto-completion with the schema and data. For example, when the
user types `chaosctl get /network`, we can complete the command to `chaosctl get
/networkchaos/` or `chaosctl get /networkchaoses` by schema. Then, if the user
choose `chaosctl get /networkchaos/`, we can send query
`{ networkchaos { name } }` and complete the command to
`chaosctl get /networkchaos/<name>` by the result.

## Alternatives

Implement all APIs one by one as our requirements.
