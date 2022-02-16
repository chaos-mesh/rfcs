# Link Application Metrics with Chaos Experiments

## Summary

Chaos Dashboard could link Chaos Experiment with Grafana Panels, and display the
application metrics relates to the chaos experiment(also the schedule, and
workflow).

## Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected
outcome? -->

It's a feature drawback that we could not measure the steady-state of the
application. It's one of the important steps in Chaos Engineering Principles.

Although it's very hard to make assertions on applications's steady-state
automatically, we could make some improvements to help people/user to make
assertions on the steady-state.

## Detailed design

<!-- This is the bulk of the RFC. Explain the design in enough detail that:

- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.
- How the feature is used. -->

### Overview

This feature is all about Chaos Dashboard, would not affect anything with other
components and CRD.

We would use Grafana as the source of application metrics, and use Grafana Panel
as the minimum unit to display the metrics. Grafana is the widely used in
monitoring stack, and it's a powerful design platform to make application's
metrics looks better.

The basic outcome features are:

- import Grafana Panel, display it in Chaos Dashboard
- link Grafana Panel with Chaos Experiment/Schedule/Workflow
- share/reuse Grafana Panel with any user in Chaos Dashboard

## Drawbacks

<!-- Why should we not do this? -->

We already have a grafana plugin as mark annotations on grafana dashboard. There
are also feature overlapping between that grafana plugin and this new feature.
It might take more effort to keep behaviors consistent.

Integrating Grafana into out components seems not really simple, most of
powerful features of Grafana, like variables and templating and drawing the
charts, are implemented in frontend. And Grafana did not well export them as a
library.

## Alternatives

<!-- - Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not
  choosing them?
- What is the impact of not doing this? -->

- Import metrics data from datasource, then draw charts by ourself. (The problem
  of "design".)
- Choose other Grafana alternative integration, like Datadog, Elastic stack, New Relic or other.

## Unresolved questions

<!-- What parts of the design are still to be determined? -->

- How to integrate Kubernetes RBAC Authorization with Grafana Credentials?
- How to resolve variables (or templating) in the Grafana Dashboard?
- How to automatic generate $_interval and $__rate_interval with promql `rate`
