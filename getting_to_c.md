# Getting to C

Now that you've managed to boot your operating system, it's time to reconsider
the language we're currently using, assembly. Assembly is very good for
interacting with the CPU and enables maximum control over every aspect of the
code.  However, at least for the authors, C is a much more convenient language
to use. Therefore, we would like to use C as much as possible and only
assembler where it make sense.

## Setting Up a Stack
Before we can program in C, we need a stack. This is because we can _not_
guarantee that a C program does _not_ make use of a stack. Setting up a stack
is not harder than to make the `esp` register point to the end of an area of
free memory that is correctly aligned (alignment on 4 bytes is recommended from
a performance perspective). Also remember that the stack grows towards lower
addresses.

We could make `esp` just point to some area in memory, since so far, the only
thing in the memory is GRUB, BIOS, the OS kernel and some memory-mapped I/O.
This is not a good idea, since we don't know which address to point to, since
we don't know how much memory is available, and if any memory is reserved by
BIOS. A better idea is to reserve a piece of memory in the `bss` section in the
ELF executable of the kernel. That way, when GRUB loads the ELF binary, GRUB
will allocate stack memory for us.

To declare uninitialized data, the NASM pseudo-instruction `resb` [@resb] can
be used:

~~~ {.nasm}
    KERNEL_STACK_SIZE equ 4096                  ; size of stack in bytes

    section .bss
    align 4                                     ; align at 4 bytes
    kernel_stack:                               ; label points to beginning of memory
        resb KERNEL_STACK_SIZE                  ; reserve stack for the kernel
~~~

We can use uninitialized memory because before any stack location is read from,
it should be written to (or the code reads garbage).

Setting up the stack pointer is then trivial:

~~~ {.nasm}
    mov esp, kernel_stack + KERNEL_STACK_SIZE   ; point esp to the start of the
                                                ; stack (end of memory area)
~~~

## Calling C Code From Assembly
The next step is to call a C function from the assembly code. There are many
different calling conventions for how to call C code from assembly
[@wiki:ccall], but we will use the _cdecl_ calling convention, since it's the
one used by GCC. The cdecl calling convention states that arguments to a
function should be passed via the stack (on x86). The arguments of the function
should be pushed on the stack in a right-to-left order, that is, you push the
rightmost argument first. The return value of the function is placed in the
`eax` register. The following is an example:

~~~ {.c}
    int sum_of_three(int arg1, int arg2, int arg3)
    {
        return arg1 + arg2 + arg3;
    }
~~~

~~~ {.nasm}
    external sum_of_three   ; the function sum_of_three is defined elsewhere

    push dword 3            ; arg3
    push dword 2            ; arg2
    push dword 1            ; arg1
    call sum_of_three       ; call the function, the result will be in eax
~~~

### Packing Structs
In the rest of book, you will often come across "configuration bytes" that are
a collection of bits in a very specific order. For example, a configuration
could look like

    Bit:     | 31           16 | 15     8 | 7     0 |
    Content: | address         | index    | config  |

Instead of using an unsigned integer, `unsigned int`, for handling such
configurations, it is much more convenient to use "packed structures":

~~~ {.C}
    struct example {
        unsigned char config;
        unsigned short address;
        unsigned char index;
    };
~~~

When using the struct in the previous example there is no guarantee that the
size of the `struct` will be exactly 32 bits - the compiler can add some
padding in order to speed up element access. When using a `struct` to represent
configuration bytes, it is very important that the size does _not_ get padded,
since the struct will eventually be treated as an unsigned integer by the
hardware. To force GCC to _not_ add any padding, the attribute `packed` can be
used in the following way:

~~~ {.C}
    struct example {
        unsigned char config;
        unsigned short address;
        unsigned char index;
    } __attribute__((packed));
~~~

Note that `__attribute__((packed))` is not part of the C standard, so it might
not work with all compilers (it works with GCC and Clang [@clang]).

## Compiling C Code
When compiling the C code for the OS, quite a lot of flags to GCC has to be
used. The reason for this is that the C code should _not_ assume the presence
of the standard library, since there is no standard library available for our
OS (yet). For more information about the flags, see the GCC manual. 

The flags used for compiling C are:

~~~
    -m32 -nostdlib -nostdinc -fno-builtin -fno-stack-protector -nostartfiles
    -nodefaultlibs
~~~

As always when writing C programs, we recommend turning on all warnings and
treat warnings as errors:

~~~
    -Wall -Wextra -Werror
~~~

You can now create a `kmain` function in a file called `kmain.c` that you can
call from `loader.s`. At this point, `kmain` probably won't need any arguments
(but in later chapters it will).

## Build Tools
Now is also probably a good time to set up some build tools to make it easier
to compile and test-run the OS. We recommend using make [@make], but there are
plenty of other build systems available. A simple Makefile for the OS could
look like this:

~~~ {.Makefile}
    OBJECTS = loader.o kmain.o
    CC = gcc
    CFLAGS = -m32 -nostdlib -nostdinc -fno-builtin -fno-stack-protector \
             -nostartfiles -nodefaultlibs -Wall -Wextra -Werror -c
    LDFLAGS = -T link.ld -melf_i386
    AS = nasm
    ASFLAGS = -f elf

    all: kernel.elf

    kernel.elf: $(OBJECTS)
        ld $(LDFLAGS) $(OBJECTS) -o kernel.elf

    os.iso: kernel.elf
        cp kernel.elf iso/boot/kernel.elf
        genisoimage -R                              \
                    -b boot/grub/stage2_eltorito    \
                    -no-emul-boot                   \
                    -boot-load-size 4               \
                    -A os                           \
                    -input-charset utf8             \
                    -quiet                          \
                    -boot-info-table                \
                    -o os.iso                       \
                    iso

    run: os.iso
        bochs -f bochsrc.txt -q

    %.o: %.c
        $(CC) $(CFLAGS)  $< -o $@

    %.o: %.s
        $(AS) $(ASFLAGS) $< -o $@

    clean:
        rm -rf *.o kernel.elf os.iso
~~~

If you have followed the tutorial your directory now looks something like:

~~~
    .
    |-- bochsrc.txt
    |-- iso
    |   |-- boot
    |     |-- grub
    |       |-- menu.lst
    |       |-- stage2_eltorito
    |-- kmain.c
    |-- loader.s
~~~

You should be now able to start the OS in Bochs with the simple command:

~~~
    make run
~~~

## Further Reading

- Kernigan & Richies book, _The C Programming Language, Second Edition_, [@knr]
  is great for learning about all the aspects of C.
