# Virtual Memory, an Introduction

Virtual memory is an abstraction of physical memory. The purpose of virtual
memory is generally to simplify application development and to let processes
address more memory than what is actually physically present. We don't
want applications messing with the kernel or other applications' memory.

In the x86 architecture, virtual memory can be accomplished in two ways:
Segmentation and paging. Paging is by far the most common and versatile
technique, and we'll implement it in chapter 7. Some use of segmentation
is still necessary (to allow for code to execute under different privilege
levels), so we'll set up a minimal segmentation structure in the next chapter.

Managing memory is a big part of what an operating system does.
[Paging](#paging) and [page frame allocation](#page-frame-allocation) deals
with that.

Segmentation and paging is described in the Intel manual [@intel3a], chapter 3
and 4.

## Virtual Memory Through Segmentation?

You could skip paging entirely and just use segmentation for virtual memory.
Each user mode process would get its own segment, with base address and limit
properly set up so that no process can see someone else's memory. A problem
with this is that all memory for a process needs to be contiguous. Either we
need to know in advance how much memory the program will require (unlikely), or
we can move the memory segments to places where they can grow when the limit is
reached (expensive, causes fragmentation - can result in "out of memory" even
though enough memory is available, but in too small chunks). Paging solves both
these problems.

It might be interesting to note that in x86\_64 (another CPU architecture),
segmentation is almost completely removed.

## Further Reading

- LWN.net has an article on virtual memory: <http://lwn.net/Articles/253361/>
- So does Gustavo Duarte:
  <http://duartes.org/gustavo/blog/post/memory-translation-and-segmentation>
