# Title

## Summary

Add a new type of selector, namely "containerName selector".

## Motivation

At present, some chaos involve containers (such as container-kill). If only the current selector is used, it is possible that the required container does not exist in the target pod.

## Detailed design

We can add a type to the selector so that we can select pods with a specified container name.

## Drawbacks

It may not be necessary enough.

## Alternatives

Using existing options can solve the problem, but it is more troublesome for users. They need to know which containers exist in which pods.

## Unresolved questions

None.
