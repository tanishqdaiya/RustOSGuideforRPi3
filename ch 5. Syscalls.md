# User space isolation

So far what we've done is that we have created a separate project directory which acts as the place where all the user's programs and standard library will be written. Basically all the stuff that is supposed to run ON our operating system. However why did we create a separate project for it? Why couldn't we just keep growing our original operating system repository? Well, there's mainly two reasons.

the first reason is that user program is not just supposed to be the operating system related processes that run for the user like the shell, window manager, desktop environment, etc. The user programs can also be third party codes or applications that are written by people who are not associated with us or our operating system's development. Those third parties should have a way to write applications that can run on our operating system without having to know how the OS itself handles the hardware. So this produces sort of a design philosophy based reason to separate the user program space including the user standard library, from the kernel project itself. 

When a third party includes or imports our standard library that we wrote, they will also work in a project environment completely disconnected from our OS kernel source code. And so we make the entire standard library as a standard rust library package project. 

The second reason is because of security. We know that by letting third parties create their own programs for our operating system, we allow them to freely introduce new amazing features or programs that do cool things on our operating system. However there can be malicious third parties which for whatever reason, simply wish to cause chaose on the hardware which is running our operating system. They might try to directly access RAM to violate the memory space used by other running programs. They might try to access the disk/microSD in order to corrupt files. They might try to access the MMIO addresses in order to control the rest of the hardware maliciously.

This is the entire reason we drop to EL0 before letting a user program execute. This is because from kernel which runs in EL1, we can manually tell the hardware to block/moderate EL0's direct access to the hardware. Completely blocking the program running in EL0 from even being able to see any other process's memory in RAM, or from even being able to access the MMIO or disk.

Thus, user programs should never expect to be able to access the hardware directly like the kernel can. So we also separate our rust project for our user standard library to a different cargo project. If user programs were compiled within `kernel8.img` then it would be more complicated to separate them and isolate them for running them in EL0.

# Syscalls

However, if the user programs cannot even expect any access to the hardware, how would they even interact with the machine and do whatever they are designed to do? How could they print output to the user without access to a display/UART? How could they read input from the user without reading data from the keyboard/io/UART_RX? 

To understand lets first define these tasks which require direct sensitive accessing of the hardware as "priviledged jobs" and let the accessing type itself be called "priviledged action". 

Now, to put it plainly, most operating system standards actually expect the user program to send a request to the kernel for the priviledged job. From there, the kernel identifies what sort of priviledged job the user program wishes to perform. And then the *kernel performs* the priviledged task for the user in a safe controlled manner. The user program then receives the result of the privlidged action from the kernel. Usually in the form of return value in a register or at a place in memory that is accessible to the particular user program.

This "request" to the kernel from the user program is called a "syscall". In the following text you will learn how to make your user program send these "syscalls" to the kernel and how the kernel can indentify them, handle them, and then send responses back to the user program.

# Trapping to the kernel

In standard lingo you will often hear the word "trap". Usually when textbooks or lectures talk about the user program sending a request to the kernel, They call it as the user program "trapping into the kernel". What they mean by that is that like how we drop from EL1 to EL0 using the `ERET` method, we can also raise the level from EL0 to EL1. And just like how when dropping to EL0 we start off in the user program, i.e. the moment you drop to EL0 we simultanously switch to the user program's instructions in memorry. We also start in the kernel when we raise the exception level to EL1. This is usually achieved by a dedicated instruction that the EL0 program can execute which causes this to happen. This entire process of using the special instruction in a low priviledge level to manually send the priviledge to a higher level while handing over the control to the kernel is generally called as "trapping into the kernel".

In ARM architecture, the special instruction that lets us send execution to kernel in EL1 is `svc`. We will learn about this instruction more. But it is short for "Supervisor Control". The word "supervisor" is an old classic name of the "kernel". It accepts one immediate argument, so the full way of writing it wuld be somethign like `svc #imm` where `imm` can be any number from 0 to 65535 (`0xFFFF`)

When `svc` instruction is executed, the behavior that is triggered in the hardware can be described as following:
- A hardware exception occurs.
- The exception is taken to EL1. 
- The immediate argument to the `svc` instruction is encoded in the `ESR_EL1` register.
- The type of exception is "Synchronous exception". So the execution jumps to the appropriate entry in `VBAR_EL1` exception table, for `EL0 64 bit` source (since currently our user program executes in `EL0 64 bit` mode).

Thus the kernel's exception handler executes upon the `svc` instruction. The `svc` instruction does not touch any GPR values. So the user program has the option to store relevant information in the general purpose registers before using `svc`. Wherein the kernel's exception handler can identify the values written in the registers. This is how the communication from the user program to kernel occurs for the syscalls.

In linux kernel, different priviledged actions like "write to disk" or "read from disk" or "create new process" are all given a unique identification number. Then before `svc` the user program stores the ID of the requested priviledged action in the `x8` register, which the kernel reads and indentifies. Once the kernel is done performing the operation requested it then does ERET from the exception back to where EL0 did `svc`. Any relevant return values are also stored in GPRs.

# Implementing syscalls

We are going to do a similar procedure. For the first syscall we will try to implement a `println!` macro for the user space which works using syscalls instead of accessing the MMIO directly. 

So recall from the last chapter that for the sake of testing we just copy pasted the entire UART module from the kernel's code into the user project. As discussed earlier the user programs should never expect to be given access to the MMIO addresses freely. So currently the way the user program does `println!` is incorrect. Even though it functions, the moment we introduce EL0 restrictions to prevent user programs from accessing MMIO, it will break and not work anymore. 

Thus this print function will be turned into a syscall. We're going to remove the UART/peripheral code we copy pasted from kernel, and write brand new way of printing to the output. And this new way will actually just do `svc` whenever we want to print something, and then the kernel will do the printing for us, and return. It is going to be a **syscall**.

So this involves two aspects, firstly in the user project, we write the print function that will actually do `svc` under the hood.

First of all in the user project create a new directory `"src/stdlib"`. This directory will have all the standard library source code that our user programs will be able to use. Then create files `stdlib/mod.rs` and `stdlib/syscalls.rs`.

Now in `mod.rs` write:
```rs
pub mod syscalls;
```
so now the folder "syscalls" will be treated as a rust module with `syscalls.rs` as a part of it.

Before we start implementing the syscall in `syscall.rs` lets first cleanup `lib.rs`. First clean everything off. and only leave:

```rs
#![no_std]

pub mod stdlib;
```

Now finally in `syscalls.rs` we can write the print functionality.

```rs
pub fn _print(s: &str) -> core::fmt::Result {
    unsafe {
        core::arch::asm!(
            "svc #1",
            in("x0") s.as_ptr(),
            in("x1") s.len(),
        );
    }
    Ok(())
}
```

Here as you can see, the function here basically just takes a string. Then it puts the location and size of the string in `x0` and `x1` registers, and executes the instruction `svc #1`.

Why `#1`? And why use `x0` and `x1` registers? Well the answer is that it is entirely up to you! In linux, the instruction is always `svc #0` and the ID of the syscall is passed in the `x8` register. However you can also pass the ID as the immedaite argument of the `svc` instruction. It is entirely your choice to decide what way is better. While passing the ID as immediate value is more secure according to some sources, it is also slightly inefficient to read it for the kernel. Either way it is your operating system, so you get to decide how it works.

For my OS I chose that the print functionality will have syscall ID number 1. And of course the kernel needs to be able do know what the user program wants printed, and that we pass to the kernel in the `x0` and `x1` registers.

Now, functionality wise this is quiet functional. However this does not accept any formatted strings. Now that you have it working, you can work on making it cleaner. Of course your operating system's code organization and structure is also entirely up to you. But I chose to create a `Stdout` struct as an abstraction for the standard output pipeline, and implemented the syscall into that struct. This is similar to the `Uart` struct we create inside the `kernel::peripherals`. However the user program shouldn't assume the hardware, it shouldn't try to assume or expect that the output will happen to the UART. All the user program should know is that it can do output by prompting the kernel in EL1. That's why we don't name our struct "Uart" but instead "Stdout". It is to reflect the generalized view of the user program. Since user programs should not have to worry about the hardware being used and be able to conveniently work with the standard library through generic crossplatform terms like stdout.


My implementation ends up being as follows:

```rs
use core::{fmt, fmt::Write};

/* ~~~ STDIO ~~~ */
// For printing or getting input from the stdio (UART).
// printing is assigned syscall number 1 (svc #1),
// and getting input is assigned syscall number 2 (svc #2)
// \TODO INPUT HANDLING
pub struct Stdout;

impl fmt::Write for Stdout {
    fn write_str(&mut self, s: &str) -> fmt::Result {
        unsafe {
            core::arch::asm!(
                "svc #1",
                in("x0") s.as_ptr(),
                in("x1") s.len(),
            );
        }

        Ok(())
    }
}


pub fn _print(args: fmt::Arguments) -> fmt::Result {
    Stdout.write_fmt(args)
}
```

And then of course we create the actual `print!` and `println!` macros. 

```rs
#[macro_export]
macro_rules! print {
    ($($arg:tt)*) => ({
        $crate::stdlib::syscalls::_print(
            core::format_args!($($arg)*)
        )
    });
}

#[macro_export]
macro_rules! println {
    ($($arg:tt)*) => ({
        $crate::stdlib::syscalls::_print(
            core::format_args!(
                "{}\n",
                core::format_args!($($arg)*)
            )
        )
    });
}
```

And that is done from the user side!

Now when the user program do user::println! it is going to send the string to the kerenel through `svc`.

Now as for the part where the kernel handles the syscall. 

As we've learned that the `svc` instruction summons to EL1 by causing an hardware exception to occur to EL1's exception handler. If you recall that it causes a synchronous exception. So in our exception handler, to identify if an exception was caused by `svc` instruction we can simply check if it was a synchronous exception.

So far our exception handler had no real use. All it was doing is printing the entire context of the exception whenever an exception occured. However now we're going to actually put it to use. Instead of immediately printing the exception contex and calling it a day, we're going to make it `match` the exception type, and if it is a synchronous exception, we're going to handle it accordingly. Something like:

```rs
// called by `exceptions.s`
#[unsafe(no_mangle)]
pub extern "C" fn handle_exception_el1(ctx: &mut ExceptionContext) {
    
    // handling the exception based on the type.
    match ctx.etype {
        ExceptionType::_SYNC => handle_sync_exception(ctx),
        _ => unhandled_exception!(ctx),
    }

}
```

Here, we're going to shortly define the `handle_sync_exception` function and the `unhandled_exception!` macro.

First of all for the unhandled exception. Right now we're only identifying for a synchronous type exception, however it is possible that some other type of exceptions might also occur. It is not as though we've studied the entire ARM architecture thoroughly. So any other exceptions that might occur we consider them "unhandled" or "unexpected". 

When an unhandled exception occurs you can always just ignore it. But what you could also do is print the context so even if our code can't identify it, the person using (the developer) the OS can use the context to determine what went wrong. 

So the code from previous chapters about printing the exception context can be put into a convenient function.

```rs
fn print_exception_context(ctx: &ExceptionContext) -> () {
    let etype_str = match ctx.etype {
        ExceptionType::_SYNC => "SYNC",
    
    (abbreviated...)
    
        println!("  x{:02} = {:#018x}", i, ctx.x[i]).unwrap();
    }
    println!("=========================").unwrap();
}
```

And then the macro can be created:

```rs
macro_rules! unhandled_exception {
    ($ctx:expr) => {{
        println!("Unhandled exception detected!").unwrap();
        print_exception_context($ctx);
    }};
}
```

That's all for the `match` route where the exception is anything other than the ones we expect. For a SYNC exception, we don't want to just ignore it, we have to identify if it came from a `svc`, and then identify the immediate argument used in that instruction (since it gives the syscall's ID), and perform respective syscall's priviledged action.

Firstly, for identifying if it came from `svc` instruction:

When a synchronous exception occurs, the exception syndrome register `ESR_EL1` encodes information that helps identify the exception type. There are many fields in this register. The main one you need to know about are the bits [31:26]. These six bits make up a field which is named "EC" for "Exception class". According to the [official ESR_EL1 documentation](https://developer.arm.com/documentation/ddi0595/2020-12/AArch64-Registers/ESR-EL1--Exception-Syndrome-Register--EL1-?lang=en#fieldset_0-31_26), this field contains the value "`0x15`" when the synchronous exception occured due to an `svc` instruction.

So now you know the exception occured due to a syscall. All we need to do in exception handler is to call the function which will deal take it from there and do the actual syscall identifying and performing-- the syscall handler basically. For the syscall handler, we're going to put it in a different file in `kernel::syscalls` at `src/kernel/syscalls.rs`. 

For now in `exceptions.rs` you can define the sync handler as: 

```rs
fn handle_sync_exception(ctx: &ExceptionContext) -> () {
    let exception_class = (ctx.esr >> 26) & 0x3f;

    match ctx.esource {
        ExceptionSource::_EL064=> {
            match exception_class {
                0x15 => { // it was an svc instruction
                    syscalls::handle_syscall(ctx);
                },
                _ => unhandled_exception!(ctx),
            }
        },
        _ => unhandled_exception!(ctx),
    }
}
```

As you can see, we basically read the bits [31:26] to identify if it is a `svc` caused instruction. If it is, we simply call `syscalls::handle_syscall` (Don't worry we will define this in a bit). All other cases are treated as unhanlded. 

We also here check if the exception can from EL0 or not. Obviously no user program will ever be permitted to run in EL1. So it makes sense that all syscall requests will only come form EL0. We've not used `svc` in any other EL even once. So if we get a `svc` caused exception from some other source, it will definitely be really suspicious, and thus we treat it as unhanlded so the OS appropriately freaks out about it. 

Finanlly in `syscalls.rs`. We're going to create the sync handler. Now, you can easily get `x0` and `x1` from the ExceptionnContext. But the syscall handler will need to read the syscall ID somehow. Which was passed as an immediate argument to the `svc` instruction.

Again, according to the [official ARM documentation](https://developer.arm.com/documentation/ddi0595/2020-12/AArch64-Registers/ESR-EL1--Exception-Syndrome-Register--EL1-?lang=en#fieldset_0-24_0_9-15_0), the `ESR_EL1 register's` bits [24:0] make up a field which is named "ISS" for "Instruction Specific Syndrome". And when the exception was caused due to a `svc`, the last 16 bits of this ISS field contain the immediate argument passed to the `svc` instruction. The other bits [24:16] are reserved to zero (basically they don't mean anything and have no use, and reading them will just give you zeroes).

Thus, to get the syscall ID you can read `ESR_EL1[15:0]`. That is precisely what our handler is going to do. So far we only have decided the syscall for print, which uses ID 1. So if the ID is read as 1, we need to do the appropriate function. Which is reading the string's position and size from `x0` and `x1` (since that is where the user program passes them), and print that string to the output for the user program. This is implemented as follows:

`src/kernel/syscalls.rs`
```rs
use crate::print;
use crate::kernel::exceptions::ExceptionContext;

pub fn handle_syscall(ctx: &ExceptionContext) -> () {
    // when it was because of `svc`, 
    // lower 16 bits of the ESR will have the syscall number.
    let syscall_number = ctx.esr & 0xffff; 

    match syscall_number {
        1 => sys_print(ctx).unwrap(),
        _ => {
            print!("Unknown syscall: {}", ctx.x[8]).unwrap();
        }
    }
}

/* SYSCALL #1 -- PRINT */
// expects x0 to have the pointer to the string and x1 to have the length of the string. 
fn sys_print(ctx: &ExceptionContext) -> core::fmt::Result {
    let ptr = ctx.x[0] as *const u8;
    let len = ctx.x[1] as usize;

    let s = unsafe { core::slice::from_raw_parts(ptr, len) };
    let s = core::str::from_utf8(s).unwrap_or("");

    print!("{}", s).unwrap();
    Ok(())
}
```

of course do not forget to add `pub mod syscalls;` to `kernel/mod.rs`

And that is all! Once the syscall handler returns out to the exception handler, the exception handler will return out using `ERET` and restore the user program's context to the hardware. So after ERET we drop back to the place where the exception came from (which was EL0 64 bit mode at the user program). And execution continues at that place. So to the user program it looks like all that happened was that it used the `svc` instruction and then the execution moved on to the next line. But under the hood the `svc` caused an exception to the kernel's exception handler which did the operation requested by the `svc`.

And that is the syscall pipeline! Now whenever our user program wants to print anything, it will be able to seemingly do it without touching the MMIOs. This is more secure since the kernel can now inspect what the user program wants to be done. And accordingly either accept the request or complain about it being unsafe.

## Security

In our case our project is small in scope so we're not going to focus on security a lot. But one method that so far the user program could try to do something malicious is instead of passing a valid string to the kernel, it might put inside `x0`, a pointer to some random data. Which will cause our kernel to panic. This is of course going to crash the kernel. We're not going to talk about handling that scenario right now, but if you wish you may take it as an exercise for yourself. Perhaps Instead of doing `sys_print(ctx).unwrap()` in syscall handler, you could make the `sys_print` function itself encode a success code in one of the GPRs which the user program can read back to know if the request was successful or not. 

## Final codes.

The final codes can be found in my original project `AtOS`'s GitHub reposiory. At the file system snapshot after the syscalls commit. 

Find the files at the below link. 

https://github.com/ZackyGameDev/AtOS/tree/3aefd06111c004eae08ada9204a61e9242fe9a8b

(kindly ignore the timer.rs module along with the main.rs state)