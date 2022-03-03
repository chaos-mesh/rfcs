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

And there are also other tools/platforms like Grafana, we would talk later in
[Alternatives](#alternatives).

This proposal would using Grafana Panel as the implementation, but it would keep
the design for other platforms.

The basic outcome features are:

- import Grafana Panel, display it in Chaos Dashboard
- link Grafana Panel with Chaos Experiment/Schedule/Workflow
- enhancement ths display for metrics with chaos experiment events
- share/reuse Grafana Panel with any user in Chaos Dashboard

### Basic Concepts

#### Panel

Panel is the basic unit to display, **it always shown as a line chart on UI**.
Panel could be created with outside Grafana Panel, and it should be persisted
into the storage of Chaos Dashboard. Panel could be shared with any user, any
chaos experiment/schedule/workflow.

Panel could be linked with a chaos experiment/schedule/workflow, the relation of
linking should be persisted.

A Panel must contain some required information about its "raw panel" (the origin
Grafana Panel), which requires:

- Identify the Grafana Panel
  - Which Grafana "backend" to use
  - Dashboard ID of Grafana Dashboard which origin Grafana Panel belongs to
  - Panel ID in Grafana Dashboard
- Variables when drawing the chart
  - Which Datasource to use
  - Related Variables for templating "Expressions" (for querying time-series
    data) in origin Grafana Panel

Once a Panel is created, it should contains anything required to draw the chart.
and it could be drawn on frontend immediately.

> Features like `variable` and `templating expr` in Grafana is not considered in
> short term, once the panel is created, it is "static", the charts drawn with
> its certain parameters would not be changed (expected time range).

A Panel should also contain several additional fields(for easy-to-use purposes):

- Name: custom name of the panel
- Description: description of the panel
- Labels: panel could be labeled with many labels, it's powerful to filter the
  panel

#### Annotations

Annotations are markers on line chart, it could be "instant" or "durable". It
stands for chaos experiments, events and health check(in the future).

The data of annotations would not be persisted into the storage of Chaos
Dashboard.

### Available Behaviors for Users

Users could perform following behaviors:

#### Use Grafana as "backend"

User could create a Grafana backend with ths host of grafana instance and API
Keys. As Chaos Dashboard would not modify dashboard in Grafana, so only `Viewer`
role is enough.

#### Import a Grafana Panel

After the Grafana backend is created, user could import any panel from any
Grafana dashboard.

From the `share` feature of panel, user would get an url like
`http://localhost:13000/d/7JdIeTn7z/telegraf-flux?orgId=1&viewPanel=65081`,
Chaos Dashboard could parse the enough information to identify a Panel:

- Dashboard ID
- PanelID

Then Chaos Dashboard would ask user to choose which datasource should use, and
filling the variables which required for expression templating.

At latest, user would be asked to fill the metadata of the imported panel, like
name, description and labels.

#### View Panels with Chaos Experiment

For any existed chaos experiment, there should be another tab page to access the
linked panel for current chaos experiment. The "list view" of panels should also
display preview the data of the linked panel. And user could inspect one of
panel in more detailed view. Annotation should be displayed on both "list view"
and "inspect view".

#### Link/Unlink Panels with Chaos Experiment

User could link and unlink another panel at the "list view".

#### Edit Panel

All the fields of the panel should be editable:

- Which Grafana backend to use
- Dashboard ID
- Panel ID
- Name
- Description
- Labels

#### Duplicate a Panel

User could duplicate a panel from a existed panel. All the fields of the origin
panel would be copied to the new panel.

> The feature of duplicate panel is helpful to create series of panels with same
> Grafana Panel with different variables

#### Remove a Panel

User could remove the panel imported from Grafana. When the panel is removed, it
would also be removed from the linked chaos experiment.

### About Implementation

The implementation of this feature is based on the following design:

- using RDBMS (sqlite/mysql) and ORM (gorm) to store the data
- all the codes is written in chaos-dashboard

Entities and Relations are listed below:

- Entity:
  - Panel
  - Label
- Relation:
  - Mutli-Multi Relation between Panel and Chaos Experiment
  - Multi-Multi Relation between Panel and Label

We could also use another way to implement this feature without RDBMS, would
talk in [Alternatives](#alternative-use-configmap-instead-of-database).

### Provision Grafana Panel then creating chaos experiment

As so far, user could only link Grafana Panel to already existing chaos
experiments. Here is a way to create the link of Grafana Panel to a new chaos
experiment as the chaos experiment is created.

We would introduce a new annotation: `metrics.chaos-mesh.org/link-panel`, the
value of this annotation should be an array of names of Grafana Panels as JSON
string. Once one chaos experiment holds this annotation, links between
corresponding Grafana Panels and this chaos experiment would be created.

The behavior of this annotation is "appending only", it means if user remove the
annotation of the chaos experiment, the links would still eixst.

### About Authorization and Security

Importing panel from Grafana requires a Grafana API Key with role `Viewer`.

Chaos Mesh does not contains any user management system, so all the Grafana
backend and imported panels are simply shared with all users.

## Drawbacks

<!-- Why should we not do this? -->

We already have a grafana plugin as mark annotations on grafana dashboard. There
are also feature overlapping between that grafana plugin and this new feature.
It might take more effort to keep behaviors consistent.

Integrating Grafana into our components seems not really simple, most of
powerful features of Grafana, like variables and templating and drawing the
charts, are implemented in frontend. And Grafana did not well export them as a
library.

## Alternatives

<!-- - Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not
  choosing them?
- What is the impact of not doing this? -->

- Import metrics data from datasource directly instead of proxy by Grafana, then
  draw charts by ourself. The most important problem is that we also provide the
  feature about "exploring the query" and "design panels and dashboards" like
  Grafana, for user to tuning their charts.
- Choose other Grafana alternative integration, like Datadog, Elastic stack, New
  Relic or other. I think we would import panel from more platform other than
  Grafana, especially if we want to connect with cloud provider and other Cloud
  Monitoring Services. But for now, using Grafana just makes more sense.

### Alternative: use ConfigMap instead of database

From inspired of "Status Check Template", the resource of "Grafana Panel" could
also be stored as ConfigMaps.

There are some Pros and Cons.

Pros:

- It would prevent data loss when the Pod chaos-dashboard is deleted/recreated.

Cons:

- It would ues annotations on chaos experiment CR to present the link between
  chaos experiment and Grafana Panels. Like using "multi-valued" filed in a
  database schema.
  - When delete the Grafana Panel, it's expensive to update the related chaos
    experiment. The way to bypassing it is properly resolving the link to an
    not-existed Grafana Panel.
  - It's expensive to list all the chaos experiments which link to certain
    Grafana Panel.
- The label of Grafana Panel might be also another "multi-valued" annotation.

## Unresolved questions

<!-- What parts of the design are still to be determined? -->

- How to integrate Kubernetes RBAC Authorization with Grafana Credentials? As
  the above design said, all the grafana panel and grafana backend would be
  shared with all users. But we could start to think how do we design the user
  management system as more and more resources only belongs to Chaos Dashboard
  appears.
- How to automatic generate $_interval and $__rate_interval with promql `rate`?
  This parameter would automatically generated with grafana, related codes is
  implemented in frontend. I am not sure porting them to golang, or just let
  user to set a static value.
- Feature request about the templating/preset of chaos
  experiment/schedule/workflow, contains the relation/link with panels.
