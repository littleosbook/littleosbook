# File Systems

We are not required to have file systems in our operating system, but it is
quite convenient, and it often plays a central part in many operations of
several existing OS's, especially UNIX-like systems. Before we start the
process of supporting multiple processes and system calls, we might want to
consider implementing a simple file system.

## Why a File System?

How do we specify what programs to run in our OS? Which is the first program to
run? How do programs output data? Read input?

In UNIX-like systems, with their almost-everything-is-a-file convention, these
problems are solved by the file systems. It might also be interesting to read a
bit about the Plan 9 project, which takes this idea one step further (See
[further reading](#further-reading-6) below).

## A Simple File System

The most simple file system possible might be what we already have - one
"file", existing only in RAM, loaded by GRUB before the kernel starts. When our
kernel and operating system grows, this is probably too limiting.

A next step could be to add structure to this GRUB-loaded module. With a
utility program - perhaps called `mkfs` - which we run at build time, we can
create our file system in this file.

`mkfs` can traverse a directory on our host system and add all subdirectories
and files as part of our target file system. Each object in the file system
(directory or file) can consist of a header and a body, where the body of a
file is the actual file and the body of a directory is a list of entries -
names and "addresses" of other files and directories.

Each object in this file system will become contiguous, so they will be easy to
read from our kernel. All objects will also have a fixed size (except for the
last one, which can grow); it might be difficult to add new or modify existing
files. We can make the file system read-only.

`mmap` is a handy system call that makes writing the "file system-in-a-file"
easier.

## Inodes and Writable File Systems

When we decide that we want a more complex - and realistic - file system, we
might want to look into the concept of inodes. See [further
reading](#further-reading-6).

## A Virtual File System and devfs

What abstraction should we use for reading and writing to devices such as the
screen and the keyboard?

With a virtual file system (VFS) we create an abstraction on top of any real
file systems we might have. The VFS mainly supplies the path system and file
hierarchy, and delegates operations on files to the underlying file
systems. The original paper on VFS is succinct, concrete, and well worth a
read. See [further reading](#further-reading-6).

With a VFS we can mount a special file system on `/dev`, which handles all
devices such as keyboards and the screen. Or we can take the traditional UNIX
approach, with major/minor device numbers and `mknod` to create special files
for our devices.

## Further Reading

- The ideas behind the Plan 9 operating systems is worth taking a look at:
  <http://plan9.bell-labs.com/plan9/index.html>
- Wikipedia's page on inodes: <http://en.wikipedia.org/wiki/Inode> and the
  inode pointer structure:
  <http://en.wikipedia.org/wiki/Inode_pointer_structure>.
- The original paper on the concept of vnodes and a virtual file system is
  quite interesting:
  <http://www.arl.wustl.edu/~fredk/Courses/cs523/fall01/Papers/kleiman86vnodes.pdf>
- Poul-Henning Kamp discusses the idea of a special file system for `/dev` in
  <http://static.usenix.org/publications/library/proceedings/bsdcon02/full_papers/kamp/kamp_html/index.html>
