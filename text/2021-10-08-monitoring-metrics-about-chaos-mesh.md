# Monitoring Metrics about Chaos Mesh

## Summary

More metrics are needed for improving the observability of `chaos-controller-manager`,
`chaos-daemon` and `chaos-dashboard`. Now, we already have several metrics in
`chaos-controller-manager` and `chaos-daemon`, but it's not enough.

## Motivation

At present, we only collected several metrics about webhooks like
`chaos_mesh_injections_total`, chaos experiments information in
`chaos-controller-manager`, and HTTP and gRPC in `chaos-daemon`.
These metrics are not easy to reflect the overall appearance of chaos-mesh system,
so we need more metrics to improve the observability.

According to this proposal https://github.com/chaos-mesh/chaos-mesh/issues/2198 ,
below is several metrics about logic pattern and performance should be implemented
and exposed on `/metrics` HTTP endpoint:

`chaos-controller-manager`:

- time histogram for each `Reconcile()` in `Reconciler`(already provided by `controller-runtime`)
- count of Chaos Experiments, Schedule, and Workflow
- count of the emitted event by `chaos-controller-manager`
- common metrics of gRPC client
- metrics for kubernetes webhook (already provided by `controller-runtime`)

`chaos-daemon`:

- common metrics of grpc-server (already provided)
- count of processes controlled by `bpm` (background process manager)
- count of iptables/ipset/tc rules

`chaos-dashboard`:

- time histogram for each HTTP query
- count for archived object
- time histogram for archive reconciler (already provided by `controller-runtime`)

## Detailed design

### Metrics Plan

The metrics design is as follows. Here are a few things should be noded:

- The implementations of `chaos_controller_manager_chaos_experiments_count` and
  `chaos_daemon_grpc_server_handling_seconds` have been provided,
  (called `chaos_mesh_experiments` and `grpc_server_handling_seconds`). It is just
  hoping to modify the names to standardize naming. More about naming, see this
  guideline: [Metric and label naming](https://prometheus.io/docs/practices/naming/).
- Time histogram metrics for each `Reconcile()` in `Reconciler` have been provided
  by `controller-runtime` called `controller_runtime_reconcile_time_seconds`.
- Metrics for kubernetes webhook also have been provided by `controller-runtime`.
  They are called `controller_runtime_webhook_latency_seconds`, `controller_runtime_webhook_requests_total`,
  and `controller_runtime_webhook_requests_in_flight`.

<!-- markdownlint-disable MD013 -->

| Name                                                  | Description                                                  | Type         | Label                                                 | Buckets                      |
| ----------------------------------------------------- | ------------------------------------------------------------ | ------------ | ----------------------------------------------------- | ---------------------------- |
| chaos_controller_manager_chaos_experiments            | Total number of chaos experiments and their phases           | GaugeVec     | namespace, kind, phase                                | /                            |
| chaos_controller_manager_chaos_schedules              | Total number of chaos schedules                              | GaugeVec     | namespace                                             | /                            |
| chaos_controller_manager_chaos_workflows              | Total number of chaos workflows                              | GuageVec     | namespace                                             | /                            |
| chaos_controller_manager_emitted_event_total          | Total number of the emitted event by chaos-controller-manager | CounterVec   | type, reason, namespace                               | /                            |
| chaos_controller_manager_grpc_client_handling_seconds | common metrics of grpc-client                                | HistogramVec | grpc_code, grpc_method, grpc_service, grpc_type       | DefBuckets                   |
| chaos_daemon_grpc_server_handling_seconds             | common metrics of grpc-server                                | HistogramVec | grpc_code, grpc_method, grpc_service, grpc_type       | ChaosDaemonGrpcServerBuckets |
| chaos_daemon_bpm_controlled_process_total             | Total count of bpm controlled processes                      | Count        | /                                                     | /                            |
| chaos_daemon_bpm_controlled_processes                 | Current number of bpm controlled processes                   | Guage        | /                                                     | /                            |
| chaos_daemon_iptables_packets                         | Total number of iptables packets                             | GaugeVec     | namespace, pod, container, table, chain, policy, rule | /                            |
| chaos_daemon_iptables_packet_bytes                    | Total bytes of iptables packets                              | GaugeVec     | namespace, pod, container, table, chain, policy, rule | /                            |
| chaos_daemon_ipset_members                            | Total number of ipset members                                | GuageVec     | namespace, pod, container                             | /                            |
| chaos_daemon_tcs                                      | Total number of tc rules                                     | GuageVec     | namespace, pod, container                             | /                            |
| chaos_dashboard_http_request_duration_seconds         | Time histogram for each HTTP query                           | HistogramVec | path, method, status                                  | DefBuckets                   |
| chaos_dashboard_archived_experiments                  | Total number of archived chaos experiments                   | GaugeVec     | namespace, type                                       | /                            |
| chaos_dashboard_archived_schedules                    | Total number of archived chaos schedules                     | GaugeVec     | namespace                                             | /                            |
| chaos_dashboard_archived_workflows                    | Total number of archived chaos workflows                     | GaugeVec     | namespace                                             | /                            |

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
func (it *GrpcBuilder) WithGrpcMetricsCollection() *GrpcBuilder {
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
consider that in BPM each Identifier corresponds to a process. So we could
believe that the length of identifiers of BPM is the metric value.

#### iptables / ipset / tc

For metrics such as `chaos_daemon_iptables_packets`, we should enter the
container network namespace to collect it, such as
`/usr/bin/nsenter -n/proc/%d/ns/net - iptables-save`. In order to collect these
metrics, first we need to list the PIDs of all containers, and then run commands
such as `iptables-save -c` using BPM, and parse the output to obtain information.

1. First, `crclients.ContainerRuntimeInfoClient` needs to provide a new
   interface `ListContainerPIDs() uint32[]`, for each runtime:
   - Docker: call `ContainerAPIClient.ConatinerList` to get PIDs directly, see
     <https://pkg.go.dev/github.com/docker/docker@v20.10.8+incompatible/client#ContainerAPIClient>.
   - containerd: call `Client.Containers` to get the PIDs directly, see
     <https://pkg.go.dev/github.com/containerd/containerd#Client.Containers>.
   - CRI-O: call the gRPC interface `ListContainers` to obtaining the ContainerID,
     and then get the PIDs by `GetPidFromContainerID` which is already
     implemented. See
     <https://github.com/kubernetes/cri-api/blob/a3ef1f9ba8e3252064955235a2e277b0586e1709/pkg/apis/runtime/v1/api.proto#L78>
2. Then, we need the pod name and container name for each containerPID. So
  `GetLabelsFromContainerID` should be provided by `crclients.ContainerRuntimeInfoClient`
  similarly as above.
3. Run the command and parse the output:
   - For iptables metrics: use [iptables_exporter](https://github.com/retailnext/iptables_exporter/blob/master/iptables/parser.go)
     to parse the result of `iptables-save -c` to get the number of chains and
     rules, packet number, and byte number.
   - For ipset metrics: parse `ipset list` to obtain the name and members of each
     set in the ipset.
   - For tc metrics: parse `tc qdisc` to get the number of rules.
