# Chapter 1: Baremetal Rust

Now that you know exactly how code is actually run and what it looks like after compilation, we have to actually make it run. We are trying to write an Operating System in rust. So of course we need to figure out how to make program written in rust run directly on our RPi without any underlying operating systems.

## Project init

<small> NOTE: This section is for Linux (where you'll write your rust code and compile it for the Pi). If you're using a different OS some things may or may not be different. Feel free to take help from LLMs or other online resources for setting up your baremetal rust project with aarch64-unknown-none as your target.</small>

Let's first initialize our bare bones project. 
But before we do that we need to run:

```bash
rustup target add aarch64-unknown-none
```

This will install and add whatever Rust compiler needs to be able to compile code to instructions that can run on "AArch64" architecture (basically RPi3's CPU architecture). The "unknown" and "none" refer to the "vendor" and "underlying operating system" of your target respectively.

Now you can initialize your project with `cargo`. (comes with rust)

```bash
cargo new atos --bin
cd atos
```

Now in your project folder there must be a file named "Cargo.toml". You can add some basic information about your project there. For me it was:

```toml
[package]
name = "at-os"
version = "0.1.0"
authors = ["ZackyGameDev <zaacky2456@gmail.com>"]
```

Next, we have to tell our project what our rust code is meant to be compiled for. Create a file `.cargo/config.toml` which will have information about our compiling settings. and put the following in it:

```toml
[build]
target = "aarch64-unknown-none"
```

Optionally, if you're using vscode then tell the linter/syntax checker what you're targetting by putting the following in `.vscode/settings.json`:

```json
{
    "rust-analyzer.check.allTargets": false,
    "rust-analyzer.cargo.target": "aarch64-unknown-none"
}
```

Now there's still some more we have to setup before we can actually build it, but before that we have to understand some more things and write a little bit of code.

## Lack of Underlying OS

In rust, and in pretty much any programming language actually, there are many features that rely on Operating Systems beneath them, for them to work. For example in C, when you want to open a file, the instructions from your compiled C code itself don't really read the disk directly trying to decipher the data on the disk to find your file and start writing to it. Instead, your code will ask the Operating System beneath to do it for you, and just give you something like a pointer to the file. This is usually so programs written don't need to access the memory directly (becaues if they're evil or terrible they might try to mess up with parts of memory you wouldn't want them to mess up). 

If you've read up on this, you'd know this way of asking the operating system for something is called "System Calls". Where essentially the programming language (in user space) is relying on the kernel (in kernel space) to provide low level interactions through system calls. 
(read about it at [OSTEP: Ch6](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-mechanisms.pdf))

Well what about when you're running your code directly on the computer without any operating system beneath? What if there's no "kernel" to do system calls to? It's simple. Those features become unusable and pretty much useless. And in rust, "those features" includes the entire Rust Standard Library called `std`.

## Lack of `crt0`

Rust code, normally on windows or linux just running normally on the same machine, doesn't run directly. What happens under the hood is that first, something called the "C Runtime Library" is run. What it does is basically setup the environment and parameters of the hardware, in preparation for Rust code to run. It first does something called "setting up the stack" and "initializing .data" or "zeroing the .bss" (we'll learn what it is later). And then it literally calls the function named "main" in your rust code. (Which is why the first lines of code to run are written in a function called "main()", because that's the name `crt0` calls).

As you may have guessed, when making our rust on baremetal, there is no such thing. The labour of "setting up stack" or preparing environment for rust is also going to be need to be done by you manually, before your rust code even starts. But wait... if we have to write some code that does some stuff before Rust code can run... that means that preparation code can't be written in Rust... The truth is unavoidably, we'll have to write this preparatory code in assembly. Because it's pretty much the only language that can run immediately without any preparation (because you're literally writing the structure of the machine code directly). Although you can try to minimize it as much as possible, in this entire project you'll have to unavoidably write some bits in assembly 

## Assembly entry point and preparing for Rust main

Time to finally write those first ever lines of code that will run at `0x80000` (I hope you know the relevance of this address by now).

You can put your file anywhere, for me I put it at root of the project, `./entry.S`. This is what it looks like:

```asm
.section ".text.boot"
.global _start

_start:
    // set stack pointer
    ldr   x0, =_stack_top
    mov   sp, x0

    // jump to rust
    bl   main

1:
    wfe
    b 1b
```

If you've never used assembly before, this might look scary, but it's pretty simple. Let's understand this line by line. The first line defines the section this code will go to (discussed under linking further below). Then `.global _start` just means "this symbol called `_start` should be accessible globally, to all files and codes". This is just for the linking discussed further below.

Next, the first lines of code (labeled under `_start`). All we're doing is loading up an address `_stack_top` in the `x0` register of the chip, and then copying that value to the `sp` register from `x0` register. We load it into `x0` first because there's no instruction to copy an address directly to `sp` register. 

(If this seems alien, refer to chapter 0 about registers and stack)

`ldr`, `mov` and `bl` are all instructions. There's probably uncountable instructions on the ARM chip. But you can google for them as you need them. Writing assembly literally just involves listing all the instructions that need to be run line by line. Sometimes those instructions might need some more specification like "which register is this instruction going to copy value to?" or "what value is this instruction dealing with?". Those are written after the instruction name, separated by commas. every line of instruction is just "INSTRUCTION ARG0, ARG1". 

And then finally we do `bl main` which literally just means "call the code under the label `main`". Where is main you ask? Well.. that's the main function in our rust code of course! So what this is essentially doing is calling the rust main function after setting the `sp` register to some value.

You might've noticed we just pulled this variable called `_stack_top` and just assumed it has the correct address to set the stack to. But where was this defined? Where did it come from? Well again, it is defined in the linking process. Assembly doesn't really have variables. It's not like there's some variable defined on the stack or heap which the assembly accesses like other programming langauges. If it worked like that we would've written the instructions to access data from the heap and use it. This is actually something called a "symbol". kind of like inline macros from C. The linker while producing the final output file will see this symbol, and try to find where that symbol is defined. And then everywhere in code it will replace the symbol with its correct defined value.

In any case, as it turns out setting `sp` register to the correct value happens to be pretty much all you need to do to call your `main` funciton in rust and expect it to work correctly. So that's what we do. The way the `bl` instruction works is that once the rust `main` function finishes executing, it's going to come back to this assembly code and resume executing the instructions written after the `bl` instruction. But Operating Systems don't really finish executing. They continuously in an infinite loop manage the hardware, respond to user, and facilitate services for softwares running on it. So that's what the last few lines are for. It's pretty much just an infinite loop which tells it to wait for a bit `wfe`, and then jump back to label "1". Hopefully these instructions will never be executed because our rust main function will never return. But it is still there as a fail safe.

## Rust main.rs

Finally, we can write some Rust code.

There must be a file `src/main.rs` in your project created automatically. It houses our main function.

Earlier we discussed that the standard library will not function in our project. Therefore we disable it by writing `#![no_std]` at the top of `main.rs`.

Also, since we do not have `crt0`, we also have to tell rust to NOT generate anything for it, and not expect the C Runtime Library to be there to run the main function automatically. For this you put `#![no_main]` at the top of your file.

Next our actual main function!

```rust
#[no_mangle]
pub extern "C" fn main() -> ! {
    loop {}
}
```

We use `extern "C"` to let it know that this function must follow C-style convention for being called. The C-style convention basically decides how can code written in assembly call this function by trying to jump to it as a label with instructions like `bl`. It is important because this is how our `bl main` line in `entry.S` will actually work. 

Also, when the compiler compiles our code, it may obfuscate or change our function's name to some random garble to make it unique and save space. But we don't want that. We want the function name to stay the same after compilation, so our `entry.S` and other files can recognize it with the symbol `main`. For that we put `#[no_mangle]` before the definition. 

as for the return type, `!` just means that this function should NEVER return. That means it will never ever finish executing (through an infinite loop).

## Panic handling

Now, if you try to do something illegal in rust, which raises an exception. Or in Rust lingo, causes your program to panic, Rust usually prints out a very helpful message to `stdout` to tell you where your code messed up. It also shows a very generous option to see a backtrace to exactly which function called which function to lead to the line at which the error happened. Sadly, all this functionality is defined in `std`. Which we have excluded. So now we are left to implement that panic handling ourselves.

It is actually very simple. You just need to write a function that accepts a `core::panic::PanicInfo` object, and just mark it as the "panic handler". Like so:

```rust
use core::panic::PanicInfo;
#[panic_handler]
fn panic(_: &PanicInfo) -> ! {
    loop {}
}
```

Whenever a panic occurs, this function will be called. And you can access panic information from the one argument. We can implement panic handling later. For now we're just trying to appease Rust compiler. So all our handler does right now is enter an infinite loop upon panic.

Now Rust will no longer complain about no panic handler being implemented.

Another thing is that when a panic happens, rust uses something called "unwinding". Which basically frees up memory in the event a panic happens. And gracefully exits out to the parent who called the panic inducing function/thread. However, unwinding uses some OS specific libraries which we obviously do not have. Therefore, we are going to disable this feature as well to avoid unpredictable behaviour.

For this just append the following `.cargo/config.toml`:

```rust
[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
```

This is going to change the behaviour on panic, from unwind to abort. So Rust will not try to do any additional behaviour upon panic, it will simply call our panic handler and pass it a `PanicInfo` object. No unwinding will be attempted.

Now this wraps up our Rust code. 

<small> Final version of all code files are available at the end of this chapter. </small>

## Linking

Finally the part where we start putting together all our project to compile, build and produce a single file to give to the RPi.

In chapter 0 I mentioned that you will be able to tell RPi, which memory addresses to put what code or what data in when booting it. The way it works is the following: When you compile your code (by using the build commands we will talk about in a bit), all the files in your code are first compiled, i.e. translated to machine code. Then something called the "Linker" combines all those machine codes into a single file. Which we give to our raspberry pi; to copy to the memory after booting. Now RPi will always just copy the entire file exactly one to one to it's memory. so data from addres 0x00000 in the file will be copied to the address 0x80000 in the memory and so on. So to decide what data goes to which memory, you have to put it at the correct place in the compiled file itself. 

Since it is the Linker who combines all the code into a single file, it also provides us the option to let it know which code can be expected to go to which addresses when the final compiled file will run. This is done through something called a "Linker script". To be exact, we're going to give a "label" to our code which we want to run immediately after RPi boots. And tell the linker to put all the code with that label at the address "0x80000". (Because as discussed last chapter, that is the hard fixed address that RPi starts executing instructions from after booting).

To be exact, when we say we tell the linker to put all the code at address 0x80000, it doesn't mean that the data will start at that address _inside_ the file. No in fact, it means that when this code is running, it can be expected to reside in the address 0x80000 in memory. So all the instruction bytes in our final file will expect other data to be found in the addresses specified by linker.

The linker, and in general our `cargo build` command will actually produce something called an "ELF" file (stands for executable and linkable format). It is basically the format of `.exe` files followed by Unix systems (including linux). You can read about them more, but mainly what you need to know is that ELF files are meant to be run, so they have an entry point, and some additional information about them in the first couple bytes called "header". But what our RPi expects is literally just a snapshot or "image" of what the RAM should look like. That file is called "kernel8.img" (this filename is fixed, RPi will look for this exact file name). RPi just loads this file as is to memory. 

So we will first use cargo with our linker script to produce an ELF file, and then convert it to a `kernel8.img` file for RPi. and that img file is what goes to the memory card.

We're going to define the linker script in the root of the directory. Though you can put it anywhere.

`linker.ld`
```ld
ENTRY(_start)

SECTIONS
{
    . = 0x80000;

    .text :
    {
        *(.text.boot)
        *(.text*)
    }

    .rodata : { *(.rodata*) }
    .data   : { *(.data*) }
    .bss    : { *(.bss*) }

    . = ALIGN(16);
    _stack_top = . + 0x4000;
}
```

Okay now we have to understand this. 

The first thing you need to tell the linker is the entry point of your file. If you remember we made our label `_start` global in `entry.S`. The entry point defined here is irrelevant to Raspberry Pi, it will just run instructions from 0x80000. But you need to define it anyway for the intermediate ELF file to be generated. 

Next, we define our sections, where we actually tell it what to put where in memory.

`.` is a pointer, Kind if like how you have a cursor in text editing programs. When you type text, it moves ahead. And you first place your cursor to the location you want to start typing from. The same way we first do, `. = 0x80000`. Which means placing the cusor at `0x80000` location to start writing from there. This address is the place execution starts from in RPi.

Then first section is `.text` which stands for "code". It's a very strange naming choice, but yes, sections of bytes meant to be executed as instructions are typically called as "text" in low level memory.

For `.text`, we can define what the section should actually look like. Since we want the subsection we wrote `.text.boot` to be at `0x80000`, it should come first. Hence, we put `.text.boot` first and then `.text*` which means all remaining other instructions. After that the rest of the sections. `.rodata` is read only data. Values of constants defined in code, or read only strings, or string literals. Then `.data` is for variables which are initialized in the code. They are writable and mutable. Finally `.bss` is data initialized to be zero in the code, also writable.

You may have noticed we did not define sections `.text`, `.data`, `.rodata` and `.bss` ourselves. They're actually default conventional names that the compiler generates. We're just telling the linker where to put them. The only thing we defined ourselves was the subsection for `.text` called `.text.boot`.

Now, remember, the `.` is a pointer, like a cursor. So now that we've put a bunch of sections, it must've moved to hold an address at the end of the last section we put. We now do `. = ALIGN(16);`. What it does is that-- if the address that the pointer is currently at, is not divisible by 16, it will increment it till it is divisible by 16. And then we finally define the symbol `_stack_top` to be the final position of the pointer (aligned to 16 divisibility), plus, `0x4000`. That means when we set `sp` to `_stack_top` in `entry.S`, it is going to have `0x4000` size of address space avaiable to it from `_stack_top - 0x4000` -> `_stack_top`.

If this seems alien or strange, please lookup what a "stack" is in memory.

Note: We align to 16 because in AArch64 standard, stack pointer is always expected to be 16 aligned whenever a function is called. That is, every stack entry address should be a multiple of 16.

Finally, we have completed all the code we had to write. We can now build it

## Building

It is recommended to first use a tool like "Raspberry pi imager" to first flash any OS in the memory card. It sets up the memory card's partitions and file systems so we just have to copy paste our image. 

Now, you have to tell your rust script that it needs to follow our `linker.ld` script. Otherwise it will ignore it. Do so by adding the following to `.cargo/config.toml`

```toml
[target.aarch64-unknown-none]
linker = "aarch64-linux-gnu-ld"
rustflags = [
  "-C", "link-arg=-Tlinker.ld",
]
```

Next, we need to setup something that will compile our assembly files. For that create a file named "`build.rs`" in root directory. By convention cargo runs this file first if it sees it. Before doing rest of the compilation. We'll tell our `build.rs` to compile our assembly files:

```rust
fn main() {
    println!("cargo:rerun-if-changed=src/entry.S");

    cc::Build::new()
        .file("src/entry.S")
        .flag("-c")
        .compile("entry");
}
```

As you can see, it needs a dependency called "CC" which stands for "cargo C". It's a dependency that helps compile C files or Assembly files for your project. Add the following to your `cargo.toml`.

```toml
[dependencies]

[build-dependencies]
cc = "1.0"
```

Finally, the moment of truth, use the following command to get the ELF file compiled:

```bash
cargo build --release
```
This will give you an output at `target/aarch64-unknown-none/release/<your_binary_name>`.

Now to convert this ELF file to raw binary image for the RPi, we use:
```bash
aarch64-linux-gnu-objcopy \
target/aarch64-unknown-none/release/<your_binary_name>\
-O binary kernel8.img
```

Where <your_binary_name> depends on your create name by default.

Now you have a `kernel8.img` that you can just copy to your RPi!

Create one more additional file named `config.txt` and put this:
```
arm_64bit=1
enable_uart=1
```

Finally, after you've used the `Raspberry Pi Imager` tool to flash any random OS to your memory card, you can just open the files in your memory card, open the bootfs partition and copy paste the files "config.txt" and "kernel8.img" from your project directory to this bootfs partition. 

Any time you make a change to your code, just do `cargo clean` before running the two commands again. And you just have to overwrite the old kernel8.img with the new one everytime. And just plug it to the Pi and power it up!


## Final codes
`main.rs`
```rust
#![no_std]
#![no_main]

use core::panic::PanicInfo;

#[no_mangle]
pub extern "C" fn main() -> ! {
    loop {}
}

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

`.cargo/config.toml`
```toml
[build]
target = "aarch64-unknown-none"

[target.aarch64-unknown-none]
linker = "aarch64-linux-gnu-ld"
rustflags = [
  "-C", "link-arg=-Tlinker.ld",
]

[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
```

The rest of the files you can find an identical version at
[github.com/ZackyGameDev/AtOS/tree/81fcfb3cbdbd0fb0add78b5c89ff8e5bad70d260](https://github.com/ZackyGameDev/AtOS/tree/81fcfb3cbdbd0fb0add78b5c89ff8e5bad70d260)
(Do note, the naming may differ, also main.rs is different at this commit, please ignore it.)

## Other readings
[os.phil-opp.com/freestanding-rust-binary/](https://os.phil-opp.com/freestanding-rust-binary/)
Excellent blog, though it is targetted for a different hardware, it is still an amazing read that covers a lot more.
