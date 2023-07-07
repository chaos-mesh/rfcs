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
Since `filters` field is new, it is not a breaking change, so it is safe to update the existing chaos types.

The initial types AWSChaos, GCPChaos and AzureChaos represent the common Instance or VirtualMachine resource types.  CloudProviders have many different resource types that we may want to target for chaos in future.  We can introduce new chaos experiment types for these, such as AWSNetworkChaos or AWSAutoscalerChaos.

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

Supported filters are described in the Google Cloud Compute Engine reference.  For details see:
https://cloud.google.com/compute/docs/reference/rest/v1/instances/list

If multiple filters are provided, they will be combined with an AND expression.



## Drawbacks


## Alternatives

- Cloud extensions to live in a seperate repository

   It would perhaps be more scalable for the cloud provider types to live in 
   separate source repositories, allowing them to be maintained without
   impacting the main chaos-mesh code base.

   However this doesn't seem feasible at this time.  We would need some kind
   of plugin / exnension model to support this.


## Unresolved questions

- How to filter VMs list using Azure SDK.

  Documentation isn't the best.  Also the autorest library used in chaos-mesh
  is out of support: https://github.com/Azure/go-autorest

  The following does work:  `az vm list -d --query "[?tags.env=='staging']"`
  but I couldn't easily find how to do this in the golang SDK.  Have asked on
  gophers slack.
