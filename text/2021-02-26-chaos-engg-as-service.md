# Chaos Engineering as a Service (CAAS)

## Summary

Build a unified dashboard to manage and monitor experiments for multiple
platforms and multiple clusters.

Related Issue: https://github.com/chaos-mesh/chaos-mesh/issues/1462

## Motivation

Following problem exists with the current chaos-mesh design:

1. Cost of maintainance and change is high, since chaos-daemon and chaosd are
   two seperate programs that serve similar purpose.
   **Goal is to abstract out common parts of two into a common library.**
2. Poor observability of experiment results from within the dashboard
   **Goal is to collect the metrics by Prometheus and show in dashboard.**

## Detailed design

### Architecture redesign

Current architecture: https://chaos-mesh.org/docs/overview/architecture

![New architecture](https://user-images.githubusercontent.com/5793595/106101841-7235d600-6179-11eb-8d57-eadd51ac1e6a.png)

So, unlike the current architecture, Chaos Dashboard is truly multi-cluster
since it can exist outside the cluster and manage multiple chaos controllers
and even chaosd for physic nodes.

There's an addition for Prometheus to collect node metrics to incorporate
better visibility of experiment results in the dashboard.

### Unify chaosd and chaos-daemon

chaosd and chaos-daemon both serve similar purposes but targeted at different
types of nodes - physic nodes and kubernetes' worker nodes respectively.
And that's why their communication mechanism is totally different:

- chaosd lacks server support and so it's only a CLI for now
- chaos-daemon communicates with chaos controller via gRPC

So most of the logic (the ones causing chaos amongst other things) can be
abstracted down to a common library for both chaosd and chaos-daemon.
This ensures easy maintainance and easy-to-change for these components.

Server support needs to be added to chaosd so it listens for authenticated
requests on some port of the host machine.

### Authentication & Authorization

#### Chaosd

Chaosd runs on physic nodes outside kubernetes cluster, so it is vulnerable to attack
from internet. To prevent misuse of chaosd, it needs to allow only authenticated
requests. The easiest and secure setup is to use SSL certificates to both encrypt
the request data and for authentication.

From the perspective of communication, the dashboard will represent the end user
and so act as a client, whereas chaosd instance would represent the server.
The client can be authenticated here by making use of
[SSL Client Authentication](https://aboutssl.org/ssl-tls-client-authentication-how-does-it-works/)
technique.

In this setup, private key of the certificate will be generated and kept with the
dashboard and public key would be stored on chaosd nodes. On any request,
chaosd would first verify the digital signatures presented by the client to
authenticate the request.

#### Chaos Mesh

Chaos Mesh is by default authenticated using kubernetes token provided.
If needed, requests could be further protected using SSL certificates.

#### Dashboard

In dashboard, basic authentication protocol using username/password can be
implemented and the data of users can be stored in DB. To implement RBAC
(Role-based access control), **roles** can be defined to comprise of allowed
permissions for that role. User and Role and related by many-to-many relationship,
i.e. user can have many roles and a role can belong to many users.
Only the user with admin privilege can add/edit users and roles.

To allow access of a role to a particular chaos nodes (whether physic/kubernetes),
admin can permit the role to have access to nodes with particular tag,
which is set in the dashboard.

### Web Dashboard

With this new powerful dashboard, chaos-mesh will be one step closer to
making **Chaos Engineering as a Service** possible. End user can manage
multiple node groups (both kubernetes' and physic) from within this dashboard,
adding/removing cluster configuration from the UI.

For physic nodes, a URL pointing to chaosd server needs to be provided,
along with authentication credentials. Whereas for kubernetes' nodes, user
needs to provide kubeconfig of the cluster.

It'll also collect prometheus metrics for better visibility of the experiment
from within the dashboard itself.

## Drawbacks

1. The dashboard will be storing both authentication credentials and
   kubeconfig in the DB, so there's a security risk unless handled properly
   and securely.

## Alternatives

NA

## Unresolved questions

1. How to securely store auth credentials in the dashboard?
   (could refer GitHub Secrets)
