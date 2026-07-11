# Chapter 6: Process Abstraction

We already have a basic user space setup with a program running in user space in EL0, separated from the kerrnel space. Utilizing syscalls and taking input and output. However, if you've been observant you may have noticed that your device, like your computer or your mobile phone usually have more than one processes going on. How is this possible? Even if you take advantage of other CPU cores on the device, you would at most get 8 or 12 other processes executing simultaneously on an average high end device. How do operating systems handle hundreds of processes running at the same time? 

The answer is that they don't have hundred of processing running simultaneously. It is actually a very clever Sleight of hand. Let's understand this with an example where the CPU is trying to do this illusion with two processes. 

## A Tiny Introduction On Scheduling

What the operating system actually does, is that it lets one user process run for a while, for maybe a few milliseconds for example. And once that duration is up, it saves the state of the CPU registers to memory of the program, and jumps PC to the instructions of a completely different user process. It lets that process then run for a few milliseconds again. And again, once that duration is also up, the CPU register values for this process are saved and the previously saved values of the previous process are loaded up. And then the CPU jumps the PC back to the location where the original process instructions were left off. And this loop continues infinitely as long as both the processes are running.

This way, both processes take turns running on the CPU. And since the CPU is usually *super* fast in running these processes, the naked eye is never able to tell that the processes are actually taking turns running. To the naked eye it seems as though both processes are running at the same time.

The procedure of saving the CPU register values to preserve the process's context for instruction execution, and loading the register values of the next process, is called "context switching".

For more than two processes, this procedure can be extended to use some algorithm to decide which process to pick next for the CPU turn. This act of deciding which process gets to go next on the CPU, and the duration it gets on the CPU, is called "scheduling". The component that manages the process's scheduling in the kernel is called the "scheduler". 

We are going to focus on building our scheduler in the next chapters, however before we can get to that we need to implement a way for the scheduler to keep track of the different processes and their CPU context. Stuff like where the process instructions were left off when its turn on the CPU ended, if that particular process is allowed to be scheduled onto the CPU or not, etc.

And so in this chapter we will learn and implement the structure and features that are needed by the kernel to keep track of these running user programs. 

## Processes

Recall from our previous chapters that a user program is called a process when it is running and actively kept track of by the CPU. Now, on your system for example, the apps and games that you have on your hard disk/ssd/memory card are called "programs". However, when you try to open these apps on your system, the operating system copies their data to the memory, and then creates an environment in the memory for the instructions to run and utilize the hardware. That data in memory which represents the program is called a 'process'. We've already gone over this in previous chapters.

## Loading a Process

You should know by now that a process is basically a program loaded into memory. Usually, programs are saved on the disk in certain formats which contain useful information, which tells you what data is supposed to be put where in memory. The most command standard of these formats is the EXE format on windows, or ELF format on linux. Of course programs compiled for your operating system have absolutely no way of knowing what part of your memory might be free for them to be expected to loaded in. This is where virtual memory comes in. However, for now we will work without any virtual memory or virtualization. For now we will continue to work with processes that already know where they can expect to be in RAM, and are linked accordingly. 

The actual procedure of loading a process to memory will remain the same as before for now in fact. What changes here is that we are going to implement new structs and enums to track and index our created processes.

There are many things which are kept track of for each unique process. The state of the process, when it was last scheduled, the files it currently has open, the children it has, etc. However, many of those features are not needed to be implemented all at once. For now, we only really need three things.
  
1. Tracking/Storing the Process Context: all the CPU register values that the scheduler needs to load before scheduling the process again.

2. Current state of the Process: There can be many states, but for now we only need to worry about whether the process is currently running on the cpu, whether it is waiting to be scheduled, if it is waiting for something other than the scheduler (e.g. user input), and if it has been terminated for whatever reason (processes obviously have to close and end eventually).

3. Identity of the Process: Mainly just the ID of the process and the ID of who spawned the process. That process is usually called the parent process, and the process which was spawned by it is called the child process. We can also store the name of the process for the user to identify them.

## Implementation

Now we can start actually implementing it. 

Firstly, the process state. We just went over the four main states we will cover right now. But in formal lingo they are called:

- Ready State: Process is ready to be scheduled
- Running State: This process is currently scheduled
- Waiting/Blocked State: This process is waiting for something to happen on the hardware (e.g. user input, disk read, network data receival, etc).
- Terminated: This process is finished working or has been forcefully stopped.

These can be implemented as an enum.

So let's go ahead and create `src/kernel/processes.rs` and start it off with:

```rs
#[repr(u8)]
#[derive(Copy, Clone, Debug, PartialEq, Eq)]
#[allow(unused)]
pub enum ProcessState {
    Ready,
    Running,
    Blocked,
    Terminated,
} 
```

Next, for the process context, we need to create a struct which will store all the CPU registers relevant to loading to a different process's instructions.

```rs
#[repr(C)]
#[derive(Debug, Clone, Copy, Default)]
pub struct ProcessContext {
    pub x: [u64; 31], 
    pub sp: u64, 
    pub elr: u64, // address to jump to when jumping to this process
    pub spsr: u64,
}
``` 

as you can see, we account for the GPRs, the stack pointer, and then `ELR_EL1` and `SPSR_EL1`. The last two are for the entry point address location, and the process hardware state register. We have gone over these before in chapter 3 on exceptions.

Now, obviously the `ELR` registers here represents the instruction that the process may have been left off at last time before its turn on CPU was up. The `SP` register contains the stack pointer for the user process's stack. But what about when a new process is created?

For a new process, the entry point value for `ELR` would be the entry point of the process itself. That is, the address to the first instruction meant to be run in the process. Which of course is only known while loading the process. The stack pointer is also setup by whatever function that loads the process. So we're going to add a constructor function that gives a fresh `ProcessContext` for a brand new created process.

```rs
impl ProcessContext {
    pub fn new(entry_point: u64, sp: u64) -> Self {
        Self { elr: entry_point, sp, ..Default::default() }
    }
}
```

Notice tat our `ProcessContext` implements `Defaut` trait, it makes it possible to initialize all other values to zero without needing to type it out.

Now we can finally create our actual `Process` struct itself

```rs
#[repr(C)]
#[derive(Debug, Clone, Copy)]
pub struct Process {
    // identity
    pub pid: u64,
    pub name: [u8; 32],
    pub state: ProcessState,
    pub parent_pid: u64,
    pub pctx: ProcessContext,
}
```

That would be our struct which will represent processes. Notice how we don't actually have any way to track the memory used by the process yet. How do you know what memory to mark is available once the process terminates? You could of course simply track the starting and ending address of the process's memory. However, that entire system will become useless once we implement paging during the virtualization chapters later on. So for now we are going to skip adding any such thing. Our main focus is implementing enough so that the scheduler may be able to work.

Now, of course we're doing all of this because ultimately we are going to have different processes running on our OS. We need to track all of them. So we should change our `load_and_run_init_process` function to instead load and run any image provided to it. That it may be able to take some general arguments such as the process image, name, process starting address, stack pointer location, entry point address, etc. So that it may then be able to load that image and data into the correct memory location and create a valid `Process` struct for it.

But before we do that we need to implement some more things. The `Process struct needs a constructor of course. The function which loads the process could technically setup all the members of `Process` manually. However it is better that stuff like process ID be generated automatically, without the process loader to need to keep track of the next available ID. Also, we've defined the `Process::name` to be a `[u8; 32]`, i.e. array of 32 bytes. It would be convenient if there was also a system that would automatically convert strings to this format so we can simply pass in names of processes as `&str`.

This can be implemented as follows:

```rs
pub static mut NEXT_PID: u64 = 1; // 0 could be for kernel

impl Process {
    pub fn new(name: &str, parent_pid: u64, entry_point: u64, sp: u64) -> Self {
        let pid = unsafe {
            let pid = NEXT_PID;
            NEXT_PID += 1;
            pid
        };

        let mut name_bytes = [0u8; 32];
        let bytes = name.as_bytes();

        let len = core::cmp::min(bytes.len(), 32);
        name_bytes[..len].copy_from_slice(&bytes[..len]);

        Self { pid,
               name: name_bytes,
               state: ProcessState::Ready, 
               parent_pid,
               pctx: ProcessContext::new(entry_point, sp) }
    }
}
```

Lastly, you may have already wondered about this, but we do in fact need a location to store the `Process` object itself. Even if the process loader creates an appropriate `Process` object, it is still useless if we don't save it to some sort of array or table for the kernel to be able to find it and read from it later.

For that we will create something called a PROCESS TABLE. It is basically going to be an array of `Process`es. To be exact, it's data type will be `[Option<Process>; MAX_PROCESSES]`. Where `MAX_PROCESSES` is some hardcoded constant that dictates the size of this table. We choose `Option<Process>` as the entry type. This is so by default the entry can be `None`, depicting emptiness of the slot. 

```rs
pub const MAX_PROCESSES: usize = 50;
pub static mut PROCESS_TABLE: [Option<Process>; MAX_PROCESSES] = [None; MAX_PROCESSES]; 
```

Now let's create a function that will add a proess to our process table, easing the procedure.

```rs
fn add_process_to_ptable(process: Process) -> Result<(), &'static str> {
    unsafe {
        #[allow(static_mut_refs)]
        for slot in PROCESS_TABLE.iter_mut() {
            if slot.is_none() {
                *slot = Some(process);
                return Ok(());
            }
        }
    }
    Err("Process table is full")
}
```

Note that the part where we create a mutable iterator out of the process table, is actually not legal in rust by default. Rust will complain at you that you're not allowed to do this. But since we're working in the baremetal context, we have to get our hands dirty with stuff like this, so you may add the following at the top of your module:

```rs
#![allow(static_mut_refs)]
```

Now we can finally implement our actual `load_process` function as follows

```rs
pub fn load_process(process_name: &str, parent_pid: u64, process_image: &'static [u8], process_addr: u64, entry_point: u64) {
    unsafe {
        core::ptr::copy_nonoverlapping(
            process_image.as_ptr(),
            process_addr as *mut u8,
            process_image.len(),
        );
    }

    let stack_top: u64 = (process_addr + process_image.len() as u64 + 0x4000) & !0xf; // 16 byte aligned stack top 
    // right now i have just hardcoded some stack pointer for EL0

    let process: Process = Process::new(process_name, parent_pid, entry_point, stack_top);

    let _ = add_process_to_ptable(process);

    // enter_user(entry_point, stack_top); // obsolete
}
```

Notice how the `enter_user` function has been commented out and marked as obsolete! This is because the process loading function will no longer be the one responsible for actually starting the execution of the process. All the process loading function will do is copy the process data to the correct place in memory and add appropriate `Process` object to the process table. It will be the job of the **scheduler** who will go through the entire process table and run them on the CPU in a planned order. This is something we will implement in the next chapters.

For now you could manually cause a jump using `enter_user` to verify that the user program is infact executing as a process. It should give you the same output as before.

Of course, don't forget to actually call this `load_process` function properly in `main()`:

```rs
    let process_a_image: &'static [u8] = include_bytes!("user/init.bin");
    
    load_process("init", 0, process_a_image, 0x200000, 0x2002f8);
```

## Final code

The next snapshot of source code will be provided at the end of chapter 8: scheduling.

However you can still find the code mentioned in this chapter at the following commit tree:

[github.com/ZackyGameDev/AtOS/blob/ae(...)rnel/processes.rs](https://github.com/ZackyGameDev/AtOS/blob/aef3fa0c404b7423a9ef92770c3f12e91604708e/src/kernel/processes.rs)

(Please ignore the additional functions.)