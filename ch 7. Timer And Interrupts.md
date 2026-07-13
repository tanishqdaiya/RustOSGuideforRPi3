<big> Below chapter is currently under progress </big>

# Chapter 7: Timer And Interrupts

You may already know what interrupts are by now. Typically in general computer science langauge, an interrupt simply means temporarily stopping the execution of some set of instruction and executing some instructions at a completely different location. Then returning back to the original set of instructions whose execution was interrupted. 

Of course this is not a new concept to you at all. You have already seen this kind of behaviouor in earlier chapters. Which is hardware exceptions of course. In chapter 3 we set up a way to catch hardware exceptions, and once our handler is done handling the exception, it returns the `PC` back to where it was; right before it got sent to the exception handler. You are very familiar with this procedure by now. 

Now as you may remember, we learned that there were four kinds of exceptions that can occur. There was the synchronous, IRQ, FIQ, and SError. These exceptions trigger the CPU to prompt it to do some sort of handling for them. We have also seen through the chapter 5 on syscalls, that it is possible to have manually triggered exceptions for the sake of some manual prompt to the kernel. To be exact we are talking about the `svc` instruction. Which upon execution, immediately causes an exception in the EL1 space under the type "Synchronous exception". It is called a synchronous exception because it occured directly do to the execution of instructions in the EL0 space. Execution cannot conitnue without synchronous exceptions being handled. However in this chapter we will learn about a kind of exception of the type IRQ or FIQ. 

## IRQ Exceptions

Now, the full form of IRQ is "Interrupt Request". These exceptions are typically caused whenever a hardware event occurs. So an hardware event that you *may* want to watch for and do something on is notified to the CPU by the hardware as an IRQ exception. It may not be something which needs handling for the current code execution to continue. An example of an hardware event is a USB device being plugged in. If your program is not trying to do anything with USB peripherals, then it likely can continue execution without caring about USB devices being connected or disconnected. Notice how these exceptions are categorzed as interrupt *requests*. This implies that somewhere in the chain of the hardware event occurring, and the CPU handling the event as a IRQ exception, there is an *option* presented. An option that lets you choose whether or not this interrupt is worth accepting by the CPU. We will talk about this more in detail later.

## Hardware event to IRQ pipeline

Now, whenever an hardware event occurs on your machine, in some way, the electronics on your hardware recognize that event, identify that event, and send an appropriate exception to the CPU. Obviously it is not the CPU chip on the hardware which monitors every single peripheral and peripheral for possible events. There is a different part of the hardware which does this job. That part is called the **Interrupt Controller**.

The interrupt controller is a component on the hardware whose job is to be the central hub of recognizing hardware events and notifying the CPU by sending an appropriate signal for IRQ exception. Hardware events upon occuring are recognized by the interrupt controller (by signals received from the part of hardware where event occured). And then the interrupt controller records it, and triggers an Interrupt Request to the CPU. The CPU when upon receiving the interrupt request from the Interrupt Controller, it triggers an appropriate exception to the exception handler. There are two points in this pipeline which can be configured. First is the interrupt controller itself. It can be configured to choose which hardware events should cause an IRQ exception to the CPU, and which ones shouldn't. Next, the CPU itself can also be configured on if certain category of exceptions should straight up be ignored by it. 

So, the pipeline looks something like this:

```
┌─────────────────────────────┐
│      Hardware Event         │
└──────────────┬──────────────┘
               │ signal sent
               ▼
┌─────────────────────────────┐
│   Interrupt Controller      │
└──────────────┬──────────────┘
               │
  Is the interrupt controller
  configured to produce an IRQ   
    upon this hardware event?
         ┌─────┴───────┐
         │             │
       No│             │Yes
         ▼             ▼ (CPU receives IRQ signal)
 Nothing happens    ┌─────┐
                    │ CPU │
                    └─────┘
                       │
                       ▼
             is the CPU configured 
            to accept IRQ interrupts 
                  right now?
             ┌────────┴──────────┐
             │                   │
          Yes│                   │No
             ▼                   ▼
      CPU ignores IRQ   An IRQ Exception occurs 
                              in our CPU
                                  │
                                  ▼
                      ┌───────────────────────┐
                      │ ELI Exception Handler │
                      └───────────────────────┘
```

Our ultimate goal in this chapter is to set up something called a "timer interrrupt". It will be an IRQ exception that you can "schedule" ahead of time to occur after a certain duration automatically. This is the core mechanism on which our scheduler will be built. However we will talk details about that later when we implement the scheduler. First let's learn how to setup the first step of our IRQ pipeline.

## A Scheduled Hardware Event

First of all we are going to attempt to setup the first ever step of our interrupt pipeline that we discussed above. Which is the step of actually causing the hardware event. 

Our main goal is to make it so there is a certain exception caused into the CPU when `T` amount of time passes. Upon which, the timer should be reset and once again after passage of `T` amount of time, the exception must occur again. This will go infinitely.

Sure you can have a part in your kernel where you have a rust counter variable of type integer, and you could just increment it and check if it has reached some threshhold value. And when the threshhold value is reached, you could execute `svc` instruction. After which the kernel shall reset the variable to zero and start incrementing it again. This could technically work. But it is difficult to be precise with this solution. Since if you want the `svc` to trigger every lets say 20ms, then you would need to somehow figure out what threshhold value the counter variable would need exactly 20ms to reach. Another issue here is that this relies on the execution of the kernel code for the variable incrementing and `svc` call. So you have to somehow make sure that this piece of code does not get interrupted and always run exactly the same amount of time on the CPU everytime. It also wastes CPU cycles, costing time in executing instructions to just increment a variable.

There is in fact a better solution exactly for this scenario. Your CPU itself actually has features in it to cause an IRQ hardware event after a fixed amount of time. This feature is called the CPU TIMER. There is a component inside your CPU which actually has a register which it is continually incrementing, and when that register's value reaches a threshold value in another register, it causes an hardware event signal to the interrupt controller. 

It is quite literally a small component inside the CPU chip which is literally called the "timer". There are four of this on a single CPU core in fact. Since we have 4 cores on the Raspberry Pi 3, it of course means we have total 16 of these timers. Four in each CPU. However there are other timers available also. Let's list them.

1. ARM Generic Timers: These are the timers we just discussed. We will learn about these in the next heading. There's exactly four of these in each of the available cores. The four are as follows:
  - Physical Timer `CNTP`: This is the standard timer used typically by kernels running in EL1.
  - Virtual Timer `CNTV`: This is typically used by a guest operating system running in EL1 under a EL2 hypervisor.
  - Hypervisor Physical Timer `CNTH`: Used by a hypervisor at EL2.
  - Secure World Physical Timer `CNTPS`: Used by secure world framework at EL3.

2. The BCM System Timer: This is one exists outside the CPU, on the overall broadcom SoC of the system. This timer is accessed by the CPU as a peripheral with MMIO. It has a 64 bit counter register being incremented at a fixed 1MHz. You can set a compare value to this in one of the four 32 bit *compare value registers*. When the lower 32 bits of the counter register are equal to any one of the compare value register values, the timer goes off. That is, an IRQ hardware event is generated that signifies the timer going off.

3. The QA7 Timer: This timer exists inside the Interrupt Controller component. Can be accessed within the Interrupt Controller's MMIO. It is not generally used and only really exists for the sake of backwards compatibility on the Raspberry Pi. We will not discuss about this one.

For our project, we are going to utilize the ARM Generic Timer for scheduling. Specifically the Physical timer. Also called the Physical Non Secure Timer to explicitly differentiate it fromm the Secure World Physical Timer.

## The Non Secure Physical Timer

Let's first start by understanding the easiest and simplest timer among the ARM Generic Timers. The Physical Non Secure timer.

The Physical Non Secure timer works very similarly to how our hypothetical Rust solution would work. Let's go register wise as we study the functioning of this timer.

### `CNTPCT_EL0`

This is the register which serves the role of the counter register being incremented repeatedly. The name stands for "**C**OU**NT**ER **P**HYSICAL **C**OUN**T** **EL0**". It is a read only register of 64 bits. The 64 bit value in it is constantly being incremented by one at a certain frequency. That frequency is not fixed, as it can be configured and changed through registers which we will learn about next. Functionally speaking, this register just has a value which is being incremented at that frequency. Nothing more. The EL0 suffix in the name signifies that this register can be accessed from EL0 or higher levels (in read only of course).

### `CNTP_CVAL_EL0`

The name of this register stands for "**C**OU**NT**ER **P**HYSICAL: **C**OMPARE **VAL**UE **EL0**". This a 64 bit read and write register. It serves a very simple function. When the value in `CNTPCT_EL0` is greater than or equal to the value inside this register, the Physical Timer goes off! Which means an IRQ hardware event signal is generated to the interrupt controller. So for example if you wanted a timer to go off after 2ms, when the `CNTPCT_EL0` value is being increased at some `f` frequency, you could set this to `(CNTPCT_EL0 + 0.002f)`.  Where `0.002f` reprenents the increase in the counter value after 2ms. However, you don't have to actually do this calculation by yourself. There is a register which does this for you. Described next.

### `CNTP_TVAL_EL0`

The name stands for "**C**OU**NT**ER **P**HYSICAL: **T**IMER **VAL**UE **EL0**". This register is signed 32 bit read and write. It has two behaviours that happen when you attempt to preform write or read to this register. First let's discuss what happens if you try to write to this.

When you write some value to this register, it is interpreted as a signed 32 bit integer. And immediately the compare value register `CNTP_CVAL_EL0` is updated and set exactly to: `(CNTPCT_EL0 + CNTP_TVAL_EL0)`.

So this way, instead of reading the counter register, and then adding to it and updating the compare value register, you can simply set the change in count you want to wait for in this timer value register.

A passive behaviour of this register is that it is constantly being decremented at the same frequency as the counter register `CNTPCT_EL0`. This does not affect the hardware at all and is merely so that you could read it and get a value that represents how many counts are left before the counter register equals set compare value. Negative values representing that the timer has already gone off.

Of course the constant decrementing being a passive behaviour means it never stops. If the value inside it hits the largest 32 bit negative integer, it overflows and wraps arround to largest positive 32 bit integer. 

### `CNTP_CTL_EL0`

This one is very simple. It standards for "**C**OU**NT**ER **P**HYSICAL: **C**ON**T**RO**L** **EL0**". It is the main control register for the Non Secure Physical counter/timer. It is a 3 bit read and wite register. Each of the three bits represents different fields.

- `Bit[0] "ENABLE"`: This bit controls if the timer is enabled at all. Set this to `0` for the timer to stop working. Set this to `1` to turn the timer on. Very simple. When the timer is off, the hardware event is no longer fired. The increment of counter register and decrement of timer value register still happen. Read and Writes will still work as usual. However the actual hardware event will never occur for the IRQ. `Bit[2]` also behaves differently which you will see in a bit.

- `Bit[1] "MASK"`: This bit controls if the hardware event singal should be supressed or not. When set to `1`, the IRQ hardware event for the timer going off will be blocked from reaching the interrupt controller. Otherwise at `0` the interrupt controller will be delivered the hardware event signal when the timer goes off. Note the timer only "goes off" when the timer is enabled by `bit[0]`. 

- `Bit[2] "ISTATUS"`: Stands for Interrupt Status. It is a read only bit. It merely records if currently the timer's hardware event signal should be going off to the interrupt controller or not. A value of `1` means that currently the condition `CNTPCT_EL0 >= CNTP_CVAL_EL0` is true, and that the timer is enabled by `bit[0]`.

Note that the IRQ hardware event signal is sent to the interrupt controller at EVERY single increment of the counter register, for which the timer expiration condition is true. So when `CNTPCT_EL0` is incremented, and `ISTATUS` reads the value of `1`, an hardware event signal is sent. So your code needs to make sure to turn off this timer upon the first signal otherwise it will keep sending IRQ signals to the interrupt controller repeatedly until `CNTPCT_EL0` overflows and becomes smaller than compare value again. 

### `CNTFRQ_EL0`

This register actually doesn't do anything at all. On paper the name stands for "**C**OU**NT**ER **FR**E**Q**UENCY **EL0**. You would think that this is the register that must be responsible for controlling the frequency at which the counter register is increased at. However that's not true at all. The frequency is actually controlled by the interrupt controller. This design choice might seem strange or unintuitive to you, but as an OS developer all we can do is understand the design of the hardware and make use of it. 

Functionally speaking the only use for this register is to serve as a place to store the current frequency of the timer. Not for any hardware function purpose, but simply so other programs running can read this register to immediately know what frequency the timer counter is being incremented at. It is more so like a dictionary in that regard rather than a register which causes some hardware behaviour. 

We will talk about controlling or modifying the frequency later on in this chapter. Just know that on RPI3B+, by default it is 1 MHz.

