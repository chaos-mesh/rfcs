# Refine Error Handling

## Background

There are a lot of different error handling pattern in Chaos Mesh. Some of the
errors in the code are equipped with a backtrace, while some are not. The mass
in error handling has been blocking us from diagnosing the problem and handling
potential errors. The Chaos Mesh code base imports `github.com/pingcap/errors`,
`github.com/pkg/errors` and the `errors` in standard library.

## Proposal

Most of the error handling pattern follows [uber
go-style](https://github.com/uber-go/guide/blob/master/style.md), and some rules
are modified or removed because of the latest update of `pkg/errors` and
`errors` in standard library.

### Error Types

When returning errors, consider the following to determine the best choice:

1. Does the client need to extract the information of error? If so, you should
   use a custom type, and implement `Error()` method. You should also wrap it
   with `errors.WithStack` to make sure it has a backtrace.

   Example:

   ```go
   type ErrNotFound struct {
      File string
   }

   func open(file string) error {
      return errors.WithStack(ErrNotFound{file: file})
   }
   ```

   Then the user will be able to use `errors.As` to extract the `ErrNotFound`
   error, and get the `File` field.

   A new pub type of an error should be created carefully, if you don't think the
   field will be useful for error handling, you should not create a new type, and a
   simple error variable with wrap is prefered, e.g.

   ```go
   var ErrNotFound = errors.New("not found")

   func open(file string) error {
      return errors.Wrapf(ErrNotFound, "open file %s", file)
   }
   ```

2. Is it an error which needs to be detected? If so, you should use a global
   public variable to store the error, and wrap it with `errors.WithStack` when
   returning the error.

   Example:

   ```go
   var (
      ErrPodNotFound = errors.New("pod not found")

      ErrPodNotRunning = errors.New("pod not running")
   )

   func handle() error {
      return errors.WithStack(ErrPodNotFound)
   }
   ```

3. Is this an error which will not be detected but appears in a lot of
   functions? If so, you should use a global private variable, and wrap it with
   `errors.WithStack`. If you want to share the same error (like `Not Found`) in
   multiple packages, but the callers will never need to detect the `Not Found`,
   please create a new `notFound` error under every package.

   ```go
   var errNotFound = errors.New("not found")
   ```

4. Is this really a simple error that will not appear in other functions? If so,
   you should use an inline `errors.New`. The `errors.New` in `pkg/errors` is
   already equipped with a stack backtrace, so you don't need to add it again.

   Example:

   ```go
   func open(file string) error {
      return errors.New("not found")
   }
   ```

5. Are you propagating an error returned by other functions? If it's returned by
   function inside the Chaos Mesh, we could assume this error is already
   equipped with a stack backtrace, so there is not need to call
   `errors.WithStack`. However, if it's returned by other libraries, it would be
   better to call `errors.WithStack` to equip it with a stack backtrace. For
   more information, see the section on error wrapping.

   ```go
   func startProcess(cmd *exec.Cmd) error {
      err := cmd.Start()
      if err != nil {
         return nil, errors.WithStack(err)
      }

      return nil
   }
   ```

### Error Wrapping

* Return the original error if there is no additional context to add
* Add context using
  [`"pkg/errors".Wrap`](https://pkg.go.dev/github.com/pkg/errors#Wrap) so that
  the error message provides more context
* Use [`"pkg/errors".Errorf`](https://pkg.go.dev/github.com/pkg/errors#Errorf)
  with if the callers do not need to detect or handle that specific error case.

The [`"pkg/errors".Wrap`](https://pkg.go.dev/github.com/pkg/errors#Wrap) will
add a stack trace, so you don't need to wrap it with `WithStack` again.

The context usually includes: what you are doing, the object of the operation
(like the pod name, the chaos name...).

```go
func startProcess(cmd *exec.Cmd) error {
   err := cmd.Start()
   if err != nil {
      return nil, errors.Errorf("start process: %v", err)
   }

   return nil
}
```

A more realistic example in `chaos-daemon` is:

```go
func (s *DaemonServer) ExecStressors(ctx context.Context,
    req *pb.ExecStressRequest) (*pb.ExecStressResponse, error) {
   ...

   control, err := cgroups.Load(daemonCgroups.V1, daemonCgroups.PidPath(int(pid)))
   if err != nil {
      return nil, errors.Wrapf(err, "load cgroup of pid %d", pid)
   }

   ...
}
```

When adding context to returned errors, keep the context succinct by avoiding
phrases like "failed to", which state the obvious and pile up as the error
percolates up through the stack:

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
s, err := store.New()
if err != nil {
   return fmt.Errorf(
      "failed to create new store: %v", err)
}
```

</td><td>

```go
s, err := store.New()
if err != nil {
   return fmt.Errorf(
      "new store: %v", err)
}
```

</td></tr>
</tbody>
</table>

### Error Handling

The key of error handling is to identify whether this error is an expected one.
For example, if the Chaos Mesh is recovering an injection, but the kubernetes
returns `PodNotFound`, we need to "handle" it by just ignoring.

Identifying errors could have two different situation: if the target error is a
type, you could use `errors.As` to extract the error information, and if the
target error is a variable (created by `errors.New`, in most situation), you
could use `errors.Is`.

The `error` type of golang is really a chaos, a type and a varaible can both
mean a kind of error, and you should treat them in different pattern. To make
the thing simpler, you should **never** use `errors.Is` to identify a customized
error type (other than the variable created by `errors.New`). It's prefered to
use `errors.As` and then judge the equality (unless your error is recursively
wrapped, which is not common, right?)

#### Handling Error from Kubernetes

The error from kubernetes client is usually derived from an HTTP status. The
only way to identify them is to use `k8sError.Is*`, e.g.
`k8sError.IsNotFound`.

#### Handling Error from Grpc

The error returned from grpc call is also too simple, and will lose the
hierarchy error stack. The only way to identify an error returned from the grpc
call is to use the `IsXXX` provided by the callee.

### The End of Error

All error will finally be consumed by something. In this section, we will
discuss when an error is not fully handled (which is not possible in most
situations, as you don't know the full set of the error kinds in go), what
should you do.

#### Before the logger setuped

1. If the logger hasn't setuped, and the error is not bearable. You should call
   `log.Fatal(err)`.
2. If the logger hasn't setuped, and the error is acceptable. You should print
   it with `fmt.Printf("%v", err)`.

#### Inside a reconciler

Use `r.Log.Error` to print the error, with some contexts, and use
`r.Recorder.Event` to send kubernetes event if needed. The `recorder.Failed` is
a simple representation of error event.

#### Inside a grpc function implementation

Make sure the error is printed in the log.

1. If the error needn't to be identified by the client, you could simply return
   it (and it will become an `Unknown Error` with the `err.Error()` as message).
2. If the error should be detected by the client, you should pick up a status
   code, and return `"grpc/status".Error(code, message)`. The message should be
   used to take apart this error from others. You should also provide a
   `IsXXXError` function in an standalone error package to assert the error.

A template implementation for `IsXXX` could be:

```go
package error

import (
   "google.golang.org/grpc/status"
   "google.golang.org/grpc/code"
)

func IsNotFound(err error) bool {
   if grpcError, ok := err.(status.Error); ok {
      status := grpcError.GRPCStatus()
      return status.Code() == code.NotFound && status.Message() == ErrNotFoundMsg
   }
   return false
}
```

It would also be suggested to define a variable to represent the error message

```go
const ErrNotFound = "grpc/status".Error(code.NotFound, ErrNotFoundMsg)
```

This error should only be directly returned to grpc client, but not to the
function inside, as this error doesn't have any stack information.

### Print

If you will return the error to the parent function, it's suggested not to print
it out, as it will be printed by its ancestors.

#### Console

If the logger has setupped, you could use `log.Error` to print the error. The
`zapr` implementation of `logr` will add `errorVerbose` and `errorCause` fields
to represent the full information of the error (with `%+v`).

If the logger hasn't setupped, you could use `fmt.Printf("%+v", err)` to print
the error, or with additional context.

#### Kubernetes Events

All kubernetes events should be sent through `recorder.Event`, to make sure all
information could be extracted from attribution. The message of kubernetes
events shouldn't be long, so please not add the full error stack.

## Alternatives

There are tons of error handling styles. Feel free to propose your opinion
through the comments, and let us discuss and decide the most suitable one for
Chaos Mesh.
