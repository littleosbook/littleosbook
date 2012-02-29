# The Road to User Mode

Now that the kernel boots, prints to screen and reads from keyboard - what do
we do? Usually, a kernel is not supposed to do the application logic itself,
but leave that for actual applications. The kernel creates the proper
abstractions (for memory, files, devices, etc.) so that application development
becomes easier, performs tasks on behalf of applications (system calls), and
[schedules processes](#scheduling).

User mode, in contrast with kernel mode, is how the user's programs executes.
It is less privileged than the kernel, and will prevent badly written user
programs from messing with other programs or the kernel. Badly written kernels
are free to mess up what they want.

There's quite a way to go until we get there, but here follows a
quick-and-dirty start.

## Loading a program

Where do we get the application from? Somehow we need to load the code we want
to execute into memory. More feature complete operating system usually have
drivers and file system that enables them load the software from a CD-ROM
drive, a hard disk or a floppy disk. We'll do file systems in a [later
section](#file-systems).

Instead of creating all these drivers and file systems, we can simply use a
feature in GRUB called modules to load our program.

### GRUB Modules

GRUB can load arbitrary files into memory from the ISO image and these files are
usually referred to as modules. To make GRUB load a module, edit the file
`iso/boot/grub/menu.lst` and add the following line at the end of the file

    module /modules/program

Now create the folder `iso/modules`

    mkdir -p iso/modules

Later in this chapter, the application "program" will be created. We must also
update the code that calls `kmain` to pass information to `kmain` about where
it can find the modules. We also want to tell GRUB that it should align all the
modules on page boundaries when loading them (see the chapter about
["Paging"](#paging) for details about page alignment).

To instruct GRUB how to load our modules, the "multiboot header", that is the
first bytes of the kernel, must be updated as

~~~ {.nasm}
; in file `loader.s`


MAGIC_NUMBER    equ 0x1BADB002      ; define the magic number constant
ALIGN_MODULES   equ 0x00000001      ; tell GRUB to align modules

; calculate the checksum (all options + checksum should equal 0)
CHECKSUM        equ -(MAGIC_NUMBER + ALIGN_MODULES)

section .text:                      ; start of the text (code) section
align 4                             ; the code must be 4 byte aligned
    dd MAGIC_NUMBER                 ; write the magic number
    dd ALIGN_MODULES                ; write the align modules instruction
    dd CHECKSUM                     ; write the checksum
~~~

GRUB will also store a pointer to a `struct` in the register `ebx` that among
other things describes at which addresses the modules are loaded. Therefore,
you probably want to push `ebx` on the stack before calling `kmain` to make
into an argument for `kmain`.

## Executing the program

### A very simple program

Since for now any program we'd write would have trouble to communicate with the
outside, a very short program that writes a value to a register suffices.
Halting Bochs and reading its log should verify that the program has run.

~~~ {.nasm}
; set eax to some distinguishable number, to read from the log afterwards
mov eax, 0xDEADBEEF

; enter infinite loop, nothing more to do
; $ means "beginning of line", ie. the same instruction
jmp $
~~~

### Compiling

Since our kernel cannot parse advanced executable formats, we need to compile
the code into a flat binary. NASM can do it like this:

    nasm -f bin program.s -o program

This is all we need. You must now move the file `program` to the folder
`iso/modules`.


### Finding the program in memory
Before jumping to the program, we must find where it resides in memory.
Assuming that the contents of the `ebx` is passed as an argument to `kmain`, we
can do this entirely from C.

The pointer in `ebx` points to a multiboot-structure [@multiboot]. Download the
file `multiboot.h` from
<http://www.gnu.org/software/grub/manual/multiboot/html_node/multiboot.h.html>,
which describes the structure.

The pointer passed to `kmain` from the `ebx` register can now be casted to
`multiboot_info_t` pointer. The address of the first module is now in the field
`mods_addr`, that is

~~~ {.c}
int kmain(..., unsigned int ebx)
{
    multiboot_info_t *mbinfo = (multiboot_info_t *) ebx;
    unsigned int address_of_module = mbinfo->mods_addr;
}
~~~

However, before just blindly following the pointer, you should check that the
module got loaded correctly. This can be done by checking the `flags` field of
the `multiboot_info_t` stucture. You should also check the field `mods_count`
to make sure it's exactly 1. For more details about the multiboot structure,
see [@multiboot].

### Jumping to the code
What we'd like to do now is just jump to the address of the GRUB-loaded module.
Since it is easier to parse the multiboot structure in C, calling the code from
C is more convenient, but it can of course be done with `jmp` (or `call`) in
assembler as well.

~~~ {.c}
typedef void (*call_module_t)(void);
/* ... */
call_module_t start_program = (call_module_t) address_of_module;
start_program();
/* we'll never get here, unless the module code returns */
~~~

If we start the kernel, wait until it has run and entered the infinite loop in
the program, and halt Bochs we should see `0xDEADBEEF` in `eax`. We have
successfully started a program in our OS!

## The beginning of user mode

The program we've written now runs in the same mode as the kernel, we've just
entered it in a somewhat peculiar way. Do enable applications to really execute
in user mode, we'll need to do [segmentation](#segmentation), [paging](#paging)
and [allocate page frames](#page-frame-allocation).

It's quite a lot of work and technical details to go through. But in a few
chapters we'll have working user mode programs.
