# Chaos Types for Cloud Providers

## Summary

The current ASWChaos, GCPChaos and AzureChaos experiment types are quite basic.

We would like to extend them to support selecting instances using additional
filters such as tags or labels, instance type, availability zone, etc, as
supported by the different cloud providers.

In future we would like to add more types for different cloud resources
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
cloud and save these in the `records` as described
[here](https://github.com/chaos-mesh/chaos-mesh/tree/master/controllers/common/records#readme)


## Detailed design

Add new fields to the existing AWS Chaos, GCP Chaos and Azure Chaos
definitions, to allow specifying filters.

### AWS Chaos selecting multiple instances using tag

```
apiVersion: chaos-mesh.org/v1alpha1
kind: AWSChaos
metadata:
  name: ec2-stop-example
  namespace: chaos-mesh
spec:
  action: ec2-stop
  awsRegion: 'us-east-2'
  filters:
  - name: 'tag:environment'
    value: 'staging'
  - name: 'instance-type'
    value: 't2.micro'
  mode: 'all'
  duration: '5m'
```

Supported filters are defined in the AWS SDK. For details see:
https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_DescribeInstances.html

### GCP Chaos selecting multiple instances using labels

```
apiVersion: chaos-mesh.org/v1alpha1
kind: GCPChaos
metadata:
  name: node-stop-example
  namespace: chaos-mesh
spec:
  action: node-stop
  secretName: 'cloud-key-secret'
  project: 'your-project-id'
  zone: 'your-zone'
  filters:
  - 'labels.environment = staging'
  - 'name ne .*web'
  mode: 'all'
  duration: '5m'
```

## Drawbacks


## Alternatives

- Cloud extensions to live in a seperate repository

   It would perhaps be more scalable for the cloud provider types to live in 
   separate source repositories, allowing them to be maintained without
   impacting the main chaos-mesh code base.

   However this doesn't seem feasible at this time.  We would need some kind
   of plugin / exnension model to support this.


## Unresolved questions

- Should new chaos types be introduced for additional cloud resource types?

   It might be necessary to introduce new chaos types e.g. AWSNetworkChaos,
   GCPNetworkChaos and so on.

   It might be necessary to make breaking changes to the structures, so it 
   in this case it would be easier to introduce new types and eventually
   deprecate the old ones.

   We need to explore the solution in more detail before we can decide.
