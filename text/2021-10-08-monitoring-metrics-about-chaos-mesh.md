# Monitoring Metrics about Chaos Mesh

## Summary

More metrics are needed for improving the observability of `chaos-controller-manager`,
`chaos-daemon` and `chaos-dashboard`. Now, we already have several metrics in
`chaos-controller-manager` and `chaos-daemon`, but it's not enough.

## Motivation

At present, we only collected several metrics about webhooks like
`chaos_mesh_injections_total`, chaos experiments information like
`chaos-controller-manager`, and HTTP and gRPC in `chaos-daemon`.
These metrics are not easy to reflect the overall appearance of chaos-mesh system,
so we need more metrics to improve the observability.

As for the expected outcome, below is several metrics about logic pattern and
performance should be implemented and exposed on `/metrics` HTTP endpoint:

chaos-controller-manager:

- time histogram for each `Reconcile()` in `Reconciler`
- count of Chaos Experiments, Schedule, and Workflow
- count of the emitted event by chaos-controller-manager
- common metrics of grpc-client
- metrics for kubernetes webhook (already provided by `controller-runtime`)

chaos-daemon:

- common metrics of grpc-server (already provided)
- count of processes controlled by `bpm` (background process manager)
- count of iptables/ipset/tc rules

chaos-dashboard:

- time histogram for each HTTP query
- count for archived object
- time histogram for archive reconciler

## Detailed design

### Metrics Plan

The metrics design is as follows. It should be noted that the implementations
of `chaos_controller_manager_chaos_experiments_count` and
`chaos_daemon_grpc_server_handling_seconds` have been provided, and they are
called `chaos_mesh_experiments` and `grpc_server_handling_seconds`. It is just
hoping to modify the names to standardize naming. More about naming, see this
guideline: [Metric and label naming](https://prometheus.io/docs/practices/naming/).

<!-- markdownlint-disable MD013 -->
| Name                                                  | Description                                            | Type         | Label                                  | Buckets                      |
|-------------------------------------------------------|--------------------------------------------------------|--------------|----------------------------------------|------------------------------|
| chaos_controller_manager_reconcile_duration_seconds   | time histogram for each `Reconcile()` in `Reconciler`  | HistogramVec | type                                   | DefBuckets                   |
| chaos_controller_manager_chaos_schedule_count         | currrent count of Schedule                             | GaugeVec     | namespace                              | /                            |
| chaos_controller_manager_chaos_workflow_count         | currrent count of Workflow                             | GuageVec     | namespace                              | /                            |
| chaos_controller_manager_emitted_event_count          | count of the emitted event by chaos-controller-manager | GaugeVec     | type, reason                           | /                            |
| chaos_controller_manager_grpc_client_handling_seconds | common metrics of grpc-client                          | HistogramVec | code, method, service, type            | DefBuckets                   |
| chaos_daemon_grpc_server_handling_seconds             | common metrics of grpc-server                          | /            | /                                      | ChaosDaemonGrpcServerBuckets |
| chaos_daemon_bpm_controlled_process_total             | total of processes controlled by `bpm`                 | Count        | /                                      | /                            |
| chaos_daemon_bpm_controlled_process_count             | current count of processes controlled by `bpm`         | Guage        | /                                      | /                            |
| chaos_daemon_iptables_chain_count                     | current count of iptables chains                       | GaugeVec     | pod_name, container_name               | /                            |
| chaos_daemon_iptables_rule_count                      | current count of iptables rules                        | GaugeVec     | pod_name, container_name, table, chain | /                            |
| chaos_daemon_iptables_packet_count                    | current count of all packet                            | GaugeVec     | pod_name, container_name, table, chain | /                            |
| chaos_daemon_iptables_packet_bytes                    | current values of all packet byte                      | GaugeVec     | pod_name, container_name, table, chain | /                            |
| chaos_daemon_ipset_member_count                       | current count of members in an ipset                   | GuageVec     | pod_name, container_name, name         | /                            |
| chaos_daemon_tc_count                                 | current count of tc rules                              | GuageVec     | pod_name, container_name               | /                            |
| chaos_dashboard_http_request_duration_seconds         | time histogram for each HTTP query                     | HistogramVec | code, path, method                     | DefBuckets                   |
| chaos_dashboard_archived_object_count                 | current count for archived object                      | GaugeVec     | type                                   | /                            |
| chaos_dashboard_reconcile_duration_seconds            | time histogram for archive reconciler                  | HistogramVec | type                                   | DefBuckets                   |

<!-- markdownlint-enable -->

The current design of Buckets is shown in the table below. The distribution of
these data need to be obtained later in order to adjust the values so that the
number of samples in each bucket is similar.

<!-- markdownlint-disable MD013 -->

| Buckets Name                 | Value                                                             | Description                                                                |
|------------------------------|-------------------------------------------------------------------|----------------------------------------------------------------------------|
| DefBuckets                   | `[]float64{.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10}`     | default prometheus buckets                                                 |
| ChaosDaemonGrpcServerBuckets | `[]float64{0.001, 0.01, 0.1, 0.3, 0.6, 1, 3, 6, 10}`              | the buckets settings have been implemented, just set constants for clarity |
<!-- markdownlint-enable -->

### Collecting Plan

Here will introduce the metrics collection ways in complex scenarios. Common
collection methods such as pull mode can be found in `controllers/metrics/metrics.go`.

#### gRPC

The implementation of `chaos_controller_manager_grpc_client_handling_seconds`
and `chaos_daemon_grpc_server_handling_seconds` will be provided by
<https://github.com/grpc-ecosystem/go-grpc-prometheus>. It should be noted that
the metrics name needs to be replaced in the original implementation of `chaos-daemon`.

```go
// pkg/chaosdaemon/server.go
func newGRPCServer(reg prometheus.Registerer, ...) (*grpc.Server, error) {
    withHistogramName := func(name string) {
        return func(opts *prometheus.HistogramOpts) {
            opts.Name = name
        }
    }

    grpcMetrics := grpc_prometheus.NewServerMetrics()
    grpcMetrics.EnableHandlingTimeHistogram(
        // here we set customized ChaosDeamonGrpcServerBuckets, and customized histogram name
        grpc_prometheus.WithHistogramBuckets(ChaosDaemonGrpcServerBuckets),
        withHistogramName("chaos_deamon_grpc_server_handling_seconds"),
    )
    reg.MustRegister(grpcMetrics)

    // ...
}
```

For the implementation of `chaos_controller_manager_grpc_client_handling_seconds`,
add an option function to `GrpcBuilder` in `pkg/grpc/utils.go` and register
`grpc_prometheus.DefaultClientMetrics` for `controllermetrics.Registry`:

```go
// pkg/grpc/utils.go
func (it *GrpcBuilder) WithMetricsCollection() *GrpcBuilder {
    it.options = append(it.options,
        grpc.WithUnaryInterceptor(grpc_prometheus.UnaryClientInterceptor),
        grpc.WithStreamInterceptor(grpc_prometheus.StreamClientInterceptor),
    )
    return it
}

// cmd/chaos-controller-manager/main.go
func Run(params RunParams) error {
    // ...
    // register grpc_prometheus client metrics
    controllermetrics.Registry.MustRegister(grpc_prometheus.DefaultClientMetrics)
    // ...
}
```

#### BPM

To collect the metrics of `chaos_daemon_bpm_controlled_process_count`, we
suppose the child processes of chaos-daemon are the processes controlled by
bpm. So we should get all current processes and check whether their parent
process ID matches the current PID to get the processes under bpm control.

#### iptables / ipset / tc

For metrics such as `chaos_daemon_iptables_chain_count`, we should enter the
container network namespace to collect it, such as
`/usr/bin/nsenter -n/proc/%d/ns/net - ipset list`. In order to collect these
metrics, first we need to list the PIDs of all containers, and then run commands
such as `iptables-save -c` using BPM, and parse the output to obtain information.

1. First, `crclients.ContainerRuntimeInfoClient` needs to provide a new
   interface `ListContainerPIDs() uint32[]`, for each runtime:
   - Docker: call `ContainerAPIClient.ConatinerList` to get PIDs directly, see
     <https://pkg.go.dev/github.com/docker/docker@v20.10.8+incompatible/client#ContainerAPIClient>.
   - containerd: call `Client.Containers` to get the PIDs directly, see
     <https://pkg.go.dev/github.com/containerd/containerd#Client.Containers>.
   - crio: call the gRPC interface `ListContainers` to obtaining the ContainerID,
     and then get the PIDs by `GetPidFromContainerID` which is already
     implemented. See
     <https://github.com/kubernetes/cri-api/blob/a3ef1f9ba8e3252064955235a2e277b0586e1709/pkg/apis/runtime/v1/api.proto#L78>

2. Run the command and parse the output:
   - Use [iptables_exporter](https://github.com/retailnext/iptables_exporter/blob/master/iptables/parser.go)
     to parse the result of `iptables-save -c` to get the number of chains and
     rules, packet number, and byte number.
   - Parse `ipset list` to obtain the name and members of each set in the ipset.
   - Parse `tc qdisc` to get the number of rules.
