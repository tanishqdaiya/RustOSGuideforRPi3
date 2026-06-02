# below document is currently under progress

# Hardware Exceptions

You may already be familiar with what an exception is in programming. It is when you write some code which does something it is not supposed to do and your computer screams at you. Stuff like dividing by zero, using a variable that was never defined, etc.

However, even on hardware level there are some actions which inherently you are not supposed to do. For example if you tell the CPU to read from some memory address which doesnt even exist. Such hardware exceptions are dealt with differently than the exceptions in programming languages.

In programming languages, when a exception occurs, it is usually sent to the exception handler. The exception handler is provided information about where the exception occured and what kind, and it kindly lets you know in the console/terminal output. 

In hardware, when an exception occurs, an **interupt** occurs. Which basically means whatever PC the CPU was executing at is immediately paused. the value of the PC is stored, and then the CPU *jumps* to a different address in memory, where it expects to find instructions which-- if the CPU executes-- will handle the exception.

In simpler terms, you have to plan beforehand how you want an hardware exception to be dealt with. You have to write code for it, and compile it to machine language that can be directly executed by the CPU. And then you have to put all that code in memory, and you have to tell your CPU at which address in your memory you have placed said code. So now whenever an exception occurs, the CPU will make a backup of current PC register value, and change PC to the address where your *exception handler* code is. 

Now, naturally you might want to handle different kinds of exceptions differently. So you have to do this process of writing handler code and telling your CPU where it is, for each kind of exception. Telling your CPU where the handler is, is as simple as simply writing the handler code starting address to a certain register of the CPU.

# Exception Levels

You never execute user programs at the same degree of priviledge as the kernel itself. This is a basic in OS development. If you have a program written by a third party, it may be malicious, so you don't give it too much priviledge to interact with the hardware directly. However, what if some exception occurs in the user program? And what if handling that exception requires you to interact with the hardware directly? In this case when an exception occurs, we go to a higher level of hardware priviledge. 

In Raspberry Pi 3b+'s ARM Cortex-A53 CPU core, this concept is already implemented. Priviledge levels are called "Exception levels". There are four exception levels. EL0, EL1, EL2, EL3. 

- EL0: Used for user programs, lowest hardware priviledge
- EL1: Used for the kernel, higher hardware priviledge
- EL2: Used for virtualization, multiple OS at the same time
- EL3: Used for suuuper low level security stuff, affects the processor itself. Highest priviledge.

The last two will not be relevant to our project. Mainly we will be working with EL0 and EL1. 

Whenever a hardware exception occurs in EL0, either the exception is handled in EL0 itself, or if needed the level is raised and exception is sent to EL1 to be handled.

# Relevant Registers
## `ELR_EL1`
The name stands for "Exception Link Register for EL1". We have learned that whenever an exception occurs, the CPU will change PC to the address of the appropriate exception handler instructions. However once the exception handling instructions conclude their job and handle the exception, the program execution may need to go back to the address where it was originally executing at right before the exception occured. How does the CPU know where to go back to? or where to Return to after an exception handling?

The CPU stores the original PC to return to in the `ELR_EL1` register. However, this happens only if the exception is being handled in EL1. For instance, if an exception occured in EL1, and hardware decided to raise priviledge level to EL2 in order to handle it, now a different register called `ELR_EL2` will be used to store the return address into.

## `SPSR_EL1`
Stands for "Saved Program Status Register for EL1". The CPU has many other registers other than PC which work as memory to hold important information about the current execution. We will discuss them more later, but it includes something called "flags", "Interrupt masks", "program status", etc. When the CPU jumps to exception handler, it also must save this important information in case the exception handler modifies any of it. That is what this register holds. This register holds the PSTATE (program state) information of the program that was being executed right before the jump to exception handler occured. It also holds information about whether if on return, the CPU must stay in same EL or drop lower to EL1.

There are also `SPSR_EL2`, `SPSR_EL3`... for when the exeption is taken to other ELs. However right now we will only see exceptions being taken to EL1.

## `CurrentEL`
Less of an actual piece of memory, but more of a user convenience. Whenever we wish to check what EL we are currently in, we can perform a read on the `CurrentEL` register. Which will return one of the following values: `0b0000` for EL0, `0b0100` for EL1, `0b1000` for EL2, and `0b1100` for EL3. You may have realized to get the CurrentEL as integer, you can simply do `(CurrentEL >> 2) & 0b11`

## `ELR_EL2`
Same as the EL1 counterpart. Holds the address to return to upon `ERET` instruction if currently in EL2.

## `SPSR_EL2`
Same as the EL1 counterpart. When we ERET while in EL2, this register is checked to see which state must be 'restored' upon returning to the address in `ELR_EL2`

## `SP_EL0`
This is the stack pointer register for EL0 mode. Higher ELs can also access it. User programs typically use this stack pointer. in EL0 mode, `sp` register directly references to this register.

## `SP_EL1`
Same as earlier but for EL1 and higher modes. EL0 cannot access this. Mainly for the stack pointer of the kernel programs executions.

## `VBAR_EL1`
Stands for Virtual Address Base Register for EL1. Earlier we mentioned that we need to tell the CPU through a register, which address to jump to when an exception occurs. This register holds the address.

But wait.. Only one register? Aren't there many kinds of exceptions possible as earlier discussed? Is there a different register for every single exception type handler?

No. Firstly, there can be four kinds of exceptions:w
- Synchronous exception (SYNC): caused directly because of execution of some instruction that warranted an exception occurance.
- Interrupt Request (IRQ): When some hardware event occurs, like a network card receives some data, or UART receives new bytes, the hardware generates an exception to get the CPU's attention to let it know that the hardware event has occured.
- Fast Interrupt Request (FIQ): Similar to IRQ, but for wayyy more higher priority events. They are categorized into this type because they may need special more higher priority attention as soon as possible.
- System Error (SError): For hardware failures that occur at some random point, independant of instruction execution progress. E.g. an instruction asked for some data, it gets the acknowledgement, execution moves on. But later on during the actual data transfer the hardware has some failure.

There is another way to differentiate exceptions based on the mode of execution right before exception occured:
- EL1t (t stands for thread): If before exception, program was executing in EL1 mode, and using the SP_EL0 register as the stack pointer.
- EL1h (h stands for handler): If before exception, program was executing in EL1 mode, and usnig the SP_EL1 regsiter as the stack pointer.
- EL0 64 bit: Before exception we were executing in EL0, in 64 bit mode.
- EL0 32 bit: Before exception we were executing in EL0, in 32 bit mode.

Yes, as you may have noticed you can choose which of the two registers are used as stack pointer in EL1. We will discuss that later. For now just know that it can be done.
Secondly, yes, EL0 could be running in both 32bit and 64bit modes. The entire processor can switch between the two modes. But you can also selectively choose for only EL0 to run in 32bit mode.

Now, for every type of Exception, there can be four sources of where it came from. (EL1t, EL1h, EL0 32 and 64.) So in total, there could be 16 possible combination of cases which need separate handlers. Does this mean you need 16 registers for the address of each register? The answer is no. The CPU will expect you to keep all the 16 handlers next to each other in memory, with exactly 128 bytes of difference between each handler's address. It has to be in the following order: firstly for EL1t: SYNC handler, then 128 bytes later IRQ handler for EL1t, then FIQ, then SError. Then the same order for EL1h, then EL0 64 bit, and then 32 bit. 

So the CPU actually expects the handlers to be stored like this in memory:
let's say the handler for EL1t, SYNC exception is stored at 0x00, then rest will be at addresses:


| Exception Origin | SYNC | IRQ | FIQ | SError |
|--------|---|---|---|---|
| EL1t | 0x000 | 0x080 | 0x100 | 0x180 |
| EL1h | 0x200 | 0x280 | 0x300 | 0x380 |
| EL0 (64 bit) | 0x400 | 0x480 | 0x500 | 0x580 |
| EL0 (32 bit) | 0x600 | 0x680 | 0x700 | 0x780 |

And then that's the neat part, you only need to tell CPU the address where this table begins. That is, address of EL1t SYNC Exception handler. The rest it already knows will be at offsets of 128 bytes.

Now, you may have noticed that this means for each exception handler you only get like 128 bytes, and if one instruction is of 4 bytes, you can only write 32 instructions for each handler, including `ERET`. This is not necessary. You can kind of cheat this limitation by simply using a `BL` instruction or something similar, to simply jump to some other location in memory where the actual handler code exists. This is typically the standard way in most OS. 

Additional note, the address in `VBAR_EL1` must be 2048 bytes aligned. that basically means that the address must be divisible by 2048.

## `HCR_EL2`
Stands for Hypervisor Configuration Register. EL2 is often called hypervisor mode. This register is suffixed with EL2 because it can only be modified in EL2 or higher priviledge mode. This register acts as a rulebook for EL1. What mode EL1 executes in (32 or 64 bit), whether EL1 exceptions should directly be taken to EL2 or not, interrupts setup, memory translation, etc are configured here. When the system boots the state of this register is random (UNKNOWN).

## `ESR_EL1`
Stands for Exception Syndrome Register for EL1. Holds more detailed information about the identity of the exception. Only works if exception was of SError type. Otherwise it is useless.

## `FAR_EL1`
Stands for Fault Address Register for EL1. If and only if the exception was of type SError, and it involved some sort of memory failure, this register will hold that exact virtual memory address that caused the crash. Whether or not the SError was related to memory can be confirmed through information held in `ESR_EL1`. This register is completely useless in other scenarios.


Well, that wraps it up!!! That was a lot of information to memorize, it will take time to truly get to know all of these registers personally.


# Booting to EL1

When you start your OS. It actually starts in EL2 for the Raspberry Pi 3 CPU. Since we want our OS to work in EL1. We will need to switch to EL1 in boot process in `entry.s`. Sadly there is no actual direct way to just switch over to a different EL. But the correct way to do it is much more clever.

You already know if we are in EL1, and an exception occcurs, and the hardware decides that it needs to be taken to EL2, then the CPU raises priviledge to EL2, and then exception is handled there. Wherein at the end `ERET` instructionn is executed which causes the CPU to go back to EL1.

But the trick is you don't actually need to be handling some exception for `ERET` to send you back to EL1! The CPU is just a machine, it does not know what was going on. All it does is if you execute `ERET` in EL2, it will check `ELR_EL2` to know which address to go to, and it will simply go there. And it will check `SPSR_EL2` to know what mode to set the execution to once it goes to that address. You don't need to be handling an exception! As long as you set `ELR_EL2` and `SPSR_EL2` to some valid values, you can use `ERET` whenever you want in EL2!

*So...* we *could* simply set `SPSR_EL2` in such a way that the CPU thinks that upon `ERET` it must change EL to EL1.

First know the general structure of a SPSR register:

```arm
Bits [4:0]  = M field (mode)
Bit  6      = F (FIQ mask)
Bit  7      = I (IRQ mask)
Bit  8      = A (SError mask)
Bit  9      = D (Debug exception)
Bits [31:27] = NZCVQ flags
Other bits = Reserve or not relevant right now
```

- Bits [4:0] are:
  - 0b00000: we're supposed to enter EL0 mode after `ERET`  
  - 0b00100: EL1t after `ERET`
  - 0b00101: EL1h
  - 0b01000: EL2t
  - 0b01001: EL2h (all five being AArch64 mode)
- Bits 6, 7, 8 are for turning off exception handling for FIQ, IRQ and SError type exceptions (set bit to 1 to ignore them). SYNC type exceptions do not have an option to be ignored because the CPU cannot continue execution without them handled. 
- Bit 9 handles Debug exceptions. They are usually SYNC exceptions, but unlike failure they're usually for things like breakpointns, debugging, etc.
- Bits 31 to 27 are just the flags that the CPU sets during instruction execution. They are just saved here upon exception because upon ERET the following instructions may rely on flags set by previous instructions that were executing before exception occured.

Now, in order to switch to EL1 mode. All we need to do is set Bits 4:0 to 64 bit EL1h mode. (Not EL1t because we're switching for our kernel, and kernel typically uses SP_EL1). Set appropriate values for the interrupt masks and endianness, and then simply execute `ERET`!

We can do this during our entry assembly.

Now, in our entry.S instead of directly doing `bl _rust_main` we do:

```arm
// EL stack pointer
ldr     x0, =_stack_top
msr     SP_EL1, x0

mov     x0, #(1 << 31) // bit 31 selects Aarch32/64
msr     HCR_EL2, x0 // HCR_EL2 is like "a rulebook that EL2 writes for EL1"

// set SPSR_EL2 to EL1h, basically once we ERET, we must switch to EL1h mode
mov     x0, #(0b00101)  // EL1h 
orr     x0, x0, #(0b1111 << 6) // mask all exceptions (D, A, I, F)
msr     SPSR_EL2, x0

// finally set the ERET PC
adr     x0, _rust_main
msr     ELR_EL2, x0

eret // insane
```

Notice that we also set bit 31 in `HCR_EL2` register to 1. This enables 64 bit mode for EL1. Other bits are irrelevant to us right now.

Now let's create a utility function that can read `CurrentEL` register so we can confirm our code is working:

```rs
use core::arch::asm;
pub fn get_current_el() -> u64 {
    let el: u64;
    unsafe { 
        asm!(
            "mrs {0}, CurrentEL",
            "lsr {0}, {0}, #2",
            out(reg) el,
        ); 
    }
    el
}
```

Now just include this function in your main.rs and test it out!

```rs
println!("Current EL is: EL{}", get_current_el()).unwrap();
````

You get output:
```
Current EL is: EL1

```
That means our EL is successfully EL1!

We can now move to exception handling.

# Setting up and loading exception table

Earlier we described what format we have to prepare our exception vector table in memory for the `VBAR_EL1` register.

We can write the entire table in simple assembly. Since we want to stay away from assembly and stay in rust as much as possible, we'll make the exception table simply call a rust function with appropriate arguments for each exception type and source.

Firstly, the exception table must be aligned to 2048 bytes. That means lower 11 bits in the address must be zero. We can do this in assembly as follows: 

```asm
.align 11              // 2^11 = 2048 alignment
```

Now give a label (to name the start of our vector table)

```asm
el1_vectors:
```
and then follow it with something like the following:

```asm
b   el11t_sync
.space 124
b   el11t_irq
.space 124
b   el11t_fiq
.space 124
b   el11t_serror
.space 124

b   el11h_sync
.space 124
b   el11h_irq
.space 124
b   el11h_fiq
.space 124
b   el11h_serror
.space 124

b   el1064_sync
.space 124
b   el1064_irq
.space 124
b   el1064_fiq
.space 124
b   el1064_serror
.space 124

b   el1032_sync
.space 124
b   el1032_irq
.space 124
b   el1032_fiq
.space 124
b   el1032_serror
.space 124
```

The `b` instruction, called branch instruction, basically just jumps to a given address in memory. We have set up 16 branch instructions here, one for each combination of exception type and source. Notice how each branch instruction has a different label for the address to branch to. We can either Define 16 functions in rust with the name of these labels. However, when we jump to rust, rust actually may completely destroy whatever values were in the CPU registers, because it needs the CPU registers for its own rust related purposes. But the handler may need to have the original values in the registers preserved, to inspect during exception handling. So we will first save all the values of all the relevant registers into the memory. Then we will call the rust handler function, this time just giving it the address to all the relevant register values in memory. So it can still have original values somewhere it can see. 

# Handling Exceptions in Rust

How do we pass values to rust? In asm, how do we call a rust function if the function actually takes in some arguments? in `entry.s` when we calk the rust main function, it doesn't expect any arguments so we could simply do a branch instruction to just jump to the address of the rust main function in memory like any other address label. However, when we wish to give arguments to a rust function, we can actually do so by passing the values in the x0, x1, ..., x7 registers. This gives you 8 arguments that you can pass to the rust function (since these registers are 8 byte sized, that's the max size of the arguments you can pass). For greater than 8 arguments, you can utilize the stack. All of this is defined according to the "[AArch64 Procedure Call Standard (AAPCS64)](https://github.com/ARM-software/abi-aa/releases)".

Now we are passing more than 8 arguments. We will need to pass all the General Purpose Registers (GPRs) and the registers relevant to exception handling-- `ELR_EL1`, `SPSR_EL1`, `ESR_EL1`, `FAR_EL1`. Last four registers are for the Rust program to read/modify return-to state for the exception, and last two being for exception diagnosis and details. 

You must be familiar with `structs` in C. They are a way to group up different datatypes into memory. When you create a struct having three different members. Then you must know that those three members are placed consecutively in memory next to each other. So as long as you have the starting address of the struct (`*struct` pointer), you can determine the address of any member by adding its offset from the beginning of the structure.

So far a struct with int, int, char, memory will look like:

```
| 4 bytes for int | 4 bytes for int | 1 byte for char |
^ start of        ^                 ^                 ^
memory           (0x004)           (0x008)           (0x009)
(0x000)
```

So what we will do, is instead of passing all the different registers to the Rust function as multiple arguments, we are going to simply write all the register values to memory in a way, so that it represents a valid Rust/C structure. And then we will simply give the starting address of the part of memory to Rust as a pointer to a structure. 

First we will use a single byte to categorize the Exception type and source:

```rust
#[repr(u8)]
#[derive(Copy, Clone)]
pub enum ExceptionType {
    _SYNC,  // 0
    _IRQ,  // 1
    _FIQ, // and so on
    _SE,
}

#[repr(u8)]
#[derive(Copy, Clone)]
pub enum ExceptionSource {
    _EL1t,
    _EL1h,
    _EL064,
    _EL032,
}
```

Let's say the struct will look like:

```rust
#[repr(C)]
pub struct ExceptionContext {
    pub etype: ExceptionType, // u8
    pub esource: ExceptionSource, // u8
    pub _padding: [u8; 6], // because this struct follows c style repr
    pub x: [u64; 31],   // x0–x30
    pub elr: u64,
    pub spsr: u64,
    pub esr: u64,
    pub far: u64,
}
```

Note the `repr(C)` attribute. Rust might try to optimize the way it stores the struct objects in memory. We don't want that. We want the certainty and unambiguity of how structs are in C. So we have the option of telling rust that, by this attribute.
Other than that you will notice that I have included something called "padding".
This is because in C style structs, u64 values are 8 byte aligned. E.g. they are placed in such a way so that Their address is divisible by 8. But u8 are 1 byte aligned.
Also since largest member in the struct is 8 aligned, the entire struct itself is also expected to be 8 aligned. That's why we have to keep a padding of 6 bytes. Because there will be a padding 6 bytes by default, we explicitly write it here to make the code more clear.

Now that we know what the struct looks like, we can go ahead in the asm, and save the register values to memory in the same format as this struct representation is expected to be.

And then lets say our Rust function that we will call looks like:

```
#[no_mangle]
pub extern "C" fn handle_exception_el1(ctx: &mut ExceptionContext) {
    ...
}
```

So our final pipeline for the exception will be:

Exception table entry points to an assembly label.
Jump to that label -> At that label, instructions exist to save registers to a 8 aligned memory address (lets call that address `ectx`). -> Also write appropriate values for etype and esource members of struct. -> write address `ectx` to x0 (as argument for the rust funciton) -> then finally `bl handle_exception_el1` to rust handler function. -> rust function returns and comes back -> LOAD the registers from memory back to original values. -> ERET.

Notice that when we call a Rust funciton using `bl`, then when the function concludes, it returns back to the location in original assembly where the `bl` was called.

You can store the register values in any location in memory. However, according to the official ARM standard, stack pointers are always 16 byte aligned. So we can simply write the entire struct on the stack (since 16 aligned address is automatically 8 aligned). Also, using the stack has many conveniences like nested exceptions working naturally.

So in our original pipeline. In assembly, we will first move the stack pointer by the amount of bytes needed for the entire struct. Then manually write all the registers and struct member values to memory, with reference to stack pointer. Then pass stack pointer to x0 and call rust handler function.

Note that from here since the assembly will become very repetitive if we try to write the entire handler pipeline 16 different times for each exception table entry.
So we will utilize something called "assembly macros". It is recommended to look it up before reading forward.

## Assembly for handing exception handling to Rust

To show you the general structure. For each of the 16 entries, we want to do the following:

```asm
.macro HANDLE_EXCEPTION type source
    sub     sp, sp, #0x120 // allocating space for etype + esource + gprs + 4 u64 reg
    
    // save registers
    SAVE_REG

    // call rust handler with correct arg
    SET_EXCEPTION_ARG \type \source
    bl      handle_exception_el1

    // load back the registers
    LOAD_REG

    add     sp, sp, #0x120 // restore sp

    eret // handling completed :)
.endm
```

In rust, the value of exception type can be 0 to 3 for sync, irq, fiq, and SError. And source can be 0 to 3 for `EL1t`, `EL1h`, `EL0 64 bit`, `EL0 32 bit`. As you can see in the Rust `enum` definitions earlier.

- First we move the pointer by subtracting the number of bytes we need. 
- `SAVE_REG` is a macro which saves the registers to the correct locations in memory with the stack pointer as the start of the struct.
- `SET_EXCEPTION` writes the `ExceptionType` and `ExceptionSource` appropriate values to the first and second byte of the struct. And it also writes the struct address to `x0`.
- Then we call the rust function. It will return back after handling the exception.
- `LOAD_REG` this macro reads the memory and writes the values from the struct in memory back to the original registers.
- Then since we don't need the bytes we used we can move the stack pointer forward again to the original position.

The subtracting and adding to stack pointer is to abide the method of pushing and popping from stack. In a way we are manually pushing to the stack by moving the stack and writing values to memory addresses after it. 
And then adding to the stack is us popping from the stack.

`SAVE_REG` macro works as follows:

```asm
.macro SAVE_REG
    stp     x0,  x1,  [sp, #0x08]
    stp     x2,  x3,  [sp, #0x18]
    stp     x4,  x5,  [sp, #0x28]
    stp     x6,  x7,  [sp, #0x38]
    stp     x8,  x9,  [sp, #0x48]
    stp     x10, x11, [sp, #0x58]
    stp     x12, x13, [sp, #0x68]
    stp     x14, x15, [sp, #0x78]
    stp     x16, x17, [sp, #0x88]
    stp     x18, x19, [sp, #0x98]
    stp     x20, x21, [sp, #0xA8]
    stp     x22, x23, [sp, #0xB8]
    stp     x24, x25, [sp, #0xC8]
    stp     x26, x27, [sp, #0xD8]
    stp     x28, x29, [sp, #0xE8]
    str     x30,      [sp, #0xF8]
    
    mrs     x0, ELR_EL1
    str     x0, [sp, #0x100]

    mrs     x0, SPSR_EL1
    str     x0, [sp, #0x108]

    mrs     x0, ESR_EL1
    str     x0, [sp, #0x110]

    mrs     x0, FAR_EL1
    str     x0, [sp, #0x118]
.endm
```
`stp` instruction is basically for writing two 8-byte registers to memory together as a pair. At some memory address.

First we write all GPRs. Then once we can safely modify x0 value, we use it to write the remaining four registers.

`LOAD_REG` macro is similar. 
```asm
.macro LOAD_REG
    // load these first so x1 won't be needed after
    ldr     x1, [sp, #0x100]
    msr     ELR_EL1, x1

    ldr     x1, [sp, #0x108]
    msr     SPSR_EL1, x1
    // we do not need to load back the other two registers

    ldp     x0,  x1,  [sp, #0x08]
    ldp     x2,  x3,  [sp, #0x18]
    ldp     x4,  x5,  [sp, #0x28]
    ldp     x6,  x7,  [sp, #0x38]
    ldp     x8,  x9,  [sp, #0x48]
    ldp     x10, x11, [sp, #0x58]
    ldp     x12, x13, [sp, #0x68]
    ldp     x14, x15, [sp, #0x78]
    ldp     x16, x17, [sp, #0x88]
    ldp     x18, x19, [sp, #0x98]
    ldp     x20, x21, [sp, #0xA8]
    ldp     x22, x23, [sp, #0xB8]
    ldp     x24, x25, [sp, #0xC8]
    ldp     x26, x27, [sp, #0xD8]
    ldp     x28, x29, [sp, #0xE8]
    ldr     x30,      [sp, #0xF8]
.endm
```

`SET_EXCEPTION_ARG` macro works as follows:
```asm
.macro SET_EXCEPTION_ARG type source
    mov     w9, #\type
    strb    w9, [sp]
    mov     w9, #\source
    strb    w9, [sp, #1]
    mov     x0, sp
.endm
```

the w9 register is another GPR. It is basically x9 register but for 32 bit mode. It is kind of useless in AArch64 mode. It basically points to the lower 4 bytes of the x9 register. But we use it since the store byte instruction `strb` only accepts one of the 4 byte `w0...w30` registers. `x9` would not be accepted. We use `strb` instruction because we only wish to write a single byte (the lowest byte of w9 is written).

I have labelled all the source and type values as well at the beginning of my assembly instructions.

```asm
.equ E_SYNC,   0 // to tell rust handler what the exception is
.equ E_IRQ,    1
.equ E_FIQ,    2
.equ E_SERROR, 3

.equ FROM_EL1t, 0
.equ FROM_EL1h, 1
.equ FROM_EL064, 2
.equ FROM_EL032, 3
```

So now, finally, the labels that each entry jumps to can be defined as:

```asm

el11t_sync:
    HANDLE_EXCEPTION E_SYNC FROM_EL1t
    
el11t_irq:
    HANDLE_EXCEPTION E_IRQ FROM_EL1t

el11t_fiq:
    HANDLE_EXCEPTION E_FIQ FROM_EL1t

el11t_serror:
    HANDLE_EXCEPTION E_SERROR FROM_EL1t


el11h_sync:
    HANDLE_EXCEPTION E_SYNC FROM_EL1h

el11h_irq:
    HANDLE_EXCEPTION E_IRQ FROM_EL1h

el11h_fiq:
    HANDLE_EXCEPTION E_FIQ FROM_EL1h

el11h_serror:
    HANDLE_EXCEPTION E_SERROR FROM_EL1h


el1064_sync:
    HANDLE_EXCEPTION E_SYNC FROM_EL064

el1064_irq:
    HANDLE_EXCEPTION E_IRQ FROM_EL064

el1064_fiq:
    HANDLE_EXCEPTION E_FIQ FROM_EL064

el1064_serror:
    HANDLE_EXCEPTION E_SERROR FROM_EL064


el1032_sync:
    HANDLE_EXCEPTION E_SYNC FROM_EL032

el1032_irq:
    HANDLE_EXCEPTION E_IRQ FROM_EL032

el1032_fiq:
    HANDLE_EXCEPTION E_FIQ FROM_EL032

el1032_serror:
    HANDLE_EXCEPTION E_SERROR FROM_EL032
```

And that wraps up the Assembly code! you can wrap it up in some file like `src/exception.s`. And then give your assembly section a name at the top like

```asm
.section ".text.vectors"
```

And then simply modify linker script `linked.ld` to place the exception table in memory wherever you wish.

```ld
    .text :
    {
        *(.text.boot)
        *(.text.vectors)
        *(.text*)
    }
```

## Completing the Rust function

Now that your assmebly is correctly saving exception context to memory and passing it to a rust functions you just need to work in rust from now!
For now, we're not going to do anything crazy, lets just make the handler print all the exception context it is receiving, as a test.

```rust
// called by `exceptions.s`
#[no_mangle]
pub extern "C" fn handle_exception_el1(ctx: &mut ExceptionContext) {
    
    println!("An exception has been detected :D").unwrap();
    
    // printing the full context for now.
    let etype_str = match ctx.etype {
        ExceptionType::_SYNC => "SYNC",
        ExceptionType::_IRQ  => "IRQ",
        ExceptionType::_FIQ  => "FIQ",
        ExceptionType::_SE   => "SError",
    };
    let esource_str = match ctx.esource {
        ExceptionSource::_EL1t => "EL1t",
        ExceptionSource::_EL1h => "EL1h",
        ExceptionSource::_EL064 => "EL064",
        ExceptionSource::_EL032 => "EL032",
    };
    println!("=== Exception Context ===").unwrap();
    println!("Type : {} ({})", etype_str, ctx.etype as u8).unwrap();
    println!("Source : {} ({})", esource_str, ctx.esource as u8).unwrap();
    println!("ELR  : {:#018x}", ctx.elr).unwrap();
    println!("SPSR : {:#018x}", ctx.spsr).unwrap();
    println!("ESR  : {:#018x}", ctx.esr).unwrap();
    println!("FAR  : {:#018x}", ctx.far).unwrap();
    println!("Registers:").unwrap();
    for i in 0..31 {
        println!("  x{:02} = {:#018x}", i, ctx.x[i]).unwrap();
    }
    println!("=========================").unwrap();
}
```

# Testing

Of course, to test the exception handler, you need to first cause an exception.

There is an instruction `svc`. Which is used to manually cause synchronous exceptions. It has many uses. But right now it will help us cause an exception so we can watch our handler handle it.

So now in your `main.rs`, Somewhere in the `_rust_main` function add:

```rs
println!("\r\nWelcome to, AtOS.").unwrap();
println!("Current EL is: EL{}", get_current_el()).unwrap();
unsafe { core::arch::asm!("svc #0"); }
println!("Returned from exception!").unwrap();
```

Now simply build your `kernel8.img`, and then write it to your memory card and boot it on your RPi. Or, build your `kernel8.img` and then run it in QEMU (Raspberry Pi 3b+ emulator) as follows:

```bash
$ qemu-system-aarch64 -M raspi3b -kernel kernel8.img -serial null -serial stdio
```
(If you don't have qemu, you can get it from your respective package manager.)


And then, we will see something like:

```
Welcome to, AtOS.
Current EL is: EL1
An exception has been detected :D
=== Exception Context ===
Type : SYNC (0)
Source : EL1h (1)
ELR  : 0x0000000000082598
SPSR : 0x0000000060000345
ESR  : 0x0000000056000000
FAR  : 0x0000000000000000
Registers:
  x00 = 0x0000000000000000
  x01 = 0x0000000000084928
  x02 = 0x0000000000084928
  x03 = 0x0000000000081fe0
  x04 = 0x0000000000000000
  x05 = 0x0000000000000001
  x06 = 0x0000000000000000
  x07 = 0x0000000000000000
  x08 = 0x0000000000000000
  x09 = 0x00000000000856b8
  x10 = 0x000000000000000a
  x11 = 0x0000000000000060
  x12 = 0x0000000000000000
  x13 = 0x0000000000000000
  x14 = 0x0000000000000000
  x15 = 0x0000000000000000
  x16 = 0x0000000000000000
  x17 = 0x0000000000000000
  x18 = 0x0000000000000000
  x19 = 0x000000003f215040
  x20 = 0x0000000000000000
  x21 = 0x00000000000833a8
  x22 = 0x0000000000000000
  x23 = 0x0000000000000000
  x24 = 0x0000000000000000
  x25 = 0x0000000000000000
  x26 = 0x0000000000000000
  x27 = 0x0000000000000000
  x28 = 0x0000000000000000
  x29 = 0x0000000000000000
  x30 = 0x00000000000824e8
=========================
Returned from exception!
```

And that means that exception handling is successfully implemented.

# Final codes

`src/kernel/exceptions.s`
```asm
.section ".text.vectors"
.global el1_vectors

.equ E_SYNC,   0 // to tell rust handler what the exception is
.equ E_IRQ,    1
.equ E_FIQ,    2
.equ E_SERROR, 3

.equ FROM_EL1t, 0
.equ FROM_EL1h, 1
.equ FROM_EL064, 2
.equ FROM_EL032, 3

// VECTOR TABLE FOR EXCEPTIONS AT EL1

.align 11              // 2^11 = 2048 alignment

el1_vectors:

b   el11t_sync
.space 124
b   el11t_irq
.space 124
b   el11t_fiq
.space 124
b   el11t_serror
.space 124

b   el11h_sync
.space 124
b   el11h_irq
.space 124
b   el11h_fiq
.space 124
b   el11h_serror
.space 124

b   el1064_sync
.space 124
b   el1064_irq
.space 124
b   el1064_fiq
.space 124
b   el1064_serror
.space 124

b   el1032_sync
.space 124
b   el1032_irq
.space 124
b   el1032_fiq
.space 124
b   el1032_serror
.space 124


// HANDLERS FOR EL1 EXCEPTIONS

.macro SAVE_REG
    stp     x0,  x1,  [sp, #0x08]
    stp     x2,  x3,  [sp, #0x18]
    stp     x4,  x5,  [sp, #0x28]
    stp     x6,  x7,  [sp, #0x38]
    stp     x8,  x9,  [sp, #0x48]
    stp     x10, x11, [sp, #0x58]
    stp     x12, x13, [sp, #0x68]
    stp     x14, x15, [sp, #0x78]
    stp     x16, x17, [sp, #0x88]
    stp     x18, x19, [sp, #0x98]
    stp     x20, x21, [sp, #0xA8]
    stp     x22, x23, [sp, #0xB8]
    stp     x24, x25, [sp, #0xC8]
    stp     x26, x27, [sp, #0xD8]
    stp     x28, x29, [sp, #0xE8]
    str     x30,      [sp, #0xF8]
    
    mrs     x0, ELR_EL1
    str     x0, [sp, #0x100]

    mrs     x0, SPSR_EL1
    str     x0, [sp, #0x108]

    mrs     x0, ESR_EL1
    str     x0, [sp, #0x110]

    mrs     x0, FAR_EL1
    str     x0, [sp, #0x118]
.endm

.macro LOAD_REG
    // load these first so x1 won't be needed after
    ldr     x1, [sp, #0x100]
    msr     ELR_EL1, x1

    ldr     x1, [sp, #0x108]
    msr     SPSR_EL1, x1
    // we do not need to load back the other two registers

    ldp     x0,  x1,  [sp, #0x08]
    ldp     x2,  x3,  [sp, #0x18]
    ldp     x4,  x5,  [sp, #0x28]
    ldp     x6,  x7,  [sp, #0x38]
    ldp     x8,  x9,  [sp, #0x48]
    ldp     x10, x11, [sp, #0x58]
    ldp     x12, x13, [sp, #0x68]
    ldp     x14, x15, [sp, #0x78]
    ldp     x16, x17, [sp, #0x88]
    ldp     x18, x19, [sp, #0x98]
    ldp     x20, x21, [sp, #0xA8]
    ldp     x22, x23, [sp, #0xB8]
    ldp     x24, x25, [sp, #0xC8]
    ldp     x26, x27, [sp, #0xD8]
    ldp     x28, x29, [sp, #0xE8]
    ldr     x30,      [sp, #0xF8]
.endm

// handling them
.macro SET_EXCEPTION_ARG type source
    mov     w9, #\type
    strb    w9, [sp]
    mov     w9, #\source
    strb    w9, [sp, #1]
    mov     x0, sp
.endm

.macro HANDLE_EXCEPTION type source
    sub     sp, sp, #0x120 // allocating space for etype + esource + gprs + 4 u64 reg
    // make sure sp is aigned to 16 bytes for rust handler according to arm standard

    // save registers
    SAVE_REG

    // call rust handler with correct arg
    SET_EXCEPTION_ARG \type \source
    bl      handle_exception_el1

    // load back the registers
    LOAD_REG

    add     sp, sp, #0x120 // restore sp

    eret // handling completed :)
.endm

el11t_sync:
    HANDLE_EXCEPTION E_SYNC FROM_EL1t
    
el11t_irq:
    HANDLE_EXCEPTION E_IRQ FROM_EL1t

el11t_fiq:
    HANDLE_EXCEPTION E_FIQ FROM_EL1t

el11t_serror:
    HANDLE_EXCEPTION E_SERROR FROM_EL1t


el11h_sync:
    HANDLE_EXCEPTION E_SYNC FROM_EL1h

el11h_irq:
    HANDLE_EXCEPTION E_IRQ FROM_EL1h

el11h_fiq:
    HANDLE_EXCEPTION E_FIQ FROM_EL1h

el11h_serror:
    HANDLE_EXCEPTION E_SERROR FROM_EL1h


el1064_sync:
    HANDLE_EXCEPTION E_SYNC FROM_EL064

el1064_irq:
    HANDLE_EXCEPTION E_IRQ FROM_EL064

el1064_fiq:
    HANDLE_EXCEPTION E_FIQ FROM_EL064

el1064_serror:
    HANDLE_EXCEPTION E_SERROR FROM_EL064


el1032_sync:
    HANDLE_EXCEPTION E_SYNC FROM_EL032

el1032_irq:
    HANDLE_EXCEPTION E_IRQ FROM_EL032

el1032_fiq:
    HANDLE_EXCEPTION E_FIQ FROM_EL032

el1032_serror:
    HANDLE_EXCEPTION E_SERROR FROM_EL032
```

`src/kernel/exceptions.rs`
```rs
use crate::println;

#[repr(u8)]
#[derive(Copy, Clone)]
pub enum ExceptionType {
    _SYNC,
    _IRQ,
    _FIQ,
    _SE,
}

#[repr(u8)]
#[derive(Copy, Clone)]
pub enum ExceptionSource {
    _EL1t,
    _EL1h,
    _EL064,
    _EL032,
}


#[repr(C)]
pub struct ExceptionContext {
    pub etype: ExceptionType, // u8
    pub esource: ExceptionSource, // u8use crate::println;

#[repr(u8)]
#[derive(Copy, Clone)]
pub enum ExceptionType {
    _SYNC,
    _IRQ,
    _FIQ,
    _SE,
}

#[repr(u8)]
#[derive(Copy, Clone)]
pub enum ExceptionSource {
    _EL1t,
    _EL1h,
    _EL064,
    _EL032,
}


#[repr(C)]
pub struct ExceptionContext {
    pub etype: ExceptionType, // u8
    pub esource: ExceptionSource, // u8
    pub _padding: [u8; 6], // because this struct follows c style repr
    pub x: [u64; 31],   // x0–x30
    pub elr: u64,
    pub spsr: u64,
    pub esr: u64,
    pub far: u64,
}

// called by `exceptions.s`
#[no_mangle]
pub extern "C" fn handle_exception_el1(ctx: &mut ExceptionContext) {
    
    println!("An exception has been detected :D").unwrap();
    
    // printing the full context for now.
    let etype_str = match ctx.etype {
        ExceptionType::_SYNC => "SYNC",
        ExceptionType::_IRQ  => "IRQ",
        ExceptionType::_FIQ  => "FIQ",
        ExceptionType::_SE   => "SError",
    };
    let esource_str = match ctx.esource {
        ExceptionSource::_EL1t => "EL1t",
        ExceptionSource::_EL1h => "EL1h",
        ExceptionSource::_EL064 => "EL064",
        ExceptionSource::_EL032 => "EL032",
    };
    println!("=== Exception Context ===").unwrap();
    println!("Type : {} ({})", etype_str, ctx.etype as u8).unwrap();
    println!("Source : {} ({})", esource_str, ctx.esource as u8).unwrap();
    println!("ELR  : {:#018x}", ctx.elr).unwrap();
    println!("SPSR : {:#018x}", ctx.spsr).unwrap();
    println!("ESR  : {:#018x}", ctx.esr).unwrap();
    println!("FAR  : {:#018x}", ctx.far).unwrap();
    println!("Registers:").unwrap();
    for i in 0..31 {
        println!("  x{:02} = {:#018x}", i, ctx.x[i]).unwrap();
    }
    println!("=========================").unwrap();
}
    pub _padding: [u8; 6], // because this struct follows c style repr
    pub x: [u64; 31],   // x0–x30
    pub elr: u64,
    pub spsr: u64,
    pub esr: u64,
    pub far: u64,
}

// called by `exceptions.s`
#[no_mangle]
pub extern "C" fn handle_exception_el1(ctx: &mut ExceptionContext) {
    
    println!("An exception has been detected :D").unwrap();
    
    // printing the full context for now.
    let etype_str = match ctx.etype {
        ExceptionType::_SYNC => "SYNC",
        ExceptionType::_IRQ  => "IRQ",
        ExceptionType::_FIQ  => "FIQ",
        ExceptionType::_SE   => "SError",
    };
    let esource_str = match ctx.esource {
        ExceptionSource::_EL1t => "EL1t",
        ExceptionSource::_EL1h => "EL1h",
        ExceptionSource::_EL064 => "EL064",
        ExceptionSource::_EL032 => "EL032",
    };
    println!("=== Exception Context ===").unwrap();
    println!("Type : {} ({})", etype_str, ctx.etype as u8).unwrap();
    println!("Source : {} ({})", esource_str, ctx.esource as u8).unwrap();
    println!("ELR  : {:#018x}", ctx.elr).unwrap();
    println!("SPSR : {:#018x}", ctx.spsr).unwrap();
    println!("ESR  : {:#018x}", ctx.esr).unwrap();
    println!("FAR  : {:#018x}", ctx.far).unwrap();
    println!("Registers:").unwrap();
    for i in 0..31 {
        println!("  x{:02} = {:#018x}", i, ctx.x[i]).unwrap();
    }
    println!("=========================").unwrap();
}
```

`main.rs`
```rs
#![no_std]
#![no_main]

mod kernel;

use core::panic::PanicInfo;

#[no_mangle]
pub extern "C" fn _start() -> ! {
    println!("\r\nWelcome to, AtOS.").unwrap();
    println!("Current EL is: EL{}", get_current_el()).unwrap();
    unsafe { core::arch::asm!("svc #0"); }
    println!("Returned from exception!").unwrap();

    loop {}
}

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    println!("Some exception happened!").unwrap();
    loop {}
}
```
