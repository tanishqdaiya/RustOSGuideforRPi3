# (following is underwork)

# User space isolation

So far what we've done is that we have created a separate project directory which acts as the place where all the user's programs and standard library will be written. Basically all the stuff that is supposed to run ON our operating system. However why did we create a separate project for it? Why couldn't we just keep growing our original operating system repository? Well, there's mainly two reasons.

the first reason is that user program is not just supposed to be the processes that run for the user like the shell, window manager, desktop environment, etc. The user programs can also be third party codes or applications that are written by people who are not associated with us or our operating system's development. They should have a way to write applications that can run on our operating system without having to know how the OS itself handles the hardware.

(to be continued)