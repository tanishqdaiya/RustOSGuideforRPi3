# Chapter 4: First User Program

Your operating system is not for running by itself. Ultimately of course the goal would be for it to run processes for the user, and for the user to be able to run whatever process they want on it. Which, of course, our OS will need to achieve in a safe manner. Without letting the user processes access the hardware through any CPU or MMIO registers which could be dangerous. Our ultimate goal for our operating system would be to facilitate a safe usage of hardware by the processes, in a asynchronous manner. That is, in a manner where multiple processes can run at the same time.

## But what is a Process? 

Your computer or phone may have many different apps or programs on it. Normally something like your calculator app is not running on your device. It is usually stored on your storage/disk. However, when you wish to open the calculator app, somehow your operating system turns that bit of program on your disk into a running functioning execution of code. To put it plainly, the operating system reads the instructions and data of the app, and writes it to the memory (RAM). And then it sets `PC` to the first instruction meant to be executed from this newly loaded set of instructions in memory. Execution then continues and your app appears to run.

However, your app/program is still on your disk. It's not as if when the program is loaded to memory, it is erased from the disk. If that were the case, then every time a power outage happened in the middle of execution, your app would disappear from your device (since RAM memory is lost upon power loss). One could say that the program and data of the application on your disk is just there for your OS to load the correct instructions into memory. However, once those instructions are loaded into memory and they began executing, what do you call them? What do you call this bit of chunk in your memory that is executing instructions of your application? It needs a name to. And we call it a "Process".

Simply put, a process is running instance in memory of any program. At a bare minimum the way of creating a process is pretty similar to loading our `kernel8.img`. For a process, you go through its data/files on the disk to figure out what its image in memory will look like. And then you simply place it in memory at an appropriate memory location. Then you set a stack pointer for it, and simply set `PC` to the entry point of that process, i.e. the first instruction meant to execute for this new set of instructions. However, it differs from our `kernel8.img` in the sense that our operating system will keep track of processes and give them an identity. They will have a unique identification number, a name, information about who created them, their memory addresses, etc. Our operating system will then control how they interact with the hardware and decide which process gets to run at the moment. We will also manage new processes running and terminating. 

## Loading a simple process

A process will have many different parts to it. Also, deciding where to place the process and how to prevent it from accessing MMIO addresses will be a complicated procedure covered under Memory Management. Memory management is something which we will implement after we have some basic processes running and executing together on the CPU. For now, we will first create a process image on our host system itself, and just include it in our `kernel8.img`. Since it is included in `kernel8.img`, it will be loaded alongside to memory when RPi loads `kernel8.img` to memory. From their our kernel will copy that process image from `kernel8.img` section, over to where it is actually supposed to be. 

How do we decide where to place it? First of all, the user processes will be compiled in a manner extremely similar to how our kernel is compiled right now. We will create a separate cargo project named "user" which will also target baremetal AArch64 Raspberry Pi platform. And exactly like how our kernel project has a linking script which tells our compiler where the kernel will be loaded in memory (which is at 0x80000), we will also have a linker script in the `user` project, where instead of 0x80000, we will decide where we will place our process in memory. We will then write the linker script to start at our chosen address (any free address in memory can work).

Then again, the same way we first compile our OS project to an ELF, then a kernel8.img is dumped out of that elf. We will also dump an image from the compiled ELF produced in the user project. And then this image will be loaded by our running OS kernel into memory at the correct intended location. 

Then for starting the execution of the loaded program. We can simply check the original compiled ELF to find out the entry point address for the image. Then simply jumping to that address in memory.

However, recall that we actually are supposed to have user programs to run in EL0. And also that any running program executing in EL0 is actually going to use the `SP_EL0` register to find the stack pointer. So we will need to setup that as well. It is a very similar process to how we dropped from EL2 to EL1 into our kernel. As we will see.

## User project

Firstly lets start off by creating said user project. with the following in our project directory at `src/`.

```cargo new user```

Then `cargo.toml` can be something like:

```toml
[package]
name = "user"
version = "0.1.0"
edition = "2024"
authors = ["ZackyGameDev <zaacky2456@gmail.com>"]
```

Next up, in this project we will have two things we need to write.

1. **The different user programs.** Of course we will train our OS to handle multiple processes running on it at the same time. And for achieving that, we are going to need different user programs that will be loaded into the memory at separate places at the same time.

2. **a common standard library.** Right now our user programs are going to be able to access MMIO memory address freely. However, in the future we are going to limit it to prevent user programs from doing that. But then how will a user program print something to the output if it cannot access UART registers through MMIO? The answer is that when the user program requires some operation which needs to access MMIO or any other priviledged/dangerous feature, it will need to send a request to kernel. These request are handled through "syscalls". And all of these syscalls will be accessible to the user program in common module that is generally called the standard library.

The standard library also defines higher level featuers which aren't syscalls by themselves, but may rely heavily on syscalls to function. Or provide other structures and methods that a common user might need (for e.g. `String` struct in `std` in Rust).

The way we are going to do this is that the entire user crate that we have created will be a space for our stdlib source code. create a file `src/lib.rs` and unlike `main.rs`, `lib.rs` signifies to rust compiler that this project is meant to be used as a library for other rust projects. Once it is done we are going to create another directory `src/bin/` and within this directory will be our user programs that will use the stdlib that we write. 

## Writing the first user program

Next, in `src/bin/` directory create a new files named whatever you want. For example `init.rs`. In rust, all the files in the `bin/` directory are compiled into separate individual output binaries. And you guessed it, they will be able to use our `lib.rs` as the stdlib. For now let us just focus on compiling a user program and getting it loaded and executing in the RPi memory. We will look into writing stdlib some other time. Write your `init.rs` as another baremetal code similar to our kernel's `main.rs`.

```rust
#![no_std]
#![no_main]

use user::println;

#[unsafe(no_mangle)]
pub extern "C" fn _start() -> ! {
    println!("hello this code is running in the init program!").unwrap();

    let mut x = 1;
    println!("x = {}", x).unwrap();
    x += 1;
    println!("x = {}", x).unwrap();
    
    println!("init program is done working, it will now loop forever.").unwrap();
    loop {}
}

use core::panic::PanicInfo;
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    println!("Some error occurred in the init program!").unwrap();
    loop {}
}
```

Now, you will notice that I am using a `println!` macro in this code. This is because ultimately when this code is compiled and loaded and running on the machine, it's not like we ourselves will have a way to confirm that it is running if it is not outputing something somewhere. So for that, just like how we created our own print functionality in `kernel::peripherals`, we will do similarly for this user crate. 

Formally, that print functionality in this user code is going to be completely different from the kind written in `kernel::peripherals`. Remember, ultimately we are going to take away the user program's ability to freely touch the UART MMIO addresses. How will the user programs (with our stdlib) write to the output if they cannot actually access the peripherals? The answer is that instead of letting the user programs do that, we'll write our `println!` functionality in stdlib in such a way, so that instead of directly writing to UART like the kernel, it simply takes the string to print and passes it to the kernel through something called a "trap exception". And then the kernel will print the string for the user program, and then handle the control back to the user program.

This concept is called "syscalls". In this scenario we say that the user program will make a syscall to the kernel; requesting the kernel to do a priviledged move (e.g. writing data to hardware MMIO for printing to UART). We will formally introduce and learn how to implement this procedure in next chapters.

For now, our main focus is first getting the user process in the memory and running. Since currently we have done nothing related to securing the MMIO addresses from the user process, by default currently they can access the entire hardware peripherals. So for the sake of testing we can just copy the contents of the entire `kernel/peripherals.rs` to our `lib.rs`. Make sure you rename the references in the macro to occur in `user::` instead of `$crate::kernel`.

Now anything in lib.rs can be imported/included in our `bin/` codes using the `user` crate name. Which in our scenario we do `use user::println;`

### Linker

Now as for the linker, we need to tell the compiler where we are going to load the program for execution. On the raspberry pi 3b+ you can pick any address between your kernel ending address to roughly `0x3EFFFFFF`. For safety we will just pick some address away from our kernel at `0x200_000`. That is where I'm choosing to put our process for now. You can choose any safe address at this stage, however ultimately it doesn't matter when we get to implementing MMU.

Here is what the linker script looks like 

```
ENTRY(_start)

SECTIONS
{
    /* once memory virtualization works, i will change this to start at 0x00000000 */
    . = 0x200000;

    .text : {
        *(.text*)
    }

    .rodata : {
        *(.rodata*)
    }

    .data : {
        *(.data*)
    }

    .bss : {
        *(.bss*)
    }
}
```

### Compilation

now all you need to do is to compile your code as usual. 

```bash
$ cargo build
```

And then to convert the compiled binary to an image we can load:

```bash
$ aarch64-linux-gnu-objcopy \
target/aarch64-unknown-none/debug/init \
-O binary init.bin
```

Now you have an init.bin that your RPi can execute. 

However you also need to find out the entry point of this image, since we did not explicitly mention the entry point address in the linker script. To find out where it is, you can use the `readelf` command to read from the compiled binary.

```bash
$ readelf -h target/aarch64-unknown-none/debug/init 
```

This will read the ELF file's header information and provide you some information. Look for something like:

```
(...)
  Entry point address:               0x2002f8
(...)
```

That is the address of the first instruction meant to be executed in our `init.bin` image. 

## Running the first user program

Now, you have the image and you know where to start the execution of the image. All you need to do now is include the `init.bin` image data in your `kernel8.img` and make it copy it to the correct address `0x200_000` in memory. Then, to start execution of it from the entry point address.

All of this has to be performed as you do a drop to EL0. Since user programs normally are supposed to run in EL0. 

First, to include the user program bytes in our kernel we can simply use the macro `include_bytes!("path/to/file")`. This macro, during compilation, reads the given file in the argument and expands the macro into a `&'static [u8; N]` where N is the number of bytes in the given file. So basically if you write `let image = include_bytes!("user.bin")` it is the equivalent of writing `let image: &[u8] = &[0x4A, 0X24, 0X23, ...];` where the right side of `=` is literally all the bytes from the `user.bin` file in the correct order.

And then of course, the user program will refer to the stack pointer whenever it wants to allocate memory for any variables. And since we plan on dropping to EL0 before execution of the user program, we will need to put the stack pointer in `SP_EL0` register, since that is what programs running in EL0 use for stack pointer.

All of this can be implemented as follows: 

```rust
fn load_and_run_init_process() {
    const INIT_PROCESS_IMAGE: &[u8] = include_bytes!("user/init.bin");
    const INIT_PROCESS_ADDR: usize = 0x200000;
    unsafe {
        core::ptr::copy_nonoverlapping(
            INIT_PROCESS_IMAGE.as_ptr(),
            INIT_PROCESS_ADDR as *mut u8,
            INIT_PROCESS_IMAGE.len(),
        );
    }

    // i got this entry point from the compiled init elf by doing `readelf -h init`.
    const ENTRY_POINT: usize = 0x2002f8;
    const STACK_TOP: usize = (INIT_PROCESS_ADDR + INIT_PROCESS_IMAGE.len() + 0x4000) & !0xf; // 16 byte aligned stack top 
    // right now i have just hardcoded some stack pointer for EL0

    enter_user(ENTRY_POINT, STACK_TOP);
}
```

And then the entry into the program can be done the same way we dropped from EL2 into EL1 at `main.rs`'s `main()`, as following:

```rust
fn enter_user(entry_point: usize, stack_top: usize) {
    unsafe {
        core::arch::asm!(
            "
            msr sp_el0, {stack}
            msr elr_el1, {entry}
    
            mov x0, xzr
            msr spsr_el1, x0
    
            eret
            ",
            stack = in(reg) stack_top,
            entry = in(reg) entry_point,
            options(noreturn)
        );   
    }
}
```

As you can see, we load `SP_EL0` to stack pointer and `ELR_EL1` register to the entry point of the user program. We then set `SPSR_EL1` to zero, which signifies that after `ERET` we drop to EL0. This all is ultimately set in stone by the final `eret` at the end. Thus, after this `ERET` the execution will go to the loaded user program in memory at the first instruction's byte. At EL0. And execution will continue on from there. 

From here all that is left to do is to call it.

Remove the exception handling testing related lines of code in `main()` if you still have them from the last chapter. And call the function in place of it. 

```rust
load_and_run_init_process();
```

Now if you `make` and then `make run` you will see output as follows:

```bash
$ qemu-system-aarch64 -M raspi3b -kernel kernel8.img -serial null -serial stdio

Welcome to, AtOS.
Current EL is: EL1
hello this code is running in the init program!
x = 1
x = 2
init program is done working, it will now loop forever.
```

## Postface

Now, just like that you have a user program separate from your kernel project. Compiled separately and loaded by your kernel into memory. Running in EL0 mode as it should. However, it is still far fetched to call it a "Process". Since currently our OS cannot track, control or facilitate the user program in anyway. Once it is loaded into memory, it is just going to run the same way our kernel was running. We are going to change this in future chapters. First we will write a new `println!` functionality for our user space which will be a proper syscall instead of our current bodge. And then we will create a way for the kernel to routinely get control back from the running user program. Letting it inspect if everything is okay or run for a while doing whatever it wants, before it chooses to give back control to a running user program.

## Final codes

`main.rs`
```rs
#![no_std]
#![no_main]

mod kernel;

use core::panic::PanicInfo;

#[no_mangle]
pub extern "C" fn main() -> ! {
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

`user/bin/init.rs`
```rs
#![no_std]
#![no_main]

use user::println;

#[unsafe(no_mangle)]
pub extern "C" fn _start() -> ! {
    println!("hello this code is running in the init program!").unwrap();

    let mut x = 1;
    println!("x = {}", x).unwrap();
    x += 1;
    println!("x = {}", x).unwrap();
    
    println!("init program is done working, it will now loop forever.").unwrap();
    loop {}
}

use core::panic::PanicInfo;
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    println!("Some error occurred in the init program!").unwrap();
    loop {}
}
```
