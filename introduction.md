# Introduction

This text is a practical guide to writing your own x86 operating system. It is
designed to give enough help with the technical details while at the same time not
reveal too much with samples and code excerpts. We've tried to collect
parts of the vast (and often excellent) expanse of material and tutorials
available, on the web and otherwise, and add our own insights into the problems
we encountered and struggled with.

This book is not about the theory behind operating systems, or how any specific
operating system (OS) works. For OS theory we recommend the book _Modern
Operating Systems_ by Andrew Tanenbaum [@ostanenbaum]. Lists and details on
current operating systems are available on the Internet.

The starting chapters are quite detailed and explicit, to quickly get you into
coding. Later chapters give more of an outline of what is needed, as more and
more of the implementation and design becomes up to the reader, who should now
be more familiar with the world of kernel development. At the end of some
chapters there are links for further reading, which might be interesting and
give a deeper understanding of the topics covered.

In [chapter 2](#first-steps) and [3](#getting-to-c) we set up our development
environment and boot up our OS kernel in a virtual machine, eventually starting
to write code in C. We continue in [chapter 4](#output) with writing to the
screen and the serial port, and then we dive into segmentation in [chapter
5](#segmentation) and interrupts and input in [chapter
6](#interrupts-and-input).

After this we have a quite functional but bare-bones OS kernel. In [chapter
7](#the-road-to-user-mode) we start the road to user mode applications, with
virtual memory through paging ([chapter
8](#a-short-introduction-to-virtual-memory) and [9](#paging)), memory
allocation ([chapter 10](#page-frame-allocation)), and finally running a user
application in [chapter 11](#user-mode).

In the last three chapters we discuss the more advanced topics of file systems
([chapter 12](#file-systems)), system calls ([chapter 13](#system-calls)), and
multitasking ([chapter 14](#multitasking)).

## About the Book

The OS kernel and this book were produced as part of an advanced individual
course at the Royal Institute of Technology [@kth], Stockholm.  The authors had
previously taken courses in OS theory, but had only minor practical experience
with OS kernel development.  In order to get more insight and a deeper
understanding of how the theory from the previous OS courses works out in
practice, the authors decided to create a new course, which focused on the
development of a small OS. Another goal of the course was writing a thorough
tutorial on how to develop a small OS basically from scratch, and this short
book is the result.

The x86 architecture is, and has been for a long time, one of the most common
hardware architectures. It was not a difficult choice to use the x86
architecture as the target of the OS, with its large community, extensive
reference material and mature emulators. The documentation and information
surrounding the details of the hardware we had to work with was not always easy
to find or understand, despite (or perhaps due to) the age of the architecture.

The OS was developed in about six weeks of full-time work. The implementation
was done in many small steps, and after each step the OS was tested manually.
By developing in this incremental and iterative way, it was often easier to
find any bugs that were introduced, since only a small part of the code had
changed since the last known good state of the code. We encourage the reader to
work in a similar way.

During the six weeks of development, almost every single line of code was
written by the authors together (this way of working is also called
_pair-programming_). It is our belief that we managed to avoid a lot of bugs
due to this style of development, but this is hard to prove scientifically.

## The Reader

The reader of this book should be comfortable with UNIX/Linux, systems
programming, the C language and computer systems in general (such as
hexadecimal notation [@wiki:hex]). This book could be a way to get started
learning those things, but it will be more difficult, and developing an
operating system is already challenging on its own. Search engines and other
tutorials are often helpful if you get stuck.

## Credits, Thanks and Acknowledgements

We'd like to thank the OSDev community [@osdev] for their great wiki and
helpful members, and James Malloy for his eminent kernel development tutorial
[@malloy]. We'd also like to thank our supervisor Torbjörn Granlund for his
insightful questions and interesting discussions.

Most of the CSS formatting of the book is based on the work by Scott Chacon for
the book Pro Git, <http://progit.org/>.

## Contributors
We are very grateful for the patches that people send us. The following users
have all contributed to this book:

- [alexschneider](https://github.com/alexschneider)
- [Avidanborisov](https://github.com/Avidanborisov)
- [nirs](https://github.com/nirs)
- [kedarmhaswade](https://github.com/kedarmhaswade)
- [vamanea](https://github.com/vamanea)
- [ansjob](https://github.com/ansjob)

## Changes and Corrections

This book is hosted on Github - if you have any suggestions, comments or
corrections, just fork the book, write your changes, and send us a pull
request. We'll happily incorporate anything that makes this book better.

## Issues and where to get help
If you run into problems while reading the book, please check the issues on
Github for help: <https://github.com/littleosbook/littleosbook/issues>.

## License

All content is under the Creative Commons Attribution Non Commercial Share
Alike 3.0 license, <http://creativecommons.org/licenses/by-nc-sa/3.0/us/>. The
code samples are in the public domain - use them however you want. References
to this book are always received with warmth.
