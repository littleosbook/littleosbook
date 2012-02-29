# First steps

Developing an operating system (OS) is no easy task, and the question " Where
do I start?" is likely to come up several times during the course of the
project.  This chapter will help you set up your development environment and
booting a very small (and boring) operating system.

## Requirements
In this book, we will assume that you are familiar with the C programming
language and that you have some programming experience. It will be also be
helpful if you have basic understanding of assembly code.

## Tools

### The easy way
The easiest way to get all the required tools is to use Ubuntu [@ubuntu] as
your operating system. If you don't want to run Ubuntu natively on your
computer, it works just as well running it in a virtual machine, for example in
VirtualBox [@virtualbox].

The packages needed can then be installed by running

    sudo apt-get install build-essential nasm genisoimage bochs bochs-x

### Programming languages
The operating system will be developed using the C programming language
[@knr][@wiki:c]. The reason for using C is because developing and OS requires a very
precise control of the generated code and direct access to the memory,
something which C enables. Other languages can be used that does the same, for
example C++, can also be used, but this book will only cover C.

The code will make use of one type attribute that is specific for GCC [@gcc]

    __attribute__((packed))__

(the meaning of the attribute will be explained later).
Therefore, the example code might hard to compile using another C
compiler.

For writing assembly, we choose NASM [@nasm] as the assembler. The reason for
this is that we prefer the NASMs syntax over GNU Assembler.

Shell [@wiki:shell] will be used as the scripting language throughout the book.

### Host operating system
All the examples assumes that the code is being compiled on a UNIX like
operating system. All the code is known to compile on Ubuntu [@ubuntu] versions
11.04 and 11.10.

### Build system
GNU Make [@make] has been used when constructing the example Makefiles, but we
don't make use of any GNU Make specific instructions.

### Virtual machine
When developing an OS, it's very convenient to be able to run your code in a
virtual machine instead of a physical one. Bochs [@bochs] is emulator for the
x86 (IA-32) platform which is well suited for OS development due to its
debugging features. Other popular choices are QEMU [@qemu] and VirtualBox
[@virtualbox], but we will use Bochs in this book.

## Booting
The first goal when starting to develop an operating system is to be able to
boot it. Booting an operating system consists of transferring control to a
chain of small programs, each one more "powerful" than the previous one, where
the operating system is the last "program". See figure 1-1 for an overview of
the boot process.

![Figure 1-1: An overview of the boot process, each box is a program.](images/boot_chain.png)

### BIOS
When the PC is turned on, the computer will start a small program called that
adheres to the Basic Input Output System (BIOS) [@wiki:bios] standard.
The program is usually
stored in a read only memory chip on the motherboard of the PC. The
original role of
the BIOS program was export some library functions for printing to the screen,
reading keyboard input etc. However, todays operating system does not use the
BIOS functions, instead they interact with the hardware using drivers that
interacts directly with the hardware, bypassing the BIOS [@wiki:bios].

BIOS is a legacy technology, it operates in 16-bit mode (all x86 CPUs are
backwards compatible with 16-bit mode). However, they still have one very
important task to perform, and that is transferring control to the next program
in the chain.

### The bootloader
The BIOS program will transfer control of the PC to a program called
_bootloader_. The bootloaders task is to load the to transfer control to us, the
operating system developers, and our code. However, due to some legacy
restrictions, the bootloader is often split into two parts, so the first part
of the bootloader will transfer control to the second part which will finally
give the control of the PC to the operating system.

Writing a bootloader involves writing a lot of low-level code that is
interacting with the BIOS, so in this book, we will use an existing bootloader,
the GNU GRand Unified Bootloader (GRUB) [@grub].

Using GRUB, the operating system can be built as an ordinary ELF [@wiki:elf]
file, which will be loaded by GRUB into the correct memory location. However,
the compilation requires some care regarding how things are to by laid out in
memory, which will be discussed later in this chapter.

### The operating system
GRUB will transfer control to the operating system by jumping to the kernels
start position in memory. Before the jump, GRUB will also look for a magic
number to make sure that is is actually jumping to a kernel and not some random
code. The magic number is part of the multiboot specification [@multiboot]
which GRUB adheres to.

## Hello CAFEBABE
This section will show the implementation of the smallest possible operating
system kernel that can be used together with GRUB.

### Compiling the kernel
This part of the kernel has
to be written in assembly, since C requires a stack, which isn't available at
this point, since GRUB doesn't set one up. The code for the kernel then is:

~~~ {.nasm}
global loader                   ; the entry symbol for ELF

MAGIC_NUMBER equ 0x1BADB002     ; define the magic number constant
CHECKSUM     equ -MAGIC_NUMBER  ; calculate the checksum (magic number + checksum should equal 0)

section .text:                  ; start of the text (code) section
align 4                         ; the code must be 4 byte aligned
    dd MAGIC_NUMBER             ; write the magic number
    dd CHECKSUM                 ; write the checksum

loader:                         ; the loader label
    mov eax, 0xCAFEBABE         ; place the number 0xCAFEBABE in the register eax
.loop:
    jmp .loop                   ; loop forever
~~~

The only thing the kernel will do is write the very specific number
`0xCAFEBABE` into the `eax` register. It is _very_ unlikely that the number
`0xCAFEBABE` would be in the `eax` register if our kernel didn't boot. Save the
code for a file named `loader.s`.

The assembly code can now be compiled with the commando

    nasm -f elf32 loader.s

to produce an ELF 32 bits objects file.

### Linking the kernel
The code must now be linked to produce an executable file, which requires some
extra thought compared to when linking most programs. The reason for this is
that we want GRUB to load the kernel at a memory address larger than or equal
to `0x00100000` (1 mega byte (MB)). This is because addresses lower than 1 MB
might be used by GRUB itself, by BIOS and for memory-mapped I/O. Therefore, the
following linker script is needed:

    ENTRY(loader)           /* the name of the entry symbol */

    . = 0x00100000          /* the code should be loaded at 1 MB */

    .text ALIGN (0x1000)    /* align at 4 KB */
    {
        *(.text)            /* all text sections from all files */
    }

    .rodata ALIGN (0x1000)  /* align at 4 KB */
    {
        *(.rodata*)         /* all read-only data sections from all files */
    }

    .data ALIGN (0x1000)    /* align at 4 KB */
    {
        *(.data)            /* all data sections from all files */
    }

    .bss ALIGN (0x1000)     /* align at 4 KB */
    {
        *(COMMON)           /* all COMMON sections from all files */
        *(.bss)             /* all bss sections from all files */
    }

Save the linker script into a file called `link.ld`. The executable can now be
linked by running

    ld -T link.ld -melf_i386 loader.o -o kernel.elf

where the file `kernel.elf` is the final executable.

### Obtaining GRUB
The GRUB version we will use is GRUB Legacy, since then the ISO image can be
generated on systems using both GRUB Legacy and GRUB 2. We need the GRUB Legacy
`stage2_eltorito` bootloader. This file be built from GRUB 0.97 by downloading
the source from <ftp://alpha.gnu.org/gnu/grub/grub-0.97.tar.gz>. However, the
source code `configure` script doesn't work well with Ubuntu, so the binary
files can be downloaded from <http://ijmul.com/dnlds/grub_files.tar.gz>. Locate
the file `stage2_eltorito` and copy to your current folder.

### Building an ISO image
Now the code must be placed on a media that can be loaded by a virtual (or
physical machine). In this book, we will use ISO [@wiki:iso] image files as the
media, but one can also use floppy images, depending on what the
virtual or physical machine supports.

The ISO image will be created with the program `genisoimage`. A folder must
first be created that contains the files that will be on ISO image. The
following commands create the folder and copy the files to their correct
places:

    mkdir -p iso/boot/grub
    cp stage2_eltorito iso/boot/grub/
    cp kernel.ef iso/boot/

Now a configuration file `menu.lst` for GRUB must be created

    default=0
    timeout=0

    title minios
    kernel /boot/kernel.elf

and be placed in the `iso/boot/grub/` folder. The `iso` folder should now look
like:

    iso
    |-- boot
      |-- grub
      | |-- menu.lst
      | |-- stage2_eltorito
      |-- kernel.elf

The ISO image can now be generated with the command

    genisoimage -R                              \
                -b boot/grub/stage2_eltorito    \
                -no-emul-boot                   \
                -boot-load-size 4               \
                -A minios                       \
                -input-charset utf8             \
                -quiet                          \
                -boot-info-table                \
                -o minios.iso                   \
                iso

where the flags are

-------------------------------------------------------------------------------
           Flag Description
--------------- ---------------------------------------------------------------
              R Use the Rock Ridge protocol (needed by GRUB)

              b The file to boot from (relative to the root folder of the ISO)

   no-emul-boot Do not perform any disk emulation

 boot-load-size The number 512 byte sectors to load. Apparently most BIOS
                likes the number 4.

boot-info-table Writes information about the ISO layout to ISO (needed by GRUB)

              o The name of the iso

              A The label of the iso

  input-charset The charset for the input files

          quiet Disable any output from genisoimage
-------------------------------------------------------------------------------

### Running BOCHS
Now that we have an ISO image with operating system and GRUB, the final step is
to run it in BOCHS. BOCHS needs a configuration file to start, and a simple
configuration file is given below

    megs:            32
    display_library: x
    romimage:        file=/usr/share/bochs/BIOS-bochs-latest
    vgaromimage:     file=/usr/share/bochs/VGABIOS-lgpl-latest
    ata0-master:     type=cdrom, path=minios.iso, status=inserted
    boot:            cdrom
    log:             bochslog.txt
    mouse:           enabled=1
    clock:           sync=realtime, time0=local
    cpu:             count=1, ips=1000000

you might need to change the path to `romimage` and `vgaromimage` to match the
installation directory of BOCHS on your computer. Save the file as
`bochsrc.txt`.

You can now start the operating system by running

    bochs -f bochsrc.txt -q

`-f` tells BOCHS to use the given configuration file and `-q` tells BOCHS to
skip the interactive start menu. You should now see BOCHS starting and
displaying a console with some information from GRUB on it.

After quitting BOCHS, display the file log from BOCHS by running

    cat bochslog.txt

You should now see the state of the registers of BOCHS somewhere in the output.
If you find `RAX=00000000CAFEBABE` or `EAX=CAFEBABE` (depending on if you are
running BOCHS with or without 64 bit support), then your operating
system has booted successfully!

## Further reading
- Gustavo Duertes has written an in-depth article about what actually happens
  when a computers boots up,
  <http://duartes.org/gustavo/blog/post/how-computers-boot-up>
- Gustavo then continues to describe what the kernel does in the very early
  stages at <http://duartes.org/gustavo/blog/post/kernel-boot-process>
- The OSDev wiki also contains a nice article about booting a computer,
  <http://wiki.osdev.org/Boot_Sequence>
