# Spinlocks

This document aims to describe the inner workings of a spin-lock in AtOS.
Before discussing that, first we try to motivate the requirement for it, then
discuss the various implementation details and setup some groundwork for
mutexes.

## Historical Background and Motivation

Processors in the old days used to run on a single core. This meant that there
was only one CPU, running a scheduler and context-switching between processes.
Things were simpler, whatever operations you needed to perform, you perform it
and wait till it completes and do the next set of operations.  Then, we figured
how to have multiple processors, effectively introducing parallelism, making
computers faster.  However, if you're executing a program on multiple
processors, you have the benefit of running chunks of the program on different
processors, effectively dividing the work.  Having multiple processors basically
allows you to perform operations faster and do so in parallel and thus a modern
system must utilize that capability.

Our second need is a bit subtler.  Let's say a programmer writes a game and the
game needs do some I/O operation.  In a single core, the I/O might take a lot of
time and thus greatly reduce the performance and user experience. Imagine
waiting for eternity for a background I/O task to complete. That would be
devastating.  Also, while the I/O operation is being performed, the CPU just
sits idly, blocked/waiting for the I/O task to finish, effectively wasting that
computing power being idle.

## Locks and Threads

Now, we introduce the concept of locks.  Let us say you have two threads working
on a program, trying to speed it up.  Your main function starts running, creates
the two threads, and the threads can execute in any way since you don't control
how the scheduler decides to schedule them.  So it may happen that a thread,
say, T1 runs first followed by the other thread, T2.  But it is equally possible
that the CPU decides to run T2 and then run T1.

This is not a big problem when both the threads are doing different tasks but
big problems occur when you're accessing a shared resource.  For example, let's
say in an operating system process demands a memory page from the kernel (if you
don't know what pages are, don't worry, just know that in AtOS we divide the
memory into fixed-size blocks called pages and this technique is called
paging). To get a page, you need to traverse through something called a
freelist, which is a linked list keeping track of the pages that are free for
use i.e., other programs are not using that chunk of memory. Now, multiple
processes can run on different cores of the CPU at the same time and this means,
that there might be ONE small possibility that two different programs access the
freelist and choose the first available chunk of memory and now, you got
yourself a problem.

Depending on the code, you are going to mess up your freelist data structure.
Let's say another process just freed some memory and it tried to modify the
freelist at the same time as the other two processes and now you really can't
make any guarantees on the integrity of the data.  There are many cases of
things like this happening where the expected deterministic output becomes
nondeterministic and so we need locks.

There is a lot to read on concurrency, and this short text can't cover all the
intricacies of the topic and do them justice so we entrust the reader's ability
to find the necessary resources to aid the understanding of the material before
proceeding.

## Implementation

Hoping that we have successfully demonstrated the correct motivation to
implement a lock, let us look at the modern approach to implementing locks.
With sufficient intuition, you will also be able to deduce why the decisions
were taken the way they were.  In summary, just know that initially programmers
tried to implement software locks and it didn't work out so we sought some aid
of the hardware and implemented the locks the way we did.  Pure software mutual
exclusion algorithms still exist, but practical modern locks rely on atomic
primitives aided by the hardware.

### Atomicity

Before we read go through the structure itself, first let us discuss an
important thing known as atomicity.  When an operation appears to happen all at
once, it is said to be atomic.  We will discuss more about its use ahead.

### The `Spinlock` structure

In AtOS, you can see the spinlock structure in `kernel/spinlock.rs`:

```rs
pub struct Spinlock {
    lock: AtomicBool,
    holding_cpuid: Cell<Option<usize>>,

    // For debugging:
    name: &'static str, // Name of the lock
}
```

As you can see, `lock` is just a boolean, you could've also used an integer.
Note that the boolean used is an AtomicBool.  This is a Rust compiler thing, it
provides atomic operations with guarantees from the underlying hardware/memory
model.  When the value is `false`, the lock is free; and when it is `true` it
indicates the lock being held.

This is where atomicity comes into play.  Since two threads can call to acquire
our spinlock, we need to make sure that whoever calls it either completely gains
access to it or doesn't.  This functionality of atomic operation is guaranteed
by the hardware by various methods.

The `holding_cpuid` records the CPU (logical processor/core) that currently owns
the lock. It is not used to synchronize the access to the lock itself (that is
handled by the bool), it just exists mainly to facilitate debugging. The `Cell`
wrapper is a Rust thing, that allows debugging information to be updated even
when `Spinlock` is only accessed through a shared reference `&Spinlock`.  In
other words, `Cell` lets us change the stored CPU ID without requiring a mutable
reference to the entire Spinlock.

Finally, `name` is just a human-readable name for the lock, again finding its
use in debugging more than the lock itself.

### Hardware atomic operations

On AArch64, there are two common ways to implement atomic operations. The older
systems used load-exclusive/store-exclusive pair of instructions (`LDXR` and
`STXR`) while the newer ARMv8.1-A processors introduced Large System Extensions
(LSE), which provide single-instruction atomic operations such as `CAS`
(compare-and-swap), `SWP` (atomic swap), and atomic arithmetic instructions.

Regardless of which ISA we use, the goal is the same: if two cores attempt to
acquire the same lock, we just want the first thread that successfully acquires
the lock by setting the value to true, atomically.  Since, we're using the Rust
Compiler, it will implement the atomic instructions for us and convert it to the
required target assembly.

### The `acquire` function

Now, having defined the structure, let us look at how we have implemented the
`acquire` function, which lets a thread executing inside the process acquire the
lock, effectively spinning and consuming CPU cycles while waiting.  Since this
is a Spinlock implementation, the thread that acquires the lock acquires it and
does its work while the other thread that also tries to acquire the lock after
the aforementioned thread has already acquired waits for the acquiring thread to
release the lock.

Try to read the following code:

```rs
pub fn acquire(&self) {
        Interrupts::push_off();

        let current_cpuid = mycpu().cid;

        // Prevent the exact same lock from deadlocking itself.
        // If it is already locked on a single core, we'd spin forever.
        if self.lock.load(Ordering::Relaxed) && self.holding_cpuid.get() == Some(current_cpuid) {
            panic!("acquire({})", self.name);
        }

        while self.lock.compare_exchange(false,
                                         true,
                                         Ordering::Acquire,
                                         Ordering::Relaxed).is_err() {
            core::hint::spin_loop();
        }

        self.holding_cpuid.set(Some(current_cpuid));
}
```

As you can see, when another thread tries to acquire an already-acquired lock,
we just loop, or more accurately, "spin" to wait until the lock gets released.
So once the lock is released, the while condition will become false and the
calling thread will acquire the lock by the virtue of the compare-exchange
instruction right after.  Now, spinning does waste a lot of CPU cycles, but in
some cases you need a spinlock and in many others, you would do well with
something called a "mutex", a topic for another chapter.

Hopefully now a lot of locking makes sense.  If you notice, we did not yet cover
the `push_off` and its counterpart `pop_off` which will be used in releasing the
lock, but we will cover it in the subsequent sections.

### The `release` function

Now, since a thread can acquire a lock to do some work on the shared resource,
it should also have a way to release such a lock.  The `release` function does
exactly that: check if release is only called by the thread that initially
called acquire and then, set our boolean to false, indicating that the lock has
been released.  You can now read the implementation:

```rs
pub fn release(&self) {
        let current_cpuid = mycpu().cid;
        
        if !self.lock.load(Ordering::Relaxed) {
            panic!("release({})", self.name);
        }

        if self.holding_cpuid.get() != Some(current_cpuid) {
            panic!("Spinlock release error: ({}) owned by CPU {:?}, but CPU {} tried to release it!", 
                   self.name, self.holding_cpuid.get(), current_cpuid);
        }

        self.holding_cpuid.set(None);
        self.lock.store(false, Ordering::Release);

        Interrupts::pop_off();
}
```

## Interrupts, push_off and pop_off

Now we first describe the need for enabling and disabling interrupts and then
discuss the missing piece: the push_off and pop_off calls.  Suppose a timer
interrupt fires when you're in the midst of a critical section.  The CPU
switches to the interrupt handler and now imagine the interrupt handler tries to
acquire the same lock.  But since the lock was already acquired by the
now-interrupted thread, the interrupt handler will spin forever, waiting to
acquire that lock to do its thing and since the interrupted code can't continue
because of the interrupt and do its thing and release the lock, we are stuck in
the interrupt handler.  That's a deadlock.  So whenever you acquire a lock, you
must disable the interrupts.

Okay, so why not just disable them the normal way?  Why do we need 2 entirely
new functions for it?  Suppose we have two functions (pseudocode):

```c
void foo()
{
    interrupts_off();
    bar();
    interrupts_on();
}
```

```c
void bar()
{
    interrupts_off();
    thing();
    interrupts_on();
}
```

Looks harmless? Let's execute it line by line.

```text
foo():
  interrupts_off():
    bar():
      interrupts_off()
      ...
      interrupts_on() <---
  interrupts_on();
```

Interrupts are enabled when bar() exits and now while foo() was expecting
interrupts to be turned off, we introduced a point of failure: foo can be
interrupted by a timer interrupt.  And in such a critical piece of software like
an Operating System, if something has even the slightest chance of going nuclear
and ruining progress, it is better to eliminate the problem.  This one is a
fairly easy problem to fix.

Look at this code in `kernel/processes.rs`:

```rs
pub struct Cpu {
    pub cid: usize,
    pub current_pid: Option<u64>, // Tracking the running proc by id

    pub ncli: usize, // Depth of nested interrupt disabling on this CPU
    pub interrupts_enabled: bool, // Were interrupts enabled BEFORE the very first push_off?
}
```

We only cover ncli and interrupts_enabled here.  The other members hopefully
speak for themselves.

Instead of just turning off interrupts, we can keep a counter, call it `ncli` to
tell us how nested we are into disabling interrupts.  So every call to
`push_off` will increase ncli to indicate the number of levels we are nested
into and every call to `pop_off` will decrease the said counter.  Member
`interrupts_enabled` keeps track of whether the interrupts were already enabled
at the very first push_off call so that we can later restore back to this state.
Here is a simple simulated example:

Current state:
ncli = 0;
interrupts enabled

Call push_off:
  interrupts disabled
  ncli = 1;
Call push_off:
  ncli = 2;

Now the unwinding

Call pop_off:
  ncli = 1;

Call pop_off:
  ncli = 0;
  since interrupts were previously enabled, we restore the previous
  interrupt state.

We hope the example above explains the intent.  Now, you are ready to look at
its implementation in `kernel/interrupts.rs`:

```rs
// push_off and pop_off are like enabling and disabling interrupts but it
// doesn't just toggle blindly. Each push_off matches a pop_off. If
// interrupts were originally off, these functions keep them off.
pub fn push_off() {
        let enabled = Self::irq_enabled();
        Self::irq_disable();

        let c = mycpu();
        if c.ncli == 0 {
            c.interrupts_enabled = enabled;
        }
        c.ncli += 1;
}

pub fn pop_off() {
        let c = mycpu();

        if Self::irq_enabled() {
            panic!("pop_off: interrupts active when they should be masked");
        }

        if c.ncli < 1 {
            panic!("pop_off: nesting underflow!");
        }

        c.ncli -= 1;
        if c.ncli == 0 && c.interrupts_enabled {
            Self::irq_enable();
        }
}
```

These set of functions do exactly as described above, while also maintaining
checks to avoid misuse to a certain extent.

# Conclusion

With this, we have covered the fundamentals of concurrency, implemented a
working version of spinlocks and hopefully established the need for concurrency
in the first place.  In the next chapter, we aim to cover a more sophisticated
implementation of mutexes.  It is recommended that you are clear on concurrency
and scheduling before moving to mutexes as much of it involves those topics.

