# Scheduling
How do you make multiple processes appear to run at the same time? Today, this
question has two answers:

- With the availability of multi-core processors, or on system with multiple
  processors, two processes actually can run at the same time by running two
  processes on different cores or processors
- Fake it. That is, switch rapidly (faster than a human can experience) between
  the processes. At any given moment, there is only process executing, but the
  rapid switching gives the impression that they are running "at the same
  time".

Since the operating system created in this book does not support multi-core
processors or multiple processors, our only option is to fake it. The part of
the operating system responsible for rapidly switching between the processes is
called the _scheduler_.

## Creating new processes
Creating new processes is usually done with two different system calls: `fork`
and `exec`. `fork` creates an exact copy of the currently running process,
while `exec` replaces the current process with another, often specified by a
path to the programs location in the file system. Of these two, we recommend
that you start implementing `exec`, since this system call will do almost
exactly the same steps as described in ["Setting up for user
mode"](#setting-up-for-user-mode).

## Cooperative scheduling with yielding
The easiest way to achieve the rapid switching between processes is if the
processes themselves are responsible for the switching. That is, the processes
runs for a while and then tells the OS (via a syscall) that it can now switch
to another process. Giving up the control of CPU to another process is called
_yielding_ and when the processes themselves are responsible for the
scheduling, it's called _cooperative scheduling_, since all the processes must
cooperate with each other.

When a process yields, the process entire state must be saved (all the
registers), preferably in the kernel heap in a structure that represents a
process. When changing to a new process, all the registers must be updated with
the saved values.

In order to implement scheduling, you must keep a list of which processes are
running. The system call "yield" should then run the next process in the list
and puts the current one last (other schemes are possible, but this is a simple
one).
The transfer of control to the new process is done via `iret` in exactly the
same way as explained in the section
["Entering user mode"](#entering-user-mode).

We __strongly__ recommend that you start to implement support for multiple
processes by implementing cooperative scheduling. We further recommend that you
have a working solution for both `exec`, `fork` and `yield` before implementing
preemptive scheduling. The reason for this is because it is much easier to
debug cooperative scheduling, since it is deterministic.

## Preemptive scheduling with the timer
Instead of letting the processes themselves manage when it is time to change to
another process, the OS can switch processes automatically after a short period
of time. The OS can set up the programmable interval timer (PIT) to raise an
interrupt after a short period of time, for example 20 ms. In the interrupt
handler for the PIT interrupt, the OS will change the running processes to a
new one. This way, the processes themselves don't need to think about
scheduling.

### Programmable interval timer
To be able to do preemptive scheduling, the programmable interval timer (PIT)
must be first be configured to raise interrupts every _x_ milliseconds, where
_x_ should be configurable.

The configuration of the PIT is very similar to the configuration of other
hardware devices, you send a byte to an I/O port. The command port of the PIT
is located at `0x43`. To read about all the configuration options, see
[@osdev:pit]. We use the following options:

- Raise interrupts (use channel 0)
- Send the divider as low byte then high byte (see next section for an
  explanation)
- Use a square wave
- Use binary mode

This results in the configuration byte `00110110`.

Setting the interval for how often interrupts are to be raised is done via a
_divider_. Instead of sending the PIT a value (e.g. in milliseconds) that says
how often an interrupt should be raised, you send a _divider_. The PIT operates
at 1193182 Hz as default. To change this, you send a divider, for example 10.
The PIT will then run at `1193182 / 10 = 1193182` Hz. The divider can only be
16 bits, so it only possible to configure the timers frequency between 1193182
Hz and `1193182 / 65535 = 18.2` Hz. We recommend that you create a function
that takes an interval in milliseconds and converts it to the correct divider.

The divider is then sent to channel 0 data I/O port of the PIT, but since you
can only send 1 byte at a time, you must first send the lowest 8 bits, then the
highest 8 bits of the divider. The channel 0 data I/O port is at `0x40`.

### Separate kernel stacks for processes
When the yield system calls is used, a privilege level change will occur, since
the processes runs in user mode and the process switching will run in kernel
mode. If all processes uses the same kernel stack (the stack exposed by the
TSS), there will be trouble if a process in interrupted while still in kernel
mode.

The process that is being switched to will now use the same kernel stack, and
will then overwrite what the previous have written on the stack (remember that
TSS data structure points to the _beginning_ of the stack).

To solve this problem, every process should have it's own kernel stack, the
same way that each process have their own user mode stack. When switching
process, the TSS must then be updated to point to the new process kernel stack.

### Difficulties with preemptive scheduling
When using preemptive scheduling, one problem arises that doesn't exist with
cooperative scheduling. With cooperative scheduling, every time a process
yields, it must be in user mode (privilege level 3), since yield is a system
call. With preemptive scheduling, the processes can be interrupted in either
user mode or kernel mode (privilege level 0), since the process itself does not 
control when it gets interrupted.

Interrupting a process in kernel mode is a little bit different than
interrupting a process in user mode, due to the way the CPU sets up the stack
at interrupts. If a privilege level change occurs (the process was interrupted
in user mode), then the CPU will push the value of the process `eip` and `esp`
register on the stack. If no privilege level change occurs (the process was
interrupted in kernel mode), then the CPU won't push the `esp` register on the
stack. Furthermore, if there is no privilege level change, the CPU won't change
stack to the one defined it the TSS (this is actually an advantage, see the
section below).

To fix this, you must yourself calculate what the value of `esp` was _before_
the interrupt. Since you know that the CPU pushes 3 things on the stack when no
privilege change happens and you know how much you have pushed on the stack,
you can calculate what the value of `esp` was at the time of the interrupt.
This is possible since the CPU won't change stacks if there is no privilege
level change, so the content of `esp` will be the same as at the time of the
interrupt.

To further complicate things, you must think of how to handle case when
switching to a new process that should be running in kernel mode. Since you are
then using `iret` without a privilege level change, the CPU won't update the
value of `esp` with the one placed on the stack, you must update `esp`
yourself.

## Further reading
- For more information about different scheduling algorithms, see
  <http://wiki.osdev.org/Scheduling_Algorithms>
