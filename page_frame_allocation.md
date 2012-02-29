# Page frame allocation

Now that we have virtual memory with paging set up, we need to somehow allocate
some of the available memory to use for the applications we want to run. How do
we know what memory is free? That is the role of a page frame allocator.

## Managing available memory

### How much memory is there?

First we need to know how much memory the computer we're running on has.
Perhaps the easiest way to do this is to read it from the multiboot [@multiboot]
structure sent to us by GRUB. This will give us the information we need about
the memory - what is reserved, I/O mapped, read-only etc. The rest is free to
use. We must make sure that we don't mark the part of memory used by the kernel
as free. One way to do this is to export labels at the beginning and end of the
kernel binary from the linker script.

The updated linker script:

    ENTRY(loader)           /* the name of the entry symbol */

    . = 0xC0100000          /* the code should be relocated to 3GB + 1MB */

    /* these labels get exported to the code files */
    kernel_virtual_start = .;
    kernel_physical_start = . - 0xC0000000;

    /* align at 4KB and load at 1MB */
    .text ALIGN (0x1000) : AT(ADDR(.text)-0xC0000000)
    {
        *(.text)            /* all text sections from all files */
    }

    /* align at 4KB and load at 1MB + . */
    .rodata ALIGN (0x1000) : AT(ADDR(.rodata)-0xC0000000)
    {
        *(.rodata*)         /* all read-only data sections from all files */
    }

    /* align at 4KB and load at 1MB + . */
    .data ALIGN (0x1000) : AT(ADDR(.data)-0xC0000000)
    {
        *(.data)            /* all data sections from all files */
    }

    /* align at 4KB and load at 1MB + . */
    .bss ALIGN (0x1000) : AT(ADDR(.bss)-0xC0000000)
    {
        *(COMMON)           /* all COMMON sections from all files */
        *(.bss)             /* all bss sections from all files */
    }

    kernel_virtual_end = .;
    kernel_physical_end = . - 0xC0000000;

NASM code can now read these labels directly, and perhaps push them onto the
stack so we can use them from C:

~~~ {.nasm}
extern kernel_virtual_start
extern kernel_virtual_end
extern kernel_physical_start
extern kernel_physical_end

; ...

push kernel_physical_end
push kernel_physical_start
push kernel_virtual_end
push kernel_virtual_start

call some_function
~~~

This way we get them as arguments to a C function. There is no clean way to take
the address of a label directly from C. One way to do it is to declare the
label as a function and take the address of the function:

~~~ {.c}
void kernel_virtual_start(void);

/* ... */

unsigned int vaddr = (unsigned int) &kernel_virtual_start;
~~~

If we use GRUB modules we also need to make sure the memory they use is marked
as "in use" as well.

Note that not all available memory needs to be contiguous. In the first 1MB
there are several I/O-mapped memory sections, as well as memory used by GRUB
and the BIOS. Other parts of the memory might be similarly unavailable.

It's convenient to divide the memory sections into complete page frames, as we
can't map part of pages into memory.

### Managing available memory

How do we know which page frames are in use? The page frame allocator needs to
keep track of which are free and which aren't. There are several ways to do
this: Bitmaps, linked lists, trees, the Buddy System (used by Linux) etc. See
<http://wiki.osdev.org/Page_Frame_Allocation> for more details.

Bitmaps are quite easy to implement. One bit for each page frame, and dedicate
one (or more) page frames to store the bitmap. But other designs might be
nicer and/or more fun.

## How can we access a page frame?

When we use our page frame allocator to allocate page frames, it gives us the
physical start address of the page frame. This page frame is not mapped in - no
page table points to the frame. (If we mapped an entire 4MB chunk for the
kernel, and the page frame lies somewhere in there, it will be mapped in.) How
can we read and write data to the frame?

We need to map the page frame into virtual memory. Just update the PDT and/or
PT used by the kernel.

But what if all page tables available are full? Then we can't map the page
frame into memory, because we'd need a new page table - which takes up an
entire page frame - and to write to this page frame we'd need to map it in...

One solution is to reserve part of the first page table used by the kernel (or
some other higher-half page table) for temporarily mapping in page frames so
that we can access to them. If the kernel is mapped in at `0xC0000000` (page
directory entry with index 768), and we've used 4KB page frames, the kernel
has at least one page table. If we assume, or limit us to, a kernel of size
at most 4MB minus 4KB, we can dedicate the last entry (entry 1023) of this page
table for temporary mappings. The virtual address of pages mapped in like this
will be:

    (768 << 22) | (1023 << 12) | 0x000 = 0xC03FF000

After we've temporarily mapped in the page frame we want to use as a page
table, and set it up to map in our first page frame, we can add it to the
paging directory, and remove the temporary mapping. (This leads to the quite
nice property that no paging tables need to be mapped in unless we need to
edit them).

## A kernel heap

So far we've been able to work with only fixed-size data, or directly with raw
memory. Now that we have a page frame allocator we can implement `malloc` and
`free` to use in the kernel.

Kernigan and Ritchie have an example implementation in [@knr] that we can draw
inspiration from. The only real modifications we need to do is to remove calls
to `sbrk`/`brk`, and directly ask the page frame allocator for the page frames,
and the paging system to map them in.

This can be done quick and dirty. A real implementation should also give back
page frames to the page frame allocator on `free`, whenever sufficiently large
blocks are freed.

## Further reading

- The OSDev wiki page on page frame allocation:
  <http://wiki.osdev.org/Page_Frame_Allocation>
