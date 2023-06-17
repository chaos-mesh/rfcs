# Chaos Types for Cloud Providers

## Summary

The current ASWChaos, GCPChaos and AzureChaos experiment types are quite basic.
We would like to improve them and add more types for different cloud resources
such as networking.

## Motivation

We would like to use chaos-mesh to orchestrate more complex testing in cloud
environments.

For example, when stopping an instance it should be possible to select by name,
tags or labels, instance type, availability zone, and other properties as
supported by the cloud provider.

Additionally, it would be useful to simulate outages by changing network ACLs
in a similar way to how NetworkChaos works.

| ChaosType | Selector | Notes |
| --------- | -------- | ----- |
| AWSChaos  | Single instance by `InstanceID`| No support for selectors |
| GCPChaos  | Single instance by `name`| No support for selectors |
| AzureChaos | Single instance by `resource_group` and `name`| Docs does
mention a`mode` parameter, but behavious is not documented |

It would be good to make these types similar, so users of chaos-mesh with
multiple cloud providers can create similar experiments easily across all
clouds.  However since we rely on the underlying SDK libraries from each
provider, it seems sensible that we should keep close the the naming
conventions used in the SDK.

Currently, none of these types use `Selectors` currently.  We should use the
`RecordsController` with the `impl.Select` to lookup matching resources in the
cloud and save these in the `records` as described [here](https://github.com/chaos-mesh/chaos-mesh/tree/master/controllers/common/records#readme)


## Detailed design


## Drawbacks


## Alternatives

- Cloud extensions to live in a seperate repository

   It would perhaps be more scalable for the cloud provider types to live in 
   separate source repositories, allowing them to be maintained without
   impacting the main chaos-mesh code base.

   However this doesn't seem feasible at this time.  We would need some kind
   of plugin / exnension model to support this.

- Extend the existing AWSChaos, GCPChaos and AzureChaos types

   It might be simpler to add code to the existing types.  However, there are
   a potentially large number of different cloud resource types, so this would
   not scale well.

   In addition we might require breaking changes to the structures, so it would
   be easier to introduce new types and eventually deprecate the old ones.

## Unresolved questions
