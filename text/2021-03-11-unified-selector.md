# Unified Selector

## Summary

The whole controller framework should do more things for the implementation of
chaos. Now, every implementation of chaos selects pods by themselves. However,
in order to track the running status, the controller framework should know the
concrete status of every selected targets, which would be a disaster to record
these status inside the implementation of every chaos :(. An unified selector
framework in the controller framework could solve this problem. This RFC will
talk about the design of this unified selector.

## Motivation

There have been a lot of problems about current selector. For example, the
IOChaos should select volumes, but it selects container now (and at first, it
selects pods). Some chaos use `containerName string` to select a container while
some use `containerNames []string` to select several containers at one time. If
we can abstract selectors into one place, these errors won't happen. The
developer of every chaos will not need to consider "select" or these platform
dependent things.

Another benifit is that it could help the controller to track the status. With
it, the controller would know which target (pod/container) has been injected and
which has not. It's the first step towards the target of a standalone `Schedule`
CRD.

## Detailed Design

Every chaos specification would define a function to get the specification of
selectors:

```go
type StatefulObjectWithSelector interface {
    v1alpha1.StatefulObject

    GetSelectorSpecs() map[string]interface{}
}
```

The method `GetSelectorSpecs` will return a map from `string` to selector
specification (like `struct {v1alpha1.PodSelectorSpec, v1alpha1.PodMode,
v1alpha1.Value}`). The key would be the identifier of the selector, as there
will be multiple selectors in one chaos specification, for example the `.` and
`.Target` selectors in `NetworkChaos`. The controller will iterate this map to
select every `SelectSpec`. It will construct a unified selector first, and then
use this selector to `Select` targets. The unified selector may contain a lot of
implementation inside. The construction would be like:

```go
selector := selector.New(selector.SelectorParams{
    PodSelector: pod.New(r.Client, r.Reader, config.ControllerCfg.ClusterScoped, config.ControllerCfg.TargetNamespace, config.ControllerCfg.AllowedNamespaces, config.ControllerCfg.IgnoredNamespaces),
    ContainerSelector: container.New(r.Client, r.Reader, config.ControllerCfg.ClusterScoped, config.ControllerCfg.TargetNamespace, config.ControllerCfg.AllowedNamespaces, config.ControllerCfg.IgnoredNamespaces),
})
```

With the help of dependency injection, it would be constructed easier. The
`Selector` method of `PodSelector` would be like:

```go
func (impl *PodSelector) Select(ctx context.Context, ps *v1alpha1.PodSelector) ([]interface{}, error)
```

The type of second parameter will decide which selector to use. For example, if
you want to select with `*v1alpha1.PodSelector`, then the unified selector will
use `*PodSelector`, and if you want to select with
`*v1alpha1.ContainerSelector`, then the unified selector will use
`*ContainerSelector` to select.

The definition of the unified selector would be:

```go
func (s *Selector) Select(ctx context.Context, spec interface{}) ([]interface{}, error)
```

The controller would construct the `Selector` first, and then use the `Selector`
to select the chaos targets. However, as the target may have multiple type (it
could be a pod, a container, a volume or an AWS machine), we can only return an
`interface{}`, and the implementation of chaos would assert the type by
themselves.

After selecting every `selectSpecs`, the controller will iterate over selected
items, and call `Apply`/`Recover` for them. All selected items, current item and
the identifier of the `selectorSpec` of current item will be passed into the
`Apply`/`Recover` function.

## Alternatives

This is a RFC about internal design, and there is little choice. If you have
better idea, please comment.
