# User Mode

User mode is now almost within our reach. There are just a few more steps
required. They don't seem too much work on paper, but they can take quite a few
hours to get right in code and design.

## Segments for User Mode

To enable user mode we need to add two more segments to the GDT. They are very
similar to the kernel segments we added when we [set up the
GDT](#the-global-descriptor-table-gdt) in the [chapter about
segmentation](#segmentation).

  Index   Offset   Name                 Address range             Type   DPL
-------  -------   -------------------  ------------------------- -----  ----
      3   `0x18`   user code segment    `0x00000000 - 0xFFFFFFFF` RX     PL3
      4   `0x20`   user data segment    `0x00000000 - 0xFFFFFFFF` RW     PL3

Table: The segment descriptors needed for user mode.

The difference is the DPL, which now allows code to execute in PL3. The
segments can still be used to address the entire address space, so just using
these segments for user mode code will not protect the kernel. For that we need
paging.

## Setting Up For User Mode

There are a few things every user mode process needs:

- Page frames for code, data and stack. For now we can just allocate one page frame
  for the stack, and enough page frames to fit the code and fixed-size data.

- We need to copy the binary from the GRUB modules to the frames used for code.

- We need a page directory and page tables to map these page frames into
  memory. At least two page tables are needed, because the code and data should
  be mapped in at `0x00000000` and increasing, and the stack should start just
  below the kernel, at `0xBFFFFFFB`, growing to lower addresses. Make sure the
  U/S flag is set to allow PL3 access.

It might be convenient to store these in a struct of some kind, dynamically
allocated with the kernel `malloc`.

## Entering User Mode

The only way to execute code with a lower privilege level than the current
privilege level (CPL) is to execute an `iret` or `lret` instruction - interrupt
return or long return, respectively.

To enter user mode, we set up the stack as if the processor had raised an
inter-privilege level interrupt. The stack should look like this:

~~~
    [esp + 16]  ss      ; the stack segment selector we want for user mode
    [esp + 12]  esp     ; the user mode stack pointer
    [esp +  8]  eflags  ; the control flags we want to use in user mode
    [esp +  4]  cs      ; the code segment selector
    [esp +  0]  eip     ; the instruction pointer of user mode code to execute
~~~

(source: The Intel manual [@intel3a], section 6.2.1, figure 6-4)

`iret` will then read these values from the stack and fill in the corresponding
registers. Before we execute `iret` we need to change to the page directory we
setup for the user mode process. Important to remember here is that, to
continue executing kernel code after we've switched PDT, the kernel needs to be
mapped in. One way to accomplish this is to have a "kernel PDT", which maps in
all kernel data at `0xC0000000` and above, and merge it with the user PDT
(which only maps below `0xC0000000`) when we do the switch. Also, remember that
we need to use the physical address for the page directory when we set `cr3`.

`eflags` is a register for a set of different flags, specified in section 2.3
of the Intel manual [@intel3a]. Most important for us is the interrupt enable
(IF) flag. When in PL3 we aren't allowed to use `sti` like we'd normally do to
enable interrupts. If interrupts are disabled when we enter user mode, we can't
enable them when we get there. When we use `iret` to enter user mode it will
set `eflags` for us, so setting the IF flag in the `eflags` entry on the stack
will enable interrupts in user mode.

For now, we should have interrupts disabled, as it requires a little more
twiddling to get inter-privilege level interrupts to work properly. See the
section on [system calls](#system-calls).

The `eip` should point to the entry point for the user code - `0x00000000` in
our case. `esp` should be where the stack should start - `0xBFFFFFFB`.

`cs` and `ss` should be the segment selectors for the user code and user data
segments, respectively. As we saw in the [segmentation
chapter](#creating-and-loading-the-gdt), the lowest two bits of a segment
selector is the RPL - the Requested Privilege Level. When we execute the `iret` we
want to enter PL3, so the RPL of `cs` and `ss` should be 3.

~~~
    cs = 0x18 | 0x3
    ss = 0x20 | 0x3
~~~

`ds` - and the other data segment registers - should be set to the same segment
selector as `ss`. They can be set the ordinary way, with `mov`.

Now we are ready to execute `iret`. If everything has been set up right, we now
have a kernel that can enter user mode. Yay!

## Using C for User Mode Programs

The kernel is compiled as an ELF [@wiki:elf] binary, which is a format that
allows for more complex and convenient layouts and functionality than a plain
flat binary. The reason we can use ELF for the kernel is that GRUB knows how to
parse the ELF structure.

If we implement an ELF parser, we can compile the user mode programs into ELF
binaries as well. We leave this as an exercise for the reader.

One thing we can do to make it easier to develop user mode programs is to allow
them to be written in C, but still compile them to flat binaries. In C the
layout of the generated code is more unpredictable and the entry point
(`main()`) might not be at offset 0. One way to work around this is to add a
few assembler lines placed at offset 0 which calls `main()`.

Assembler code (`start.s`):

~~~ {.nasm}
    extern main

    section .text
        ; push argv
        ; push argc
        call main
        ; main has returned, eax is return value
        jmp  $    ; loop forever
~~~

Linker script (`link.ld`) to place `start.o` first:

~~~
    OUTPUT_FORMAT("binary")    /* output flat binary */

    SECTIONS
    {
        . = 0;                 /* relocate to address 0 */

        .text ALIGN(4):
        {
            start.o(.text)     /* include the .text section of start.o */
            *(.text)           /* include all other .text sections */
        }

        .data ALIGN(4):
        {
            *(.data)
        }

        .rodata ALIGN(4):
        {
            *(.rodata*)
        }
    }
~~~

Note: `*(.text)` will not include the `.text` section of `start.o` again. See
[@ldcmdlang] for more details.

With this script we can write programs in C or assembler (or any other language
that compiles to object files linkable with `ld`), and it is easy to load and
map in for the kernel. (`.rodata` will be mapped in as writeable, though.)

When we compile user programs we want the usual `CFLAGS`:

~~~
    -m32 -nostdlib -nostdinc -fno-builtin -fno-stack-protector -nostartfiles
    -nodefaultlibs
~~~

And these `LDFLAGS`:

~~~
    -T link.ld -melf_i386  # emulate 32 bits ELF, the binary output is specified in the linker script
~~~

### A C Library

It might now be interesting to start thinking about writing a short libc for
your programs. Some of the functionality requires [system calls](#system-calls)
to work, but some, such as the `string.h` functions, does not.

## Further Reading

- Gustavo Duarte has an article on privilege levels:
  <http://duartes.org/gustavo/blog/post/cpu-rings-privilege-and-protection>
