# RPC Through Unix Socket

## Summary

Communicate through abstract unix socket instead of stdin/stdout.

This RFC **don't** touch anything about application layer rpc protocol.

## Motivation

The chaos-daemon communicates with `rs-tproxy`, `toda`... with stdin/stdout.
However, the `stdin` and `stdout` is not good enough for communicating. We
should take care of the log print (make sure that every log falls into
`stderr`), blocking process (through the `bpm/buffer`, which is hard to
understand), and establish a mordern rpc application layer on such a fragile
bottom layer.

This problem can be solved by opening an extra socket or pipe to communicate
with a subprocess. As it's hard to move a named socket to another mount
namespace, the abstract unix socket is the best choice.

## Detailed Design

When it comes to the communicating between chaos-daemon and its subprocess, we
have three components need to describe:

1. Chaos Daemon
2. nsexec
3. The subprocess

### Chaos Daemon

The bpm should enable the Chaos Daemon to pass an extra file (which is the unix
socket fd) to the subprocess.

```go
func (b *ProcessBuilder) WithSocket() *ProcessBuilder {
    b.WithSocket = true
    return b
}
```

While building this process, this field should be set to the `ExtraFiles`:

```go
// Build builds the process
func (b *ProcessBuilder) Build() *ManagedProcess {
    ...
    if b.WithSocket {
        rawListener, err := net.Listen("unix", fmt.Sprintf("@chaos-daemon-%s", *b.identifier))
        listener := rawListener.(*net.UnixListener)
        listenSocket, err := listener.File()
        command.ExtraFiles = append(command.ExtraFiles, listenSocket)
    }
    ...
}
```

Then the chaos daemon can send command through this this connection. **Make sure
that every LISTENING sockets are closed after the command has started, or the
parent will be blocked by dialing dead children**.

If every listening fd is closed, further request will get an error: `dial unix
@xxxx: connect: connection refused`.

### nsexec

The newest version of `nsexec` already supports passing files to its subprocess,
with the help of [command-fds](https://github.com/google/command-fds)

### Subprocess

The subprocess, for example `toda`, will need to establish its own transport
from a raw fd. In go, the raw fd can be converted to a `os.File` directly:

```go
s := os.NewFile(3, "socket")

listener, err := net.FileListener(s)
```

## Alternative

1. Don't touch it! Yes. This RFC doesn't provide any obvious improvement (except
   removing the blocking buffer inside bpm), but I think it's valuable enough
   considering the complexity of the blocking buffer and the further progress on
   rpc through HTTP or gRPC... (But I agree that this proposal doesn't have high
   priority.)

2. Dial abstract unix socket directly (rather than passing fd) Unfortunately,
   the abstract unix socket is binded with the network namespace, and the named
   unix socket (with a path) is binded with the mnt namespace. Considering toda
   (changing mnt namespace) and rs-tproxy (changing network namespace), both of
   them needs to pass the fd through extra files. We can also consider the named
   unix socket (which represented as a file on some path). The negative part of
   this solution is that we have to manage (and clean) the file, while the
   abstract unix socket is automatically cleaned after all fds are closed.

3. Passing anonymous pipe, and use the pipe to communicate. The greatest
   advantage of using unix socket rather than anonymous pipe (with `pipe`
   syscall) is that the unix socket is full-duplex, and nearly all protocol
   running on TCP can also run on it with little modification. With anonymous
   pipe, we have to handle the session layer manually, which is frustrating.

## POC

Here is a simple POC to show that we **CAN** communicate with a process inside
another mnt namespace with abstract socket. In this example, the 23814 is the
target pid.

Client:

```go
package main

import (
    "fmt"
    "io"
    "log"
    "net"
    "net/http"
    "os"
    "os/exec"
    "time"
)

type unixSocketDialer struct {
    addr string
}

func NewUnixSocketDialer(addr string) unixSocketDialer {
    return unixSocketDialer{addr}
}

func (u unixSocketDialer) Dial(network, addr string) (net.Conn, error) {
    return net.Dial("unix", u.addr)
}

func main() {
    rawListener, err := net.Listen("unix", "@test-client.sock")
    if err != nil {
        log.Fatal(err)
    }
    listener := rawListener.(*net.UnixListener)
    listenSocket, err := listener.File()

    pid := 23814
    mntArg := fmt.Sprintf("--mnt=/proc/%d/ns/mnt", pid)
    pidArg := fmt.Sprintf("--pid=/proc/%d/ns/pid", pid)
    netArg := fmt.Sprintf("--net=/proc/%d/ns/net", pid)
    cmd := exec.Command("/usr/local/bin/nsexec", mntArg, pidArg, netArg, "--local", "--keep-fd=3", "./server")
    cmd.ExtraFiles = []*os.File{listenSocket}
    cmd.Stdin = os.Stdin
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr
    cmd.Start()
    rawListener.Close()
    listenSocket.Close()

    dialer := NewUnixSocketDialer("@test-client.sock")
    client := http.Client{Transport: &http.Transport{Dial: dialer.Dial}}

    for {
        res, err := client.Get("http://psedo-host/some")
        if err != nil {
            log.Fatal(err)
        }
        defer res.Body.Close()
        bodyBytes, err := io.ReadAll(res.Body)
        if err != nil {
            log.Fatal(err)
        }
        fmt.Printf("%s: %s\n", time.Now(), string(bodyBytes))
        time.Sleep(time.Second)
    }
}

```

Server:

```go
package main

import (
    "log"
    "net"
    "net/http"
    "os"
)

/*
#include <stdio.h>
*/
import "C"

func main() {
    s := os.NewFile(3, "socket")

    listener, err := net.FileListener(s)
    if err != nil {
        log.Fatal(err)
    }

    httpServer := &http.Server{
        Handler: http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            some, err := os.ReadFile("/some")
            if err != nil {
                w.Write([]byte(err.Error()))
            } else {
                w.Write(some)
            }
        }),
    }

    err = httpServer.Serve(listener)
    if err != nil {
        log.Fatal(err)
    }
}
```
