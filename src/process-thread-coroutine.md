Process, Thread, and Coroutine
---

Before we start discussing the asynchronization of Rust, we'd better firstly
talk about how the operating system organizes and schedules the tasks, which
will help us understand the motivation of the language-level asynchronization
mechanisms.

# Process and thread

People always want to run multiple tasks simultaneously on the OS even though
there's only one CPU core because one task usually can't occupy the whole CPU
core at most times. Following the idea, we have to answer two questions to get
the final design, how to abstract the task and how to schedule the tasks on the
hardware CPU core.

Usually, we don't want tasks to affect each other, which means they can run
separately and manage their states. As states are stored in the memory, tasks
must hold their own memory space to achieve the above goal. For instance, the
execution flow is a kind of in-memory state, recording the current instruction
position and the on-stack states. In one word, **processes** are tasks having
separated memory spaces on Linux.

Though memory space separation is one of the key features of processes, they
sometimes have to share some memory. First, the kernel code is the same across
all processes, kernel part memory space sharing reduces unnecessary memory
redundant. Secondly, processes need to cooperate so that inter-process
communications (IPC) are unavoidable, and most high-performance [IPCs][1] are
some kind of memory sharing/transferring. Considering the above requirements
sharing the whole memory space across tasks is more convenient in some
scenarios, where thread helps.

A process can contain one (single-thread process) or more threads. Threads in a
process share the same memory space, which means most state changes are
observable by all these threads except for the execution stacks. Each thread has
its execution flow and can run on any CPU core concurrently.

Now we know that process and thread are the basic execution units/tasks on most
OSes, let's try to run them on the real hardware, CPU cores.

## Schedule

The first challenge we meet when trying to run processes and threads is the
limited hardware resources, the CPU core number is limited. When I write this
section, one x86 CPU can at most run 128 tasks at the same time, [AMD Ryzen™
Threadripper™ PRO 5995WX Processor][2]. But it's too easy to create thousands of
processes or threads on Linux, we have to decide how to place them on the CPU
core and when to stop a task, where [OS task scheduler][3] helps.

Schedulers can interrupt an executing task regardless of its state, and schedule
a new one. It's called preemptive schedule and is used by most OSes like Linux. The
advantage is that it can share the CPU time slice between tasks fairly no matter
what they're running, but the tasks have no idea about the scheduler. To
interrupt a running task, hardware interruption like time interruption is
necessary.

The other schedulers are called non-preemptive schedulers, which have to
cooperate with the task while scheduling. Here tasks are not interrupted,
instead, they decide when to release the computing resource. The tasks usually 
schedule themselves out when doing I/O operations, which usually take a while
to complete. Fairness is hard to be guaranteed as the task itself may run
forever without stopping, in which case other tasks have no opportunity to be
scheduled on that core.

No matter what kind of scheduler is taken, tasks scheduling always needs to do the
following steps:

* Store current process/thread execution flow information.
* Change page table mapping (memory space) and flush TLB if necessary.
* Restore the new process/thread execution flow from the previously stored
  state.

After adopting a scheduler operating system can run tens of thousands of
processes/threads on the limited hardware resource.

# Coroutine

We have basic knowledge of OS scheduling, and it seems to work fine in most
cases. Next, let's see how it performs in extreme scenarios. Free software
developer, [Jim Blandy][4], did an [interesting test][5] to show how much time
it takes to do a context switch on Linux. In the test, the app creates 500
thread and connect them with pipes like a chain, and then pass a one-byte
message from one side to the other side. The whole test runs 10000 iterations to
get a stable result. The result shows that a thread context switch takes around
1.7us, compared to 0.2us of a Rust async task switch.

It's the first time to mention "Rust async task", formally it's called
[coroutine][6]. The coroutines are lightweight tasks for non-preemptive
multitasking, whose execution can be suspended and resumed. Usually, the task
itself decides when to suspend and wait for a notification to resume. To suspend
and resume tasks' execution flow, the execution states should be saved, just
like what OS does. Saving the CPU register values is easy for the OS, but not
for the applications. Rust saves it to a state machine, and the machine can only
be suspended and resumed from the valid states in that machine. We name the
state machine "Future".

## Future

We all know that the Future is the data structure returned from an async function,
an async block is also a future. When we get it, it does nothing, it's just a
plan and a blueprint, telling us what it's going to do. Let's see the example
below:

```rust
async fn async_fn() -> u32 {
    return 0;
}
```

We can't see any "Future" structure in the function definition, but the compiler
will translate the function signature to another one returning a "Future":

```rust
fn async_fn() -> Future<Output=u32> {
...
}
```

Rust compiler does us a great favor to generate the state machine for us. Here's
the Futures API from [std lib][7]:

```rust
pub trait Future {
    type Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

pub enum Poll<T> {
    Ready(T),
    Pending,
}
```

The `poll` function tries to drive the state machine until a final result
`Output` is returned. The state machine is a black box for the caller of the
`poll` function, since that `Poll::Pending` means it's not in the final state,
and `Poll::Ready(T)` means it's in the final state. Whenever the `Poll::Pending`
is returned it means the coroutine is suspended. Every call to `poll` is trying
to resume the coroutine.


## Driver

Since `Future`s are state machines, there should be a driver that pushes the
machine state forward. Though we can write the driver manually by `poll`ing the
`Future`s one by one until we get the final result, that work should be done
once and reused everywhere, in the result the runtime comes. One of the key
features of Rust async runtime is to drive `Future`s forward.

# Summary

In this chapter, we learned that "Rust async" is a way to schedule tasks, also
named coroutine. And the execution state is stored in a state machine `Future`.
In the next chapters, we'll discuss `Future` automatical generation by the
compiler and its optimizations.

[1]: https://en.wikipedia.org/wiki/Inter-process_communication
[2]: https://www.amd.com/en/products/cpu/amd-ryzen-threadripper-pro-5995wx
[3]: https://en.wikipedia.org/wiki/Scheduling_(computing)
[4]: https://www.red-bean.com/jimb/
[5]: https://github.com/jimblandy/context-switch
[6]: https://en.wikipedia.org/wiki/Coroutine
[7]: https://doc.rust-lang.org/std/future/trait.Future.html
