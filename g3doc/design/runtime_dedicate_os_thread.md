# Proposal: `runtime.DedicateOSThread`

-   Status: Draft as of 2020-03-18
-   Author: jamieliu@google.com

## Objectives

-   Reduce Go runtime scheduling overhead in the gVisor sentry. (See e.g.
    https://gvisor.dev/issue/205, https://gvisor.dev/issue/1942.)

-   Minimize intrusiveness of changes to the Go runtime.

## Background

### Go Runtime Scheduling

In Go, units of executable work are referred to as *goroutines*, which the
runtime calls "G"s. The Go runtime maintains a variably-sized pool of threads
(called "M"s by the runtime) on which Gs are executed, as well as a pool of
"virtual processors" (called "P"s by the runtime) of size equal to
`runtime.GOMAXPROCS()`. Each M requires a P in order to execute Gs, limiting the
number of threads concurrently executing goroutines to `runtime.GOMAXPROCS()`.

### Go Application Behavior

In "typical" Go programs, goroutines usually block for one of three reasons:

-   Blocking on a socket operation

-   Sending to a full channel

-   Receiving from an empty channel

In all three cases, the Go runtime implements blocking by marking the goroutines
as not runnable, and synchronously re-marking the goroutine as runnable when it
becomes unblocked (due to a notification from the runtime's "netpoller" thread
in the former case, and due to channel activity from other goroutines in the
latter 2 cases).

In contrast, goroutines in the gVisor sentry usually block due to blocking
system calls:

-   Task goroutines, which represent application threads, are usually blocked on
    system calls that execute untrusted application code, or (on platforms that
    do not support synchronous switches to application code, such as ptrace)
    wait for such execution to complete.

-   The `fdnotifier` package, which serves a similar purpose to the Go runtime's
    netpoller but sends notifications to types defined in the `waiter` package
    in order to efficiently support the sentry's implementation of application
    syscalls such as `poll`, is usually blocked in the `epoll_pwait` syscall.

-   Inbound network packets are processed by "dispatch loop" goroutines that
    will usually be blocked in the `readv` or `recvmmsg` syscalls.

Goroutines cannot be migrated from threads that are blocked in system calls. In
order to remain work-conserving in the presence of threads blocked in system
calls, the runtime maintains a "sysmon" thread. If the sysmon thread observes
that there are runnable goroutines without enough running Ms to run them, the
sysmon thread will steal Ps from Ms blocked in long (> 20us) syscalls, and
transfer them to idle or new Ms.

### `runtime.LockOSThread()`

The `runtime.LockOSThread()` function temporarily locks the invoking goroutine
to its current thread. It is primarily useful for interacting with OS or non-Go
library facilities that are per-thread. It does not reduce interactions with the
Go runtime scheduler: locked Ms relinquish their P when they become blocked, and
only continue execution after another M "chooses" their locked G to run and
donates their P to the locked M instead, ensuring that `runtime.LockOSThread`
does not circumvent `GOMAXPROCS`.

## Overview

We propose to add a function, tentatively called `DedicateOSThread`, to the Go
`runtime` package, documented as follows:

```go
// DedicateOSThread wires the calling goroutine to its current operating system
// thread, and exempts it from counting against GOMAXPROCS. The calling
// goroutine will always execute in that thread, and no other goroutine will
// execute in it, until the calling goroutine has made as many calls to
// UndedicateOSThread as to DedicateOSThread. If the calling goroutine exits
// without unlocking the thread, the thread will be terminated.
//
// DedicateOSThread should only be used by long-lived goroutines that usually
// block due to blocking system calls, rather than interaction with other
// goroutines.
func DedicateOSThread()
```

Mechanically, `DedicateOSThread` implies `LockOSThread` (i.e. it locks the
invoking G to its M), but additionally locks the invoking M to its P. Ps locked
by `DedicateOSThread` are not counted against `GOMAXPROCS`; that is, the actual
number of Ps in the system (`len(runtime.allp)`) is `GOMAXPROCS` plus the number
of bound Ps (plus some slack to avoid frequent changes to `runtime.allp`).
Corollaries:

-   If `runtime.ready` observes that a readied G is locked to a M locked to a P,
    it immediately wakes the locked M instead of waiting for a future call to
    `runtime.schedule` to select the readied G in `runtime.findrunnable`.

-   `runtime.stoplockedm` and `runtime.reentersyscall` skip the release of
    locked Ps; the latter also skips sysmon wakeup. `runtime.stoplockedm` and
    `runtime.exitsyscall` skip re-acquisition of Ps if one is locked.

-   New goroutines created by goroutines with locked Ps are enqueued on the
    global run queue rather than the invoking P's local run queue.

In essence, `DedicateOSThread` causes a goroutine to become scheduled by the
host kernel scheduler rather than the Go runtime scheduler, while minimally
affecting its interoperation with other goroutines.

While gVisor's use case does not strictly require that the association is
reversible (with `runtime.UndedicateOSThread()`), such a feature is required to
allow reuse of locked Ms, which is likely to be critical for performance.

## Alternatives Considered

-   Improve the Go runtime scheduler to the point where it is equivalent or
    superior in performance to the host kernel scheduler in all circumstances.
    This is not considered realistic.

-   Rewrite the gVisor sentry in a language that does not force userspace
    scheduling. This is a significantly inferior option due to the amount of
    code involved.
