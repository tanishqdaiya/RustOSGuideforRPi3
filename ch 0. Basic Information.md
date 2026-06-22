# Basic Information

## Introduction

Creating an Operating System from scratch is widely considered to be one of the most challenging tasks in computer science. It involves getting to know your machine at a hardware chip level, and getting acquianted with features and quirks that you never would have to worry about at software level. However it is still an attractive challenge because creating an Operating System can reward you with knowledge and insights of Computer Science at a personal level. Regardless whether you're doing this out of passion, out of need, or just because you're curious about how this all works. The point of this guide is to walk you through all the concepts involved from a beginner stage. This chapter will involve all the knowledge you need to have before you can write your first line of code.

## Where I come from

I am not a professional Operating Systems Engineer, nor am I a professor with lots of knowledge in this field. I just happen to be working on my own OS and I think I could do a helpful job of documenting the entire process. Naturally this means if I learn new things, I may edit chapters to add things to them or fix them. I can't promise that writing this guide will remain my top priority. However, I intend to fully follow through with it alongside my OS development.

## Targetted Hardware

This guide will specifically tailor to the Raspberry Pi 3 board. I myself used the RPi3b+ for my project, but everything should work for all interations of RPi3. From the start, the primary way of receiving output from our code running on the RPi will be through something called "UART" (discussed later on), which will require a TTL(female) to USB(male) cable. And of course I'll assume you have a power supply for your RPi as well.

## Project boundaries

Like I mentioned, the purpose of this entire documentation is strictly RPi3 OS development, that means it's possible that things you learn through this document may not necessarily apply on other hardwares. Any description of hardware behaviour should be assumed to be written with RPi3 in mind specifically. Though, RPi3 is not an entirely strange or alien machine, it also follows most standards that many other computers do, so knowledge is not exactly wasted if your ultimate target is a different hardware.

That wraps up the personal statements.

## Fetch-execution cycle

When writing something that will run directly on the computer chip, without any underlying operating system, it is the most foundational step that you know exactly how your code actually looks like while it is being run by the computer chip. 

### A bit about working memory

You must know what "RAM" is, its a piece of memory that your computer uses for random access of data and code. The CPU chip will always run code directly from the RAM. The way it works is that RAM, to the computer, is pretty much just a ridiculously long array of bytes. Where all the bytes are indexed from zero to... wherever the last byte is. Since there's so many of these, we will always use _hexadecimal_ notation when referring to the index of any of these bytes. (To signal that the number written is supposed to be hex notation, it will be prefixed by `0x` which is also the standard used in most programming languages.)

The index of these bytes is called the "address" of this byte, and the byte itself, is called the "data".

So if somebody asks "what is the data at address `0x5fff`" they're really just asking for the byte with the index "`0x5fff`" (which is just 24575 in decimal)

Note: whenever this guide mentions "memory" without additional context, assume that it is referring to the RAM. The ROM, like your SSDs or HDDs are usually called "disk". In our context, our "disk" is going to be the microSD card that we put our operating system data on before plugging it into the pi.

### What does your code actually look like?

You need to know that the CPU chip itself is basically just a really, ridiculously complicated electrical circuit. It isn't really "intelligent" and it doesn't exactly "know" things. It simply reads data from addresses, and depending on what that data is, some electrical circuit behaviour is triggered inside it, and then optionally it might also write data back to some memory address. 

Now, as for your code. When you compile your code, it basically gets converted into something called "Machine language". Which is a format that the CPU can actually understand. Something that if the CPU reads bytes from, it will trigger the correct electronic circuitries inside it for it to produce the behaviour that the code wanted.

Machine code is also just a series/array of bytes. The files that you run can have many sections in them, but in all of them there is a section for such bytes which do not represent any data, instead, they are supposed to be read by the CPU to trigger the correct behaviours from it. These bytes are called _"Instructions"_

Now, I grossly oversimplified it, it is not so simple that if your instructions section has 10 bytes in an array, every byte individually represents a single instructions. It really does depend on which hardware you are talking about. But in the RPi3, every four bytes represent a whole instruction. so in memory, the code is stored as an array of bytes. And picking four bytes at a time, all of them are an instruction individually. You don't really need to know the details of how to figure out from four bytes, which instruction it is. That's the job of the CPU chip. 

### Fetch-execute cycle

Now, when the Raspberry Pi 3 starts, what it does is that it loads some data you give it through its memory card. That is, it literally copies the bytes from the memory card, directly to the RAM, in the exact order written on the card. Which sections from the microSD card are copied to what section of the memory will be decided and prescribed to the RPi by you (at least, after you learn how to; by reading further chapters). It then tries to read data at the address `0x80000`. It assumes that the four bytes starting from `0x80000` represent an instruction. And then the CPU starts its job, it reads those four bytes from the memory at `0x80000`, `0x80001`, `0x80002`, `0x80003`, and then accordingly its internal circuitary behaves; the way it is designed to behave upon seeing that instruction (i.e. that exact order of four bytes). This is called "executing the instruciton". If that behaviour involves writing some data to the memory, it does that, and then moves on to the next four bytes starting from `0x80004`. Reading them, letting the circuitary inside it interpret them and behave to them, i.e. executing them, and then again moves on to next four bytes. This cycle goes on infinitely (ideally).

This is called the "Fetch execute cycle". Because the CPU is in a never ending labour of fetching an instruction, executing it, and then moving on to next instruction in memory to do it all over again.

Do note, there are some instructions that make the CPU go to a different location in memory for the next instruction, instead of the instruction right next to the current one. These are called "Jumping" or "Branching" instructions. More on them will be discussed as needed. 

Another note, the fact that the RPi3 starts executing instructions from 0x80000 instruction is completely fixed. It is just the way its processor is designed. As a systems programmer, you will encounter many design choices like these from the manufacturers of your hardware. You will just have to figure out how to deal with it and work with what there is.

## Registers x0-x30 and Stack Pointer

TODO — low priority <br>
You can probably find resources to explain this easily. <br>

But for example: <br>
https://www.geeksforgeeks.org/computer-organization-architecture/different-classes-of-cpu-registers/ <br>
https://dev.to/serputov/aarch64-x86-64-registers-and-instruction-quick-start-19bd <br>
https://en.wikipedia.org/wiki/Stack-based_memory_allocation
