# Segmentation

Segmentation in x86 means accessing the memory through segments. Segments are
portions of the address space, possibly overlapping, specified by a base
address and a limit. To address a byte in segmented memory, you use a 48-bit
*logical address*: 16 bits that specifies the segment, and 32-bits to specify
what offset within that segment you want. The offset is added to the base
address of the segment, and the resulting linear address is checked against the
segment's limit - see the figure below. If everything checks out fine,
(including access-right-checks ignored for now), this results in a *linear
address*. If paging is disabled, this linear address space is mapped 1:1 on
the *physical address* space, and the physical memory can be accessed.

![Figure: Translation of logical addresses to linear addresses.
](images/intel_3_5_logical_to_linear.png)

To enable segmentation you need to set up a table that describes each segment -
a segment descriptor table. In x86, there are two types of descriptor tables:
The global descriptor table (GDT) and local descriptor tables (LDT). An LDT is
set up and managed by user-space processes, and each process have their own LDT.
LDT's can be used if a more complex segmentation model is desired - we won't
use it. The GDT shared by everyone - it's global.

## Accessing memory

Most of the time when we access memory we don't have to explicitly specify the
segment we want to use. The processor has six 16-bit segment registers: `cs`,
`ss`, `ds`, `es`, `gs` and  `fs`. `cs` is the code segment register and
specifies the segment to use when fetching instructions. `ss` is used whenever
the accessing the stack (through the stack pointer `esp`), and `ds` is used for
other data accesses. `es`, `gs` and `fs` are free to use however we or the user
processes wish (we will set them, but not use them).

Implicit use of segment registers:

~~~ {.nasm}
func:
    mov eax, [esp+4]
    mov ebx, [eax]
    add ebx, 8
    mov [eax], ebx
    ret
~~~

Explicit version:

~~~ {.nasm}
func:
    mov eax, [ss:esp+4]
    mov ebx, [ds:eax]
    add ebx, 8
    mov [ds:eax], ebx
    ret
~~~

(You don't need to use `ss` when accessing the stack, or `ds` when accessing
other memory. But it is convenient, and makes it possible to use the implicit
style above.)

Segment descriptors and their fields are described in figure 3-8 in the Intel
manual [@intel3a].

## The Global Descriptor Table (GDT)

A GDT/LDT is an array of 8-byte segment descriptors. The first descriptor in
the GDT is always a null descriptor, and can never be used to access memory. We
need at least two segment descriptors (plus the null descriptor) for our GDT
(and two more later when we enter [user mode](#user-mode), see the [segments
for user mode](#segments-for-user-mode) section). This is because the
descriptor contains more information than just the base and limit fields. The
two most relevant fields here are the Type field and the Descriptor Privilege
Level (DPL) field.

The table 3-1, chapter 3, in the Intel manual [@intel3a] specifies the values
for the Type field, and it is because of this that we need at least two
descriptors: One to execute code (to put in `cs`) (Execute-only or
Execute-Read) and one to read and write data (Read/Write) (to put in the other
segment registers).

The DPL specifies the privilege levels required to execute in this segment. x86
allows for four privilege levels (PL), 0 to 3, where PL0 is the most
privileged. In most operating systems (eg. Linux and Windows), only PL0 and PL3
are used (although MINIX uses all levels). The kernel should be able to do
anything, so it "runs in PL0" (uses segments with DPL set to 0), and user-mode
processes should run in PL3. The current privilege level (CPL) is determined by
the segment selector in `cs`.

Since we are now executing in kernel mode (PL0), the DPL should be 0. The
segments we need are described in the table below.

  Index   Offset   Name                 Address range             Type   DPL
-------  -------   -------------------  ------------------------- -----  ----
      0   `0x00`   null descriptor
      1   `0x08`   kernel code segment  `0x00000000 - 0xFFFFFFFF` RX     PL0
      2   `0x10`   kernel data segment  `0x00000000 - 0xFFFFFFFF` RW     PL0

Table: The segment descriptors needed.

Note that the segments overlap - they both encompass the entire linear address
space. In our minimal setup we'll only use segmentation to get privilege levels.
See the [@intel3a], chapter 3, for details on the other descriptor fields.

## Creating and loading the GDT

Creating the GDT can easily be done both in C and assembler. A static
fixed-size array should do the trick.

To load the GDT into the processor we use the `lgdt` instruction, which takes
the address of a struct that specifies the start and size of the GDT:

    32-bit: start of GDT
    16-bit: size of GDT (8 bytes * num entries)

We have three entries.

If `eax` has an address to such a struct, we can just to the following:

~~~ {.nasm}
lgdt [eax]
~~~

Now that the processor knows where to look for the segment descriptors we need
to load the segment registers with the corresponding segment selectors. The
content of a segment selector is described in the table below.

-------------------------------------------------------------------------------
  Bits Name             Description
------ ---------------- -------------------------------------------------------
   0-1 RPL              Requested Privilege Level - we want to execute in PL0
                        for now.

     2 Table Indicator  0 means that this specifies a GDT segment, 1 means an
                        LDT Segment.

  3-15 Offset (Index)   Offset within descriptor table.
-------------------------------------------------------------------------------

Table: The layout of segment selectors.

The offset is added to the start of the GDT to get the address of the segment
descriptor: `0x08` for the first descriptor and `0x10` for the second, since
each descriptor is 8 bytes. The Requested Privilege Level (RPL) should be `0`,
since we want to remain in PL0.

Loading the segment selector registers is easy for the data registers - just
copy the correct offsets into the registers:

~~~ {.nasm}
mov ds, 0x10
mov ss, 0x10
mov es, 0x10
; ...
~~~

To load `cs` we have to do a "far jump":

~~~ {.nasm}
    ; using previous cs
    jmp 0x08:flush_cs

flush_cs:
    ; now we've changed cs to 0x08
~~~

A far jump is a jump where we explicitly specify the full 48-bit logical
address: The segment selector to use, and the absolute address to jump to. It
will first set `cs` to `0x08`, and then jump to `.flush_cs` using its absolute
address.

Whenever we load a new segment selector into a segment register, the processor
reads the entire descriptor and stores it in shadow registers within the
processor.

## Further reading

- Chapter 3 of the Intel manual [@intel3a] is quite good; low-level and
  technical.
- The OSDev wiki can be useful: <http://wiki.osdev.org/Segmentation>
- The Wikipedia page on x86 segmentation might be worth looking into:
  <http://en.wikipedia.org/wiki/X86_memory_segmentation>
