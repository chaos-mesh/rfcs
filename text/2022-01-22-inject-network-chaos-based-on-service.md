# Title

## Summary

<!-- One para explanation of the proposal. -->
Network chaos injection based on Kubernetes service is urgently needed.
We did some research about ChaosMesh for a few weeks, also comparing with some competitor tools. The first impression is that ChaosMesh product is good, but not perfect, but we think it can do better.

## Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected outcome? -->
The IaaS of our company is completedly based on domestic multi-clouds, just like Ali Cloud, Tencent Cloud, and may have more in future. The target is to make our SaaS & PaaS deployed on Ali Cloud and Tencent Cloud at the same time, and keep both active and active, if one cloud is outage, and the other can be completedly hot backup so that we can reduce the MTTR as most as possible. 

The architecture is briefly described as followed, the two clouds can be connected by fiber lines, and the key idea is to make SaaS & IaaS completely isolated, only the PaaS can be mutual accessed to exchange/sync data.

```              
┌┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┐
┆               DNS                 ┆
└┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┘
  ∧                               ∧
  |                               |
  ∨                               ∨  
┌┈┈┈┈┐                          ┌┈┈┈┈┐
┆SaaS┆                          ┆SaaS┆
└┈┈┈┈┘                          └┈┈┈┈┘
  ∧                               ∧
  |                               |
  ∨                               ∨
┌┈┈┈┈┐                          ┌┈┈┈┈┐
┆PaaS┆ <---- fiber lines ---->  ┆PaaS┆
└┈┈┈┈┘                          └┈┈┈┈┘
  ∧                               ∧
  |                               |
  ∨                               ∨
┌┈┈┈┈┐                          ┌┈┈┈┈┐
┆IaaS┆                          ┆IaaS┆
└┈┈┈┈┘                          └┈┈┈┈┘
(Ali Cloud. cidr:172.25.0.0/16) (Tencent Cloud. cidr:10.32.0.0/16)
```

To make above great idea happened, we need to do more network chaos experiments routinely to check if the unitization deployment is still working or broken by any unexpected RD user changes. In Tencent Cloud K8S enviroment (172.25.0.0/16), like based on a specific namespace and service, we expect to inject network 100% loss for 10.32.0.0/16 as external targets, well, it works if the two pods ping directly, but it does not work while accessing Ali Cloud appname using service name. In fact, the service name is resolved to an IPVS LB, which is a proxy for the target appname.

```
Actualy we expect this process working:


(inject network 100% loss for external target 10.32.0.0/16)
             |
             ∨
Tencent Cloud Namespace/Appname  -- ╳ -->  {appname}-svc.{namespace}  -->  Ali Cloud appname
(172.25.0.0/16)                  (can't ping 10.32.0.0/16 )                (10.32.0.0/16)


But current ChaosMesh does not support, it is still blind in service mechanism, seems like it is a little fall behind Cloud Native Concept.
```


## Detailed design

<!--This is the bulk of the RFC. Explain the design in enough detail that:

- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.
- How the feature is used. -->
I still have no specific detailed design for this feature, but have a rough proposal like this: moving the network chaos experiment from Pods to Nodes, transparently transmit the Pods's info(like ip, namespace, appname, etc) to the Nodes, and add some tc/iptable rules based on Nodes according to the specified namespace/appname. 

Hopefully ChaosMesh community experts can have an universal design for K8s service network injection.

## Drawbacks

<!-- Why should we not do this? -->
- N/A

## Alternatives
<!--
- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not
  choosing them?
- What is the impact of not doing this?
-->
- N/A

## Unresolved questions
- N/A
<!-- What parts of the design are still to be determined? -->