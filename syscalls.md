# System Calls

## What is a System Call?

System calls (syscalls) is the way for user-mode applications to interact with
the kernel - to ask for resources, request operations to be performed, etc. The
syscall API is the part of the kernel most exposed to the users, so its design
requires some thought.

## Designing System Calls

It is up to us, the kernel developers, to design the system calls that
applications developer can use. We can draw inspiration from the POSIX
standards or, if they seem a bit too much, just look at the ones for Linux, and
pick and choose. See [further reading](#further-reading-7) below.

## Implementing System Calls

Syscalls are traditionally implemented with software interrupts. The user
applications put the appropriate values in registers or on the stack and then
initiates a pre-defined interrupt which transfers execution to the kernel. The
interrupt number used is kernel-dependent. Linux uses `0x80`.

With "newer" versions of x86, Intel and AMD has implemented two different ways
to initiate syscalls that are faster than interrupts - `syscall`/`sysenter` and
`sysret`/`sysexit`, respectively.

When syscalls are executed, the current privilege level is typically changed
from PL3 to PL0. To allow this, the DPL of the entry in the IDT for the syscall
interrupt needs to allow PL3 access.

Whenever inter-privilege level interrupts occur, the processor pushes a few
important registers on the stack - the same ones we used to enter user mode
[before](#user-mode), see figure 6-4, section 6.12.1, in the Intel manual
[@intel3a]. But what stack is used? The same section in [@intel3a] specifies
that if the interrupt leads to code executing at numerically lower privilege
levels, a stack switch occurs. The values for the new `ss` and `esp` is taken
from the current Task State Segment (TSS). The TSS structure is specified in
figure 7-2, section 7.2.1 of [@intel3a].

To enable syscalls we need to setup a TSS for our currently running programs
before we enter user mode. Setting it up can easily be done in C, where we
enter the `ss` and `esp` values. To load it into the processor, we need to set
up a new TSS descriptor in our GDT. The structure of this descriptor is
described in section 7.2.2 in [@intel3a].

We specify the current TSS segment selector by loading it into the `tr`
register with `ltr`. If the TSS segment descriptor has index 5, and thus offset
`5 * 8 = 40 = 0x28`, this is what we should load into `tr`.

When we [entered user mode before](#entering-user-mode) we disabled interrupts
when executing in PL3. For syscalls via interrupts to work, we need to enable
interrupts in user mode. We enter user mode by using the `iret` instruction,
and this loads the `eflags` register from the stack for us. Setting the IF flag
bit in the `eflags` on the stack will make `iret` enable interrupts for us.

## Further Reading

- The Wikipedia page on POSIX, with links to the specifications:
  <http://en.wikipedia.org/wiki/POSIX>
- A list of syscalls used in linux:
  <http://bluemaster.iu.hio.no/edu/dark/lin-asm/syscalls.html>
- The Wikipedia page on system calls:
  <http://en.wikipedia.org/wiki/System_call>
- The Intel manual [@intel3a] sections on interrupts (chapter 6) and TSS'
  (chapter 7) are where you get all the details you need.
