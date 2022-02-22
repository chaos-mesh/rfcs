# Status Check in Workflow

## Summary

The state check is responsible for collecting the system state before,
during and after the chaos workflow execution and is used to determine
whether the chaos workflow is successful or not. The chaos workflow can
also be stopped automatically when the system becomes unhealthy during
the execution of the chaos workflow.

## Motivation

When currently executing chaos workflows, the user cannot quickly determine
the impact of the chaos workflow on the system.

One conceivable path is:

1. click to start the workflow
1. manually check the key panel on monitor system
1. if the system is observed to become unhealthy
1. go back to Chaos Dashboard and manually stop the workflow

It is clearly a user-unfriendly design. To optimize this process, it is
necessary to introduce status check in workflow.

## Detailed design

### Concept

#### StatusCheck Template

`StatusCheck Template` enables `StatusCheck` to be quickly reused by
multiple workflows. Users can create the `StatusCheck Template` in advance
and then create the `StatusCheck` by referring to the `StatusCheck Template`
when creating the workflow.

#### StatusCheck

`StatusCheck` defines how the user wants to check the health status of
the system. Users can create `StatusCheck` by referring to the
`StatusCheck Template` or by customizing it on Chaos Dashboard.

### `StatusCheck` Properties

#### General Properties

- Execution mode, it could be Continuous or Synchronous
- Overall execution time, it corresponds to the `deadline` of the
  `WorkflowNode`, after which the execution of `StatusCheck` stops
- Timeout of single execution, different types of StatusCheck have different
  implementations. For example, in HTTP StatusCheck, it is the response time
  of the request, beyond which the execution is considered to have failed
- Number of retries
  - BTW, Synchronous StatusCheck also has a retry mechanism to prevent
    the impact of system jitter
- Retry interval time
- Failure to abort the workflow
- The number of the execution history is saved

Here are the details of the execution mode. Continuous StatusCheck and
Synchronous StatusCheck are both supported as children of
`Parallel WorkflowNode` or `Serial WorkflowNode`, or as `EntryNode`
(the statuscheck as `EntryNode` is not really meaningful, but it can be
written that way).

The recommended scenario for Continuous StatusCheck is specified here:
Continuous StatusCheck is a child of the `EntryNode` which is Parallel.
In this scenario, it means that the status check will continue
throughout the workflow execution.

```yaml
templates:
  - name: the-entry
    templateType: Parallel
    deadline: 240s
    children:
      - status-check
      - node1
      - node2
  - name: status-check
    templateType: StatusCheck
    ...
```

The yaml example for the rest of the cases is as follows:

```yaml
templates:
  - name: node0
    templateType: Serial
    deadline: 240s
    children:
      - status-check
      - node1
      - node2
  - name: status-check
    templateType: StatusCheck
    ...
```

#### The Status of `StatusCheck`

- Conditions of current StatusCheck
- StatusCheck execution histories, including the execution time and outcomes

### Status Check with HTTP

HTTP StatusCheck determines the health of the system by response code,
response time, or response body returned from the request URL.

#### HTTP StatusCheck Properties

- Request URL
  - For example: system health API, system key API, Grafana alert API
- Request Method
- Request Header
  - For example: AUTH-KEY
- Response Time
- Response Code
- Response Body

here is a example yaml of HTTP StatusCheck:

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: StatusCheck
metadata:
  name: try-workflow-status-check
  annotations:
    "experiment.chaos-mesh.org/abort": "false"
    "experiment.chaos-mesh.org/description": "try-workflow-status-check"
spec:
  mode: Synchronous
  type: HTTP
  deadline: 20s
  timeoutSeconds: 1
  failureThreshold: 3
  periodSeconds: 3
  historyLimit: 10
  abortIfFailed: true
  http:
    url: http://1.1.1.1:8080
    method: GET
    headers:
    - name: a
      value: b
    result:
      code: 200
status:
  conditions:
    - type: Abort # ProbeSuccess/Accomplished/DeadlineExceed/Abort
      status: "False"
      lastProbeTime: null
      lastTransitionTime: 2018-01-01T00:00:00Z
  records:
    - probeTime: 2018-01-01T00:00:00Z
      outcome: Success # Success/Failure
```

HTTP StatusCheck VS existing `Workflow HTTP Request Task`

- `Workflow HTTP Request Task` is an instant request, not a continuous request
- `Workflow HTTP Request Task` can not stop the workflow

#### How to abort `Workflow`

There are two ways to abort `Workflow`:

- When StatusCheck Failed, it could abort workflow automatically
- Users could abort Workflow manually, by adding the annotation to the workflow

## Drawbacks

- Saving the history of `StatusCheck` in the `status` of `StatusCheck`

## Alternatives

StatusCheck in experiment/schedule？

## Unresolved questions
