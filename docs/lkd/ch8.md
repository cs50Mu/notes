### **Chapter 8. Bottom Halves and Deferring Work**

Interrupt handlers, discussed in the [previous chapter](ch7.md), can form only the first half of any interrupt processing solution, with the following limitations:

* Interrupt handlers run asynchronously and interrupt other potentially important code, including other interrupt handlers. Therefore, to avoid stalling the interrupted code for too long, interrupt handlers need to run as quickly as possible.
* Interrupt handlers run with one of the following conditions:
    * The current interrupt level is disabled (if `IRQF_DISABLED` is unset)
    * All interrupts on the current processor are disabled (if `IRQF_DISABLED` is set)

    Disabling interrupts prevents hardware from communicating with the operating systems, so interrupt handlers need to run as quickly as possible.

* Interrupt handlers are often timing-critical because they deal with hardware.
* Interrupt handlers do not run in process context, so they cannot block, which limits what they can do.

Operating systems need a quick, asynchronous, and simple mechanism for immediately responding to hardware and performing any time-critical actions. Interrupt handlers serve this function well. Less critical work can and should be deferred to a later point when interrupts are enabled.

Consequently, managing interrupts is divided into two parts, or **halves**.

1. The first part, interrupt handlers (**top halves**), are executed by the kernel asynchronously in immediate response to a hardware interrupt, as discussed in the previous chapter.
2. This chapter looks at the second part of the interrupt solution, **bottom halves**.

### Bottom Halves

The job of bottom halves is to perform any interrupt-related work not performed by the interrupt handler. You want the interrupt handler to perform as little work as possible and in turn be as fast as possible. By offloading as much work as possible to the bottom half, the interrupt handler can return control of the system to whatever it interrupted as quickly as possible.

However, the interrupt handler must perform some work; if the work is timing-sensitive, it makes sense to perform it in the interrupt handler. For example, the interrupt handler almost assuredly needs to acknowledge to the hardware the receipt of the interrupt. It may need to copy data to or from the hardware.

Almost anything else can be performed in the bottom half. For example, if you copy data from the hardware into memory in the top half, it makes sense to process it in the bottom half.

No hard and fast rules exist about what work to perform where. The decision is left entirely up to the device-driver author. Although no arrangement is *illegal*, an arrangement can certainly be *suboptimal*.

Since interrupt handlers run asynchronously, with at least the current interrupt line disabled, minimizing their duration is important. The following useful tips help decide how to divide the work between the top and bottom half:

* If the work is time sensitive, perform it in the interrupt handler.
* If the work is related to the hardware, perform it in the interrupt handler.
* If the work needs to ensure that another interrupt (particularly the same interrupt) does not interrupt it, perform it in the interrupt handler.
* For everything else, consider performing the work in the bottom half.

When attempting to write your own device driver, looking at other interrupt handlers and their corresponding bottom halves can help. When deciding how to divide your interrupt processing work between the top and bottom half, ask yourself what *must* be in the top half and what *can* be in the bottom half. Generally, the quicker the interrupt handler executes, the better.

#### Why Bottom Halves?

It is crucial to understand:

* Why to defer work
* When to defer it

You want to limit the amount of work you perform in an interrupt handler because:

* Interrupt handlers run with the current interrupt line disabled on all processors.
* Interrupt handlers that register with `IRQF_DISABLED` run with all interrupt lines disabled on the local processor (plus the current interrupt line disabled on all processors).

Thus, minimizing the time spent with interrupts disabled is important for system response and performance.

Besides, interrupt handlers run asynchronously with respect to other code, even other interrupt handlers. Therefore you should work to minimize how long interrupt handlers run. Processing incoming network traffic should not prevent the kernel's receipt of keystrokes.The solution is to defer some of the work until later.

##### **When is "later"?** *

*Later* is often simply *not now*. <u>The point of a bottom half is not to do work at some specific point in the future, but simply to defer work until any point in the future when the system is less busy and interrupts are again enabled.</u> Often, bottom halves run immediately after the interrupt returns. The key is that they run with all interrupts enabled.

Linux is not the only operating systems that separates the processing of hardware interrupts into two parts.

* The top half is quick and simple and runs with some or all interrupts disabled.
* The bottom half (however it is implemented) runs later with all interrupts enabled.

This design keeps system latency low by running with interrupts disabled for as little time as necessary.

#### A World of Bottom Halves

While the top half is implemented entirely via the interrupt handler, multiple mechanisms are available for implementing a bottom half. These mechanisms are different interfaces and subsystems that enable you to implement bottom halves. This chapter discusses multiple methods of implementing bottom halves. [p135]

##### **The Original "Bottom Half"**

In the beginning, Linux provided only the "bottom half" for implementing bottom halves. This name was logical because at the time that was the only means available for deferring work. The infrastructure was also known as *BH* to avoid confusion with the generic term *bottom half*.

The BH interface was simple.

* It provided a statically created list of 32 bottom halves for the entire system.
* The top half could mark whether the bottom half would run by setting a bit in a 32-bit integer.
* Each BH was globally synchronized. No two could run at the same time, even on different processors.

This was simple and easy to use, but was also inflexible and a bottleneck.

##### **Task Queues**

The kernel developers later introduced *task queues* both as a method of deferring work and as a replacement for the BH mechanism.

The kernel defined a family of queues.

* Each queue contained a linked list of functions to call.
* The queued functions were run at certain times, depending on which queue they were in.
* Drivers could register their bottom halves in the appropriate queue.

This worked fairly well, but it was still too inflexible to replace the BH interface entirely. It also was not lightweight enough for performance-critical subsystems, such as networking.

##### **Softirqs and Tasklets**

The *softirqs* and *tasklets* were introduced during the 2.3 development series, to completely replace the BH interface.

* Softirqs are a set of statically defined bottom halves that can run simultaneously on any processor; even two of the same type can run concurrently.
* Tasklets are flexible, dynamically created bottom halves built on top of softirqs.
    * Two different tasklets can run concurrently on different processors, but two of the same type of tasklet cannot run simultaneously.

Tasklets are a good trade-off between performance and ease of use. For most bottom-half processing, the tasklet is sufficient. Softirqs are useful when performance is critical, such as with networking. Using softirqs requires more care, however, because two of the same softirq can run at the same time. In addition, softirqs must be registered statically at compile time. Conversely, code can dynamically register tasklets.

While developing the 2.5 kernel, all BH users were converted to the other bottom-half interfaces. Additionally, the task queue interface was replaced by the work queue interface. Work queues are a simple yet useful method of queuing work to later be performed in process context.

Consequently, the 2.6 kernel has three bottom-half mechanisms in the kernel:

* Softirqs
* tasklets
* Work queues

##### **Kernel Timers** *

*Kernel timers* is another mechanism for deferring work. Unlike the mechanisms discussed in the chapter, timers defer work for a specified amount of time. That is, although the tools discussed in this chapter are useful to defer work to any time but now, you use timers to defer work until at least a specific time has elapsed.

Therefore, timers have different uses than the general mechanisms discussed in this chapter.  A full discussion of timers is given in [Chapter 11 Timers and Time Management](ch11.md).

##### **Dispelling the Confusion**

*Bottom half* is a generic operating system term referring to the deferred portion of interrupt processing. All the kernel's mechanisms for deferring work are "bottom halves". Some people also confusingly call all bottom halves "softirqs".

Bottom half also refers to the original deferred work mechanism in Linux. This mechanism is also known as a BH, so we call it by that name now and leave the former as a generic description. The BH mechanism was deprecated a while back and fully removed in the 2.5 development kernel series.

In the current three methods that exist for deferring work, tasklets are built on softirqs and work queues are their own subsystem. The following table presents a history of bottom halves.

Bottom Half | Status
----------- | ------
BH | Removed in 2.5
Task queues | Removed in 2.5
Softirq | Available since 2.3
Tasklet | Available since 2.3
Work queues | Available since 2.5

### Softirqs

Softirqs are rarely used directly; tasklets, which are built on softirqs are a much more common form of bottom half. The softirq code lives in the file [kernel/softirq.c](https://github.com/shichao-an/linux/blob/v2.6.34/kernel/softirq.c) in the kernel source tree.

#### Implementing Softirqs

Softirqs are statically allocated at compile time. Unlike tasklets, you cannot dynamically register and destroy softirqs. Softirqs are represented by the `softirq_action` structure, which is defined in [`<linux/interrupt.h>`](https://github.com/shichao-an/linux/blob/v2.6.34/include/linux/interrupt.h):

<small>[include/linux/interrupt.h#L366](https://github.com/shichao-an/linux/blob/v2.6.34/include/linux/interrupt.h#L366)</small>

```c
struct softirq_action
{
	void	(*action)(struct softirq_action *);
};
```

A 32-entry array of this structure is declared in [kernel/softirq.c](https://github.com/shichao-an/linux/blob/v2.6.34/kernel/softirq.c):

```c
static struct softirq_action softirq_vec[NR_SOFTIRQS];
```

Each registered softirq consumes one entry in the array. Consequently, there are `NR_SOFTIRQS` registered softirqs. The number of registered softirqs is statically determined at compile time and cannot be changed dynamically. The kernel enforces a limit of 32 registered softirqs. In the current kernel, only nine exist.

##### **The Softirq Handler**

The prototype of a softirq handler, `action`, looks like:

```c
void softirq_handler(struct softirq_action *);
```

When the kernel runs a softirq handler, it executes this `action` function with a pointer to the corresponding `softirq_action` structure as its argument. For example, if `my_softirq` pointed to an entry in the `softirq_vec` array, the kernel would invoke the softirq handler function as:

```c
my_softirq->action(my_softirq);
```

Note that the kernel passes the entire structure to the softirq handler. This trick enables future additions to the structure without requiring a change in every softirq handler.

A softirq never preempts another softirq. The only event that can preempt a softirq is an interrupt handler. Another softirq (even the same one) can run on another processor, however.

##### **Executing Softirqs**

A registered softirq must be marked before it will execute. This is called *raising the softirq*. Usually, an interrupt handler marks its softirq for execution before returning. Then, at a suitable time, the softirq runs. Pending softirqs are checked for and executed in the following places:

* In the return from hardware interrupt code path
* In the `ksoftirqd` kernel thread
* In any code that explicitly checks for and executes pending softirqs, such as the networking subsystem

Regardless of the method of invocation, softirq execution occurs in [`__do_softirq()`](https://github.com/shichao-an/linux/blob/v2.6.34/kernel/softirq.c#L191), which is invoked by [`do_softirq()`](https://github.com/shichao-an/linux/blob/v2.6.34/kernel/softirq.c#L253). The function is quite simple. If there are pending softirqs, `__do_softirq()` loops over each one, invoking its handler.

The following code is a simplified variant of the important part of `__do_softirq()`:

```c
u32 pending;

pending = local_softirq_pending();
if (pending) {
    struct softirq_action *h;

    /* reset the pending bitmask */
    set_softirq_pending(0);

    h = softirq_vec;
    do {
        if (pending & 1)
            h->action(h);
        h++;
        pending >>= 1;
    } while (pending);
}
```

It checks for, and executes, any pending softirqs. It specifically does the following:

1. It sets the `pending` local variable to the value returned by the `local_softirq_pending()` macro.
    * This is a 32-bit mask of pending softirqs: if bit `n` is set, the `n`th softirq is pending.
2. After the pending bitmask of softirqs is saved, it clears the actual bitmask.
3. The pointer `h` is set to the first entry in the `softirq_vec`.
4. If the first bit in pending is set, `h->action(h)` is called.
5. The pointer `h` is incremented by one so that it now points to the second entry in the `softirq_vec` array.
6. The bitmask `pending` is right-shifted by one.
    * This discards the first bit and moves all other bits one place to the right.
7. The pointer `h` now points to the second entry in the array, and the pending bitmask now has the second bit as the first. Repeat the previous steps.
8. Continue repeating until pending is zero, at which point there are no more pending softirqs and the work is done.
    * This check is sufficient to ensure `h` always points to a valid entry in `softirq_vec` because `pending` has at most 32 set bits and thus this loop executes at most 32 times.

#### Using Softirqs

Softirqs are reserved for the most timing-critical and important bottom-half processing on the system.

Currently, only two subsystems directly use softirqs:

* Networking devices
* Block devices

Additionally, kernel timers and tasklets are built on top of softirqs.

If you add a new softirq, you normally want to ask yourself why using a tasklet is insufficient. Tasklets are dynamically created and are simpler to use because of their weaker locking requirements, and they still perform quite well. Nonetheless, for timing-critical applications that can do their own locking in an efficient way, softirqs might be the correct solution.

##### **Assigning an Index**

Softirqs are declared statically at compile time via an [`enum`](https://github.com/shichao-an/linux/blob/v2.6.34/include/linux/interrupt.h#L341) in `<linux/interrupt.h>`. The kernel uses this index, which starts at zero, as a relative priority. Softirqs with the lowest numerical priority execute before those with a higher numerical priority.

Creating a new softirq includes adding a new entry to this `enum`. When adding a new softirq, you might not want to simply add your entry to the end of the list; instead, you need to insert the new entry depending on the priority you want to give it. By convention, `HI_SOFTIRQ` is always the first and `RCU_SOFTIRQ` is always the last entry. A new entry likely belongs in between `BLOCK_SOFTIRQ` and `TASKLET_SOFTIRQ`.

The following table contains a list of the existing softirq types.

Tasklet | Priority | Softirq Description
------- | -------- | -------------------
`HI_SOFTIRQ` | 0 | High-priority tasklets
`TIMER_SOFTIRQ` | 1 | Timers
`NET_TX_SOFTIRQ` | 2 | Send network packets
`NET_RX_SOFTIRQ` | 3 | Receive network packets
`BLOCK_SOFTIRQ` | 4 | Block devices
`TASKLET_SOFTIRQ` | 5 | Normal priority tasklets
`SCHED_SOFTIRQ` | 6 | Scheduler
`HRTIMER_SOFTIRQ` | 7 | High-resolution timers
`RCU_SOFTIRQ` | 8 | RCU locking

##### **Registering Your Handler**

Next, the softirq handler is registered at run-time via `open_softirq()`, which takes two parameters: the softirq's index and its handler function. For example, the networking subsystem registers its softirqs like this, in [net/core/dev.c](https://github.com/shichao-an/linux/blob/v2.6.34/net/core/dev.c):

<small>[net/core/dev.c#L6017](https://github.com/shichao-an/linux/blob/v2.6.34/net/core/dev.c#L6017)</small>

```c
open_softirq(NET_TX_SOFTIRQ, net_tx_action);
open_softirq(NET_RX_SOFTIRQ, net_rx_action);
```

* The softirq handlers run with interrupts enabled and cannot sleep.
* While a handler runs, softirqs on the current processor are disabled. However, another processor can execute other softirqs.
* If the same softirq is raised again while it is executing, another processor can run it simultaneously. This means that any shared data, even global data used only within the softirq handler, needs proper locking (as discussed in the next two chapters).

Here, locking is an important point, and it is the reason tasklets are usually preferred. Simply preventing your softirqs from running concurrently is not ideal. If a softirq obtained a lock to prevent another instance of itself from running simultaneously, there would be no reason to use a softirq. Consequently, most softirq handlers resort to per-processor data (data unique to each processor and thus not requiring locking) and other tricks to avoid explicit locking and provide excellent scalability.

The reason for using softirqs is scalability. If you do not need to scale to infinitely many processors, then use a tasklet. Tasklets are essentially softirqs in which multiple instances of the same handler cannot run concurrently on multiple processors.

##### **Raising Your Softirq**

After a handler is added to the `enum` list and registered via `open_softirq()`, it is ready to run. To mark it pending, so it is run at the next invocation of `do_softirq()`, call `raise_softirq()`. For example, the networking subsystem would call:

```c
raise_softirq(NET_TX_SOFTIRQ);
```

This raises the `NET_TX_SOFTIRQ` softirq. Its handler, [`net_tx_action()`](https://github.com/shichao-an/linux/blob/v2.6.34/net/core/dev.c#L2252), runs the next time the kernel executes softirqs. This function (`raise_softirq()`) disables interrupts prior to actually raising the softirq and then restores them to their previous state. If interrupts are already off, the function `raise_softirq_irqoff()` can be used as a small optimization. For example:

```c
/*
 * interrupts must already be off!
 */
raise_softirq_irqoff(NET_TX_SOFTIRQ);
```

Softirqs are most often raised from within interrupt handlers. In the case of interrupt handlers, the interrupt handler performs the basic hardware-related work, raises the softirq, and then exits. When processing interrupts, the kernel invokes `do_softirq()`. The softirq then runs and picks up where the interrupt handler left off. In this example, the "top half" and "bottom half" naming should make sense.

### Tasklets

Tasklets are a bottom-half mechanism built on top of softirqs. As mentioned, they have nothing to do with tasks. Tasklets are similar in nature and behavior to softirqs, but have a simpler interface and relaxed locking rules.

When writing a device driver, you almost always want to use tasklets. Softirqs are required only for high-frequency and highly threaded uses. Tasklets, on the other hand, work fine for the vast majority of cases and are very easy to use.

#### Implementing Tasklets

Because tasklets are implemented on top of softirqs, they are softirqs. As discussed, tasklets are represented by two softirqs:

* `HI_SOFTIRQ`
* `TASKLET_SOFTIRQ`.

The only difference in these types is that the `HI_SOFTIRQ`-based tasklets run prior to the `TASKLET_SOFTIRQ`-based tasklets.

##### **The Tasklet Structure**

Tasklets are represented by the `tasklet_struct` structure. Each structure represents a unique tasklet. The structure is declared in `<linux/interrupt.h>`:

<small>[include/linux/interrupt.h#L420](https://github.com/shichao-an/linux/blob/v2.6.34/include/linux/interrupt.h#L420)</small>

```c
struct tasklet_struct {
    struct tasklet_struct *next;  /* next tasklet in the list */
    unsigned long state;          /* state of the tasklet */
    atomic_t count;               /* reference counter */
    void (*func)(unsigned long);  /* tasklet handler function */
    unsigned long data;           /* argument to the tasklet function */
};
```

* The `func` member is the tasklet handler (the equivalent of `action` to a softirq) and receives `data` as its sole argument.
* The `state` member can be zero, `TASKLET_STATE_SCHED`, or `TASKLET_STATE_RUN`.
    * `TASKLET_STATE_SCHED` denotes a tasklet that is scheduled to run.
    * `TASKLET_STATE_RUN` denotes a tasklet that is running. As an optimization, `TASKLET_STATE_RUN` is used only on multiprocessor machines because a uniprocessor machine always knows whether the tasklet is running: it is either the currently executing code or not.
* The `count` field is used as a reference count for the tasklet.
    * If it is nonzero, the tasklet is disabled and cannot run.
    * If it is zero, the tasklet is enabled and can run if marked pending.

##### **Scheduling Tasklets**

Scheduled tasklets (the equivalent of raised softirqs) are stored in two per-processor structures:

* `tasklet_vec` (for regular tasklets)
* `tasklet_hi_vec` (for high-priority tasklets).

Both of these structures are linked lists of `tasklet_struct` structures. Each `tasklet_struct` structure in the list represents a different tasklet.

Tasklets are scheduled via the following functions:

* `tasklet_schedule()`
* `tasklet_hi_schedule()`

Either of them receives a pointer to the tasklet's `tasklet_struct` as its lone argument. Each function ensures that the provided tasklet is not yet scheduled and then calls `__tasklet_schedule()` and `__tasklet_hi_schedule()` as appropriate. The two functions are similar. The difference is that one uses `TASKLET_SOFTIRQ` and one uses `HI_SOFTIRQ`.

`tasklet_schedule()` undertakes the following steps:

1. Check whether the tasklet's state is `TASKLET_STATE_SCHED`. If it is, the tasklet is already scheduled to run and the function can immediately return.
2. Call `__tasklet_schedule()`.
3. Save the state of the interrupt system, and then disable local interrupts. This ensures that nothing on this processor will mess with the tasklet code while `tasklet_schedule()` is manipulating the tasklets.
4. Add the tasklet to be scheduled to the head of the `tasklet_vec` or `tasklet_hi_vec` linked list, which is unique to each processor in the system.
5. Raise the `TASKLET_SOFTIRQ` or `HI_SOFTIRQ` softirq, so `do_softirq()` executes this tasklet in the near future.
6. Restore interrupts to their previous state and return.

`do_softirq()` is run at the next earliest convenience, (as discussed in the previous section). Because most tasklets and softirqs are marked pending in interrupt handlers, `do_softirq()` most likely runs when the last interrupt returns. Because `TASKLET_SOFTIRQ` or `HI_SOFTIRQ` is now raised, `do_softirq()` executes the associated handlers. These handlers, [`tasklet_action()`](https://github.com/shichao-an/linux/blob/v2.6.34/kernel/softirq.c#L399) and [`tasklet_hi_action()`](https://github.com/shichao-an/linux/blob/v2.6.34/kernel/softirq.c#L434), are the heart of tasklet processing; they perform the following steps:

1. Disable local interrupt delivery (there is no need to first save their state because the code here is always called as a softirq handler and interrupts are always enabled) and retrieve the `tasklet_vec` or `tasklet_hi_vec` list for this processor.
2. Clear the list for this processor by setting it equal to `NULL`.
3. Enable local interrupt delivery. Again, there is no need to restore them to their previous state because this function knows that they were always originally enabled.
4. Loop over each pending tasklet in the retrieved list.
5. If this is a multiprocessing machine, check whether the tasklet is running on another processor by checking the `TASKLET_STATE_RUN` flag. If it is currently running, do not execute it now and skip to the next pending tasklet. Recall that only one tasklet of a given type may run concurrently.
6. If the tasklet is not currently running, set the `TASKLET_STATE_RUN` flag, so another processor will not run it.
7. Check for a zero `count` value, to ensure that the tasklet is not disabled. If the tasklet is disabled, skip it and go to the next pending tasklet.
8. Run the tasklet handler after ensuring the following:
    * The tasklet is not running elsewhere
    * The tasklet is marked as running so it will not start running elsewhere
    * The tasklet has a zero `count` value.
9. After the tasklet runs, clear the `TASKLET_STATE_RUN` flag in the tasklet's `state` field.
10. Repeat for the next pending tasklet, until there are no more scheduled tasklets waiting to run.

The implementation of tasklets is simple, but rather clever:

1. All tasklets are multiplexed on top of two softirqs, `HI_SOFTIRQ` and `TASKLET_SOFTIRQ`.
2. When a tasklet is scheduled, the kernel raises one of these softirqs.
3. These softirqs, in turn, are handled by special functions that then run any scheduled tasklets.
4. The special functions ensure that only one tasklet of a given type runs at the same time. However, other tasklets can run simultaneously.

All this complexity is then hidden behind a clean and simple interface.

#### Using Tasklets

In most cases, tasklets are the preferred mechanism with which to implement your bottom half for a normal hardware device. Tasklets are dynamically created, easy to use, and quick.

##### **Declaring Your Tasklet**

You can create tasklets statically or dynamically, depending on whether you have (or want) a direct or indirect reference to the tasklet. If you are going to statically create the tasklet (and thus have a direct reference to it), use one of two macros in `<linux/interrupt.h>`:

```c
DECLARE_TASKLET(name, func, data)
DECLARE_TASKLET_DISABLED(name, func, data);
```

Both these macros statically create a `struct tasklet_struct` with the given name. When the tasklet is scheduled, the given function `func` is executed and passed the argument `data`. The difference between the two macros is the initial reference count. The first macro creates the tasklet with a count of zero, and the tasklet is enabled. The second macro sets count to one, and the tasklet is disabled.

For example:

```c
DECLARE_TASKLET(my_tasklet, my_tasklet_handler, dev);
```

This line is equivalent to

```c
struct tasklet_struct my_tasklet = { NULL, 0, ATOMIC_INIT(0),
                                     my_tasklet_handler, dev };
```

This creates a tasklet named `my_tasklet` enabled with `my_tasklet_handler` as its handler. The value of `dev` is passed to the handler when it is executed.

To initialize a tasklet given an indirect reference (a pointer) to a dynamically created `struct tasklet_struct` named `t`, call `tasklet_init()` like this:

```c
tasklet_init(t, tasklet_handler, dev); /* dynamically as opposed to statically */
```

##### **Writing Your Tasklet Handler**

The tasklet handler must match the correct prototype:

```c
void tasklet_handler(unsigned long data)
```

* As with softirqs, tasklets cannot sleep. You cannot use semaphores or other blocking functions in a tasklet.
* Tasklets also run with all interrupts enabled, so you must take precautions (for example, disable interrupts and obtain a lock) if your tasklet shares data with an interrupt handler.
* Unlike softirqs, two of the same tasklets never run concurrently, though two different tasklets can run at the same time on two different processors. If your tasklet shares data with another tasklet or softirq, you need to use proper locking (see [Chapter 9. An Introduction to Kernel Synchronization](ch9.md) and [Chapter 10. Kernel Synchronization Methods](ch10.md)).

##### **Scheduling Your Tasklet**

To schedule a tasklet for execution, `tasklet_schedule()` is called and passed a pointer to the relevant `tasklet_struct`:

```c
tasklet_schedule(&my_tasklet); /* mark my_tasklet as pending */
```

After a tasklet is scheduled, it runs once at some time in the near future. If the same tasklet is scheduled again, before it has had a chance to run, it still runs only once. If it is already running, for example on another processor, the tasklet is rescheduled and runs again. As an optimization, a tasklet always runs on the processor that scheduled it, making better use of the processor's cache.

* You can disable a tasklet via a call to `tasklet_disable()`, which disables the given tasklet. If the tasklet is currently running, the function will not return until it finishes executing.
    * Alternatively, you can use `tasklet_disable_nosync()`, which disables the given tasklet but does not wait for the tasklet to complete prior to returning. This is usually not safe because you cannot assume the tasklet is not still running.
* A call to `tasklet_enable()` enables the tasklet. This function also must be called before a tasklet created with `DECLARE_TASKLET_DISABLED()` is usable.

For example:

```c
tasklet_disable(&my_tasklet); /* tasklet is now disabled */

/* we can now do stuff knowing that the tasklet cannot run .. */

tasklet_enable(&my_tasklet); /* tasklet is now enabled */
```

You can remove a tasklet from the pending queue via `tasklet_kill()`. This function receives a pointer as a lone argument to the tasklet's `tasklet_struct`. Removing a scheduled tasklet from the queue is useful when dealing with a tasklet that often reschedules itself. This function first waits for the tasklet to finish executing and then it removes the tasklet from the queue. Nothing stops some other code from rescheduling the tasklet. This function must not be used from interrupt context because it sleeps.

### Doubts and Solution

#### Verbatim

##### **p141 on softirq**

> This function (`raise_softirq()`) disables interrupts prior to actually raising the softirq and then restores them to their previous state.

<span class="text-danger">Question</span>: Why would `raise_softirq()` disable interrupt?

##### **p145 on scheduling tasklets**

> After a tasklet is scheduled, it runs once at some time in the near future. If the same tasklet is scheduled again, before it has had a chance to run, it still runs only once. If it is already running, for example on another processor, the tasklet is rescheduled and runs again

<span class="text-danger">Question</span>: I'm confused. What does it mean?
