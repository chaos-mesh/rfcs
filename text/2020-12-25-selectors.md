# Selectors

<!-- TOC -->

- [Selectors](#selectors)
  - [Summary](#summary)
  - [Motivation](#motivation)
  - [Detailed design](#detailed-design)
    - [SelectSpec](#selectspec)
    - [Selector](#selector)
      - [Implementations of Selector](#implementations-of-selector)
        - [CertainResourceSelector](#certainresourceselector)
        - [ResourceFetcher](#resourcefetcher)
    - [Filters](#filters)
      - [Composite Filter: AND/OR](#composite-filter-andor)
    - [SelectorChain](#selectorchain)
    - [SelectorBuilder/SelectorFactory](#selectorbuilderselectorfactory)
    - [Chooser](#chooser)
  - [Drawbacks](#drawbacks)
  - [Alternatives](#alternatives)
  - [Unresolved questions](#unresolved-questions)

<!-- /TOC -->

## Summary

Here is an RFC about more efficient and flexible selectors.

## Motivation

Here are already several issues about the selectors:

- [#193](https://github.com/chaos-mesh/chaos-mesh/issues/193) extend with plugin
- [#886](https://github.com/chaos-mesh/chaos-mesh/issues/886) refactor codes
- [#1102](https://github.com/chaos-mesh/chaos-mesh/issues/1102) ask for "rich selector"
- [#1222](https://github.com/chaos-mesh/chaos-mesh/issues/1222) watching changes
- [#1227](https://github.com/chaos-mesh/chaos-mesh/issues/1227) select not only pod
- [#1259](https://github.com/chaos-mesh/chaos-mesh/issues/1259) watching changes
- [#1266](https://github.com/chaos-mesh/chaos-mesh/issues/1266) set-based selector
- [#1322](https://github.com/chaos-mesh/chaos-mesh/issues/1322) select not only pod

In conclusion, issues acquire these features:

- More way to select, like set-based selectors.
- Select not only Pod resource, but also Service or any other resources.
- ~~Implement "watching" feature on selectors.~~

> After reconsidering the `watching`, I think it's heavy enough to start a new
> RFC.

Current Selector implementation is just several methods. It's quite simple but
not easy to extend.

Before entering detailed design, let's take a look at the current implementation
of Selector:

Here is an interface called `SelectSpec`, which contains three methods:

```go
type SelectSpec interface {
    GetSelector() v1alpha1.SelectorSpec
    GetMode() v1alpha1.PodMode
    GetValue() string
}
```

All Chaos CRD resource implements this interface; it means all chaos needs the
selector.

And there are several methods that often used:

```go
func SelectAndFilterPods(ctx context.Context, c client.Client, r client.Reader, spec SelectSpec, clusterScoped bool, targetNamespace string, allowedNamespaces, ignoredNamespaces string) ([]v1.Pod, error)
func SelectPods(ctx context.Context, c client.Client, r client.Reader, selector v1alpha1.SelectorSpec, clusterScoped bool, targetNamespace string, allowedNamespaces, ignoredNamespaces string) ([]v1.Pod, error)
func CheckPodMeetSelector(pod v1.Pod, selector v1alpha1.SelectorSpec) (bool, error)
```

Notice that "Mode/Value" also affect the final result of select, depends on
"Mode", it will apply "OnePod", "AllPod", "FixedPod" etc.

On bushiness logic, there are two-part of `SelectSpec`: "SelectorSpec" and
"Mode/Value". Any resources are filtered/selected by "SelectorSpec" then be
selected again by "Mode/Value", then we get the final target of chaos injecting.

I think we should split these two-part into different interfaces.

There are several usage for the selector:

- Select certain pods with pod-name.
- List pods with Kubernetes native label/field selector from target
  namespace(s), then filter with namespace, annotations, and pod-phase.
- Check a pod(resource) is matches the selector.

## Detailed design

### SelectSpec

The struct of CRD will not change.

It will add another method for `SelectSpec`:

```go
type SelectSpec interface {
    GetSelector() v1alpha1.SelectorSpec
    GetMode() v1alpha1.PodMode
    GetValue() string
    ResourceType() interface{}
}
```

It will return an empty struct with the same type of resources that will be
selected. For example, currently, all implementation should return an empty
struct `v1.Pod`, which means all chaos will be injected into **Pod**. **This
method should be implemented by code generation.**

> We already have requirement for injecting chaos into other resources, like
> **Service** or some Fake/Mirrored Resource for multi-cluster.

Since we will select more types of resources excepted Pod, so `v1alpha1.PodMode`
should be changed to another name, like `ChooseMode` or something else.

### Selector

We will define an interface called `Selector` that could "select" expected
resource from Kubernetes.

> It still depends on Kubernetes. :) We will not leave Kubernetes in shorten
> future.

```go
interface Selector {
    Select() ([]interface{}, error)
    CouldResolve() []interface{}
}

interface Filter{
    Filter(origin []interface{}) ([]interface{}, error)
    CouldResolve() []interface{}
}
```

> As golang do not contain features like generic type, method
> `Selector#Select()` just return `[]interface{}` directly. And a method
> `CouldResolve()` will return an array of the empty struct with the same type
> represents the resources which this Selector/Filter will return.

Why CouldResolve return an array of interface{} not a single interface{}?

Based on Kubernetes, it's easy to implement a shared selector that could resolve
selecting on any types of resources. Because label/namespace and other things
are out of CRD.

Let's talk about implementations:

#### Implementations of Selector

##### CertainResourceSelector

It select resources from Kubernetes by certain names, and it contains a
Kubernetes client and trying to fetch resources by name one-by-one.

##### ResourceFetcher

It also contains Kubernetes client, and select resources from Kubernetes by
Kubernetes native list API with label/field selectors.

### Filters

```go
struct NamespaceFilter {
    AllowedNamespaces []string
}

func (it *NamespaceFilter) Filter(origin []interface{}) ([]interface,error) {
    // your codes here
}
```

#### Composite Filter: AND/OR

For complex condition, we should composite filters so called `AndFilter` and
`OrFilter`, for example:

```go
struct AndFilter {
  filters []Filter
}

func (it *AndFilter) Filter(origin []interface{}) ([]interface{}, error) {
}

// constructor
func AND(filters ...Filters) Filter {
  
}
```

### SelectorChain

It's a composite Selector which contains one or more other selectors and filter.

```go
func NewSelectorChain(originFetcher Selector, filter Filter) Selector {

}
```

### SelectorBuilder/SelectorFactory

> As an ex-java-developer, I prefer SelectorFactory.

It will parse the `SelectSpec` then build a `Selector`. I am not sure to let
"SelectorBuilder/SelectorFactory" handle multi-resources thing or make it upper
level.

### Chooser

> Please give me any suggestions for an alternative name for this interface!

```go
interface Chooser {
  Choose() []interface{}
}
```

I think `Chooser` is quite simple, and even don't concern about multi-type of
the resource. It will handle logic so-called "OnePodMode", "AllPodMode";

> Although chooser is very simple now, but I still want it independency; it is
> useful to designing "watching" features with a stateful Chooser.

## Drawbacks

It will not implement `CheckPodMeetSelector`, so we might select all resources
then make an assert of "Contains".

## Alternatives

No alternative design so far; we need discussions.

## Unresolved questions

A shared Filter, any filter could as wrappered with a Selector backend, so we
could just follow the Selector interface.

```go
type SelectorFunc func(origin []interface{}) (Selector, error)

type SharedFilter struct {
  f SelectorFunc
}

func NewSharedFilter(f SelectorFunc) *SharedFilter {
  return &SharedFilter{f: f}
}

func (it *SharedFilter) Filter(origin []interface{}) ([]interface{}, error) {
  selector, err := it.f(origin)
  if err != nil {
    return nil,err
  }
  return selector.Select()
}
```
