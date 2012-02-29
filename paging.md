# Paging

Segmentation translates a logical address into a linear address. Paging
translates these linear addresses onto the physical address space, and
determines access rights and how the memory should be cached.

## Why paging?

Paging is the most common technique used in x86 to enable virtual memory.
Virtual memory means that the each process will get the impression that the
available memory range is `0x00000000` - `0xFFFFFFFF` even though the actual
size of the memory is way less. It also means that when a process addresses a
byte of memory, they will use a virtual address (linear address) instead of
physical one. The code in the user process won't notice any difference (except
for execution delays...). The linear address will then get translated to a
physical address by the MMU and the page table. If the virtual address isn't
mapped to a physical address, the CPU will raise a page fault interrupt.

Paging is optional, and some operating systems don't need it. But if we want
memory access control (so that we can have processes running in different
privilege levels), paging is the neatest way to do it.

## Paging in x86

x86 paging (chapter 4 in the Intel manual [@intel3a]) consists of a page
directory (PDT) that can contain references to 1024 page tables (PT), each of
which can point to 1024 sections of physical memory called page frames (PF);
Each page frame is 4096 byte large. In a virtual address (linear address), the
highest 10 bits specifies the offset of a page directory entry (PDE) in the
current PDT, the next highest 10 bits the offset of a page table entry (PTE)
within the page table pointed to by that PDE. The lowest 12 bits in the address
is the offset within the page frame to be addressed.

All page directories, page tables and page frames need to be aligned on 4096
byte addresses. This makes it possible to address a PDT, PT or PF with just the
highest 20 bits of a 32 bit address, since the lowest 12 need to be zero.

The PDE and PTE structure is very similar to each other: 32 bits (4 bytes),
where the highest 20 bits points to a PTE or PF, and the lowest 12 bits control
access rights and other configurations. 4 bytes * 1024 equals 4096 bytes, so a
page directory and page table both fit in a page frame themselves.

The translation of linear addresses to physical addresses is described in
the figure below.

It is also possible to use 4MB pages. A PDE then points directly to a 4MB page
frame, which needs to be aligned on a 4MB address boundary. The address
translation is almost the same as in the figure, with just the page table step
removed. It is possible to mix 4MB and 4KB pages.

![Figure: Translating virtual addresses (linear addresses) to physical addresses.
](images/intel_4_2_linear_address_translation.png)

The 20 bits pointing to the current PDT is stored in the `cr3` register, with
lower bytes doing some configuration.

For more details on the paging structures, see chapter 4 in the Intel manual
[@intel3a]. The most interesting bits are U/S, which determine what privilege
levels can access this page (PL0 or PL3), and R/W, which makes the memory in
the page read-write or read-only.

### Identity paging

The simplest kind of paging is when we map each virtual address onto the same
physical address. This can be done at compile time by, for instance, creating a
page table where each entry points to its corresponding 4MB frame. In NASM we
can do this with macros and commands (`%rep`, `times` and `dd`), or just
ordinary assembler instructions.

### Enabling paging

You enable paging by first writing the address of a page directory to `cr3` and
then setting bit 31 (the PG "paging-enable" bit) of `cr0` to `1`. If you want 4MB
pages, set the PSE bit (Page Size Extensions, bit 4) of `cr4`.

~~~ {.nasm}
; eax has the address of the page directory
; optional: or eax, "paging confiuration"
mov cr3, eax

mov ebx, cr4        ; read current cr4
or  ebx, 0x00000010 ; set PSE
mov cr4, ebx        ; update cr4

mov ebx, cr0        ; read current cr0
or  ebx, 0x80000000 ; set PG
mov cr0, ebx        ; update cr0

; now paging is enabled
~~~

### A few details

It is important to note that all addresses within the page directory, page
tables and in `cr3` needs to be physical addresses to the structures, never
virtual. This will be more relevant in later sections where we dynamically
update the paging structures, such as in the chapter on [user
mode](#user-mode).

An instruction that is useful when changing doing paging is `invlpg`. It
invalidates the TLB (Translation Lookaside Buffer) for a virtual address. This
is only required when changing a PDE or PTE that was previously mapped to
something else. If it had previously been marked as not present (bit 0 was set
to 0), doing `invlpg` is unnecessary. Also, changing the value of `cr3` will
cause all entries in the TLB to be invalidated.

Example:

~~~ {.nasm}
; invalidate any TLB references to virtual address 0
invlpg [0]
~~~

## Higher-half kernel

When GRUB loads our kernel into memory, it places the kernel at the physical
memory starting at 1MB. With identity mapping, this is also the virtual address
of the kernel.

### Reasons to not identity map the kernel

If the kernel is placed at the beginning of the virtual address - that is, the
virtual address `0x00000000` to `"size of kernel"` will map to the location of
the kernel in memory - there will be issues when linking the user mode process
code. Normally, during linking, the linker assumes that the code will be loaded
into the memory position `0x00000000`. Therefore, when resolving absolute
references, `0x00000000` will be the base address for calculating the exact
position. But if the kernel is mapped onto the virtual address space
(`0x00000000`, `"size of kernel"`), the user mode process cannot be loaded at
virtual address `0x00000000` - it must be placed somewhere else. Therefore, the
assumption from the linker that the user mode process is loaded into memory at
position `0x00000000` is wrong. This can be corrected by using a linker script
which tells the linker to assume a different starting address, but that is very
cumbersome solution for the users of the operating system.

This also assumes that we want the kernel to be part of the user mode process'
address space. As we will see later, this is kind of nice, since then during
syscalls we don't have to change any paging structures to get access to the
kernels code and data. (The kernel pages should be marked as for PL0 only, of
course.)

### The virtual address for the kernel

Preferably, the kernel should be placed at a very high virtual memory address,
for example `0xC000000` (3GB). The user mode process is not likely to
be 3GB large, which is the now the only way that it can conflict with the
kernel.

If the user mode process is larger than 3GB, some pages will need to be
swapped out by the kernel. Swapping pages will not be part of this book.

### Placing the kernel at `0xC000000`

It isn't too hard, but it does require some thought. This is once again a
linking problem.  When the linker resolves all absolute references in the
kernel, it will assume that our kernel is loaded at physical memory location
`0x00100000`, not `0x00000000`, since we tell it so in our linker script (see
the section on [linking the kernel](#linking-the-kernel).  However, we want the
jumps to be resolved to `0xC0100000` as base, since otherwise a kernel jump
will jump straight into the user mode process code (remember that the user mode
process is loaded at virtual memory `0x00000000` and up).

But, we can't simply tell the linker to assume that the kernel starts (is
loaded) at `0xC01000000`, since we want it to be placed at 1 MB. The reason for
having the kernel loaded at 1MB is because it can't be loaded at `0x00000000`,
since there is BIOS and GRUB code loaded below 1MB. We also cannot assume that
we can load the kernel at 0xC010000, since the machine might not have 3GB of
physical memory.

This can be solved by using both relocation (`.=0xC01000000`) and the `AT`
instruction in the linker script. Relocation specifies that non-relative
memory-references should should use the relocation address as base in address
calculations. `AT` specifies where the kernel should be loaded into memory.
Relocation is done as link time by `ld` [@ldcmdlang], the load address
specified by `AT` is handled by GRUB when loading the kernel, and is part of
the ELF format [@wiki:elf].

### Higher-half linker script

We can modify the [first linker script](#linking-the-kernel) to implement this:

    ENTRY(loader)           /* the name of the entry symbol */

    . = 0xC0100000          /* the code should be relocated to 3GB + 1MB */

    /* align at 4KB and load at 1MB */
    .text ALIGN (0x1000) : AT(ADDR(.text)-0xC0000000)
    {
        *(.text)            /* all text sections from all files */
    }

    /* align at 4KB and load at 1MB + . */
    .rodata ALIGN (0x1000) : AT(ADDR(.text)-0xC0000000)
    {
        *(.rodata*)         /* all read-only data sections from all files */
    }

    /* align at 4KB and load at 1MB + . */
    .data ALIGN (0x1000) : AT(ADDR(.text)-0xC0000000)
    {
        *(.data)            /* all data sections from all files */
    }

    /* align at 4KB and load at 1MB + . */
    .bss ALIGN (0x1000) : AT(ADDR(.text)-0xC0000000)
    {
        *(COMMON)           /* all COMMON sections from all files */
        *(.bss)             /* all bss sections from all files */
    }

### Entering the higher half

When GRUB jumps to the kernel code, there is no paging table.  Therefore, all
references to `0xC0100000 + X` won't be mapped to the correct physical address,
and will therefore cause a general protection exception (GPE) at the very best,
otherwise (if the user has more than 3GB of memory) the OS will just crash.

Therefore, some assembly that doesn't use absolute jumps must set up the page
table and add one entry for the first 4MB of the virtual address space that
maps to the first 4MB of the physical memory as well as an entry for
`0xC0100000` that maps to `0x0010000`. If we skip the identity mapping for the
first 4MB, the CPU would generate a page fault immediately after paging was
enabled when trying to fetch the next instruction from memory. When the table
is created, an indirect jump can be done to a label, like

~~~ {.nasm}
; assembly code executing at around 0x00100000
; enable paging for both actual location of kernel
; and its higher-half virtual location

lea ebx, [higher_half] ; load the address of the label in ebx
jmp ebx                ; jump to the label

higher_half:
    ; code here executes in the higher half kernel
    ; eip is larger than 0xC0000000
    ; can continue kernel initialisation, calling C code, etc.
~~~

Now `eip` will point to a memory location somewhere right after `0xC0100000` -
all the code can now execute as if it were at `0xC0100000`, the higher-half.
The entry mapping the first 4MB of virtual memory to the first 4MB of
physical can be removed from the page table, and its corresponding entry in
the TLB invalidated with `invlpg [0]`.

### Running in the higher half

There are a few details we must deal with when using a higher-half kernel. We
must now take care when using memory mapped I/O that uses very specific memory
locations. For example, the frame buffer is located at `0x000B8000`, but since
there is no entry in the page table for the address `0x000B8000` any longer,
the address `0xC00B8000` must be used, since the virtual address `0xC0000000`
maps to the physical address`0x00000000`.

Any explicit references to addresses within the multiboot structure needs to be
changed to reflect the new virtual addresses as well.

Mapping entire 4MB pages for the kernel is very simple, but wastes quite some
memory. Doing an higher-half kernel mapped in as 4KB pages is doable, but
somewhat trickier. The page directory and one page table can be in `.data`
(assuming your kernel is smaller than 4MB), but you need to configure the
mappings at runtime. The size of the kernel can be determined by exporting
labels from the linker script [@ldcmdlang], which we'll need to do later
anyway when writing the [page frame allocator](#how-much-memory-is-there).

## Virtual memory through paging

Paging does two things that is nice for virtual memory. First, it allows for
quite fine-grained access control to memory. You can mark pages as read-only,
read-write, only for PL0 etc. Second, it creates the illusion of contiguous
memory. User mode processes, and the kernel, can access memory as if it were
contiguous, and the contiguous memory can be extended without the need to move
stuff around. Also, we can allow the user mode programs to access all memory
below 3GB, but unless they actually use it, we don't have to assign page frames
to the pages. This allows processes to have code located near `0x00000000` and
the stack at just below `0xC0000000`, and not require more than two actual
pages.

## Further reading

- Chapter 4 (and to some extent chapter 3) of the Intel manual [@intel3a] are
  your definitive sources for the details about paging.
- Wikipedia has an article on paging: <http://en.wikipedia.org/wiki/Paging>
- The OSDev wiki has a page on paging: <http://wiki.osdev.org/Paging> and a
  tutorial for making a higher-half kernel:
  <http://wiki.osdev.org/Higher_Half_bare_bones>
- Gustavo Duarte's article on how a kernel manages memory is read-worthy:
  <http://duartes.org/gustavo/blog/post/anatomy-of-a-program-in-memory>
- Details on the linker command language can be found in [@ldcmdlang].
- More details on the ELF format can be found in this pdf:
  <http://flint.cs.yale.edu/cs422/doc/ELF_Format.pdf>
