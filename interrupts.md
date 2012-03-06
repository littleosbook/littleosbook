# Interrupts and Input
Now that the OS can produce _output_, it would be nice if it also could get some
_input_. In order to read information from the keyboard, the operating system
must be able to handle _interrupts_. An interrupt occurs when a hardware
device, such as the keyboard, the serial port or the timer, signals the CPU
that the state of the device has changed. The CPU itself can also send
interrupts due to program errors, for example when a program references memory
it doesn't has access to, or when a program divides a number by zero. Finally,
there are also _software intterupts_, which are interrupts that are caused by
the `int` assembly instruction, and they are often used for system calls
(syscalls).

## Interrupts Handlers
Interrupts are handled via the _interrupt descriptor table_ (IDT). The IDT
that describes a handler for each interrupt. Each interrupt has a number (0 -
255), and the handler for interrupt _i_ is defined at the _i:th_ position in
the table. There are three different kinds of handlers for interrupts:

- Task handler
- Interrupt handler
- Trap handler

The task handler uses Intel specific functionality, so they won't be covered
here (see the Intel manual [@intel3a], chapter 6, for more info). The only
difference between an interrupt handler and a trap handler is that the
interrupt handler disables interrupts, that is, you can't get an interrupt
while at the same time handling an interrupt. In this book, we will use trap
handlers and disables interrupts manually when we need to.

## Creating an Entry in the IDT
An entry in the IDT for an interrupt handler consists of 64 bits.
The highest 32 bits are

    Bit:     | 31              16 | 15 | 14 13 | 12 | 11 | 10 9 8 | 7 6 5 | 4 3 2 1 0 |
    Content: | offset high        | P  | DPL   | 0  | D  | 1  1 0 | 0 0 0 | reserved  |

and the lowest 32 bits are

    Bit:     | 31              16 | 15              0 |
    Content: | segment selector   | offset low        |

where the content is

             Name Description
----------------- ------------
      offset high The 16 highest bits of the 32 bit address in the segment
       offset low The 16 lowest bits of the 32 bits address in the segment
                p If the handler is present in memory or not (1 = present, 0 = not present)
              DPL Descriptor Privilige Level, the privilege level the handler can be called from (0, 1, 2, 3)
                D Size of gate, (1 = 32 bits, 0 = 16 bits)
 segment selector The index in the GDT
                r Reserved

The offset is a pointer to code (preferably an assembly label). For example, to
create an entry for a handler which code starts at `0xDEADBEEF` that runs in
privilege level 0 (same as the kernel) and therefore uses the same code segment
selector as the kernel, the following two bytes would be used:

    0xDEAD8E00
    0x0008BEEF

If the IDT then is represented an `unsigned integer idt[256];` then to register
this as the handler for interrupt 0 (divide error), you would write

~~~ {.c}
idt[0] = 0xDEAD8E00
idt[1] = 0x0008BEEF
~~~

As written in the chapter [getting to C](#getting-to-c), we recommend that you
instead of using bytes (or unsigned integers) use packed structures to make the
code more readable.

## Creating a Generic Interrupt Handler
When the interrupt occurs, the CPU will push information about the interrupt on
the stack, then look up the appropriate interrupt hander in the IDT and jump to
it. Once the interrupt handler is done, it uses the `iret` instruction to
return. `iret` pops the information from the stack, restores EFLAGS and jumps
to `CS:EIP`.

The stack at the time of the interrupt will look like

    [esp + 12] EFLAGS
    [esp + 8]  CS
    [esp + 4]  EIP
    [esp]      Error code?

The reason for the question mark behind error code is because not all
interrupts creates an error code. The CPU interrupts that put an error code on
the stack are 8, 10, 11, 12, 13, 14 and 17.

The interrupt handler has to be written in assembly, because the all the
registers has to be pushed on the stack, because the code that got interrupted
will assume that the state of its registers hasn't changed. Writing all the
logic of the interrupt handler in assembly will be tiresome, so creating an
assembly handler that saves the registers, calls a C function, restores the
registers and returns is a good idea!

The C handler should get the state of the registers, the state of the stack and
the number of the interrupt. The following definitions can for example be used:

~~~ {.c}
struct cpu_state {
    unsigned int eax;
    unsigned int ebx;
    unsigned int ecx;
    .
    .
    .
    unsigned int esp;
} __attribute__((packed));

struct stack_state {
    unsigned int error_code;
    unsigned int eip;
    unsigned int cs;
    unsigned int eflags;
} __attribute__((packed));

void interrupt_handler(struct cpu_state cpu, unsigned int interrupt, struct stack_state stack);
~~~

Unfortunately, the CPU does _not_ push the interrupt number on the stack. Since
writing one version for each interrupt is tedious, it's better to use the macro
functionality of NASM (see [@nasm:macros] for more details). Since not all
interrupts produce an error code, the error code 0 will be used for interrupts
without error code.

~~~ {.nasm}
%macro no_error_code_interrupt_handler %1
global interrupt_handler_%1
interrupt_handler_%1:
    push    dword 0                     ; push 0 as error code
    push    dword %1                    ; push the interrupt number
    jmp     common_interrupt_handler    ; jump to the common handler
%endmacro

%macro error_code_interrupt_handler %1
global interrupt_handler_%1
interrupt_handler_%1:
    push    dword %1                    ; push the interrupt number
    jmp     common_interrupt_handler    ; jump to the common handler
%endmacro

common_interrupt_handler:               ; the common parts of the generic interrupt handler
    ; save the registers
    push    eax
    push    ebx
    .
    .
    .
    push    ebp

    ; call the C function
    call    interrupt_handler

    ; restore the registers
    pop     ebp
    .
    .
    .
    pop     ebx
    pop     eax

    ; restore the esp
    add     esp, 8

    ; return to the code that got interrupted
    iret

no_error_code_interrupt_handler 0       ; create handler for interrupt 0
no_error_code_interrupt_handler 1       ; create handler for interrupt 1
.
.
.
error_code_handler              7       ; create handler for interrupt 7
.
.
.
~~~

The `common_interrupt_handler` does the following:
- Push the registers onto the stack
- Call the C function `interrupt_handler`
- Pop the registers from the stack
- Add 8 to `esp` (due to the error code and the interrupt number)
- Execute `iret` to return to the interrupted code

Since the macros declares global labels, the addresses of the interrupt
handlers can be accessed from C or assembly when creating the IDT.

## Loading the IDT
The IDT is loaded with `lidt` assembly instruction which takes the address of
the first element in the array. It is easiest to wrap this instruction and
instead use it from C:

~~~ {.nasm}
global  load_idt

; load_idt - Loads the interrupt descriptor table (IDT).
; stack: [esp + 4] the address of the first entry in the IDT
;        [esp    ] the return address
load_idt:
    mov     eax, [esp+4]    ; load the address of the IDT into register eax
    lidt    eax             ; load the IDT
    ret                     ; return to the calling function
~~~

## Programmable Interrupt Controller (PIC)
To start using hardware interrupts, you must first configure the Programmable
Interrupt Controller (PIC). The PIC makes it possible to map signals from the
hardware to interrupts. The reasons for configuring the PIC are:

- Remap the interrupts. By default, the PIC uses interrupts 0 - 15 for hardware
  interrupts, which conflicts with the CPU interrupts, so the PIC interrupts
  must be remapped to another interval.
- Select which interrupts to receive. You probably don't want to receive
  interrupts from all hardware, since you don't have code that manages these
  interrupts anyway.
- Set up the correct mode for the PIC.

In the beginning, there was only one PIC (PIC 1) and eight interrupts. As more
hardware got added in the computer, 8 interrupts were too few. The solution was
then to chain on another PIC (PIC 2) on the first PIC (see interrupt 2 on PIC
1).

The hardware interrupts are:

 PIC 1 Hardware     PIC 2 Hardware
------ ---------   ------ ---------
     0 Timer            8 Real Time Clock
     1 Keyboard         9 General I/O
     2 PIC 2           10 General I/O
     3 COM 2           11 General I/O
     4 COM 1           12 General I/O
     5 LPT 2           13 Coprocessor
     6 Floppy disk     14 IDE Bus
     7 LPT 1           15 IDE Bus

A great tutorial for configuring the PIC can be found at [@acm].

Every interrupt from the PIC has to be acknowledged, that is, sending a message
to the PIC confirming that the interrupt has been handled. If this isn't done,
the PIC won't generate any more interrupts.

This is done by sending the byte `0x20` to the PIC that raised the interrupt.
Sending the acknowledgement to the PIC that did _not_ raise the interrupt
doesn't do anything. Therefore, implementing a `pic_acknowledge` function is
straightforward:

~~~ {.c}

#include "io.h"

#define PIC1_PORT_A 0x20
#define PIC2_PORT_A 0xA0

#define PIC_ACK     0x20

/** pic_acknowledge:
 *  Acknowledges an interrupt from either PIC 1 or PIC 2.
 */
void pic_acknowledge(void)
{
    outb(PIC1_PORT_A, PIC_ACK);
    outb(PIC2_PORT_A, PIC_ACK);
}
~~~

## Reading Input from the Keyboard

The keyboard generates interrupt 1 from the PIC, so if you remapped the PIC's
interrupts to start at number 32, the interrupt number for the keyboard will be
33.

The keyboard does not generate ASCII characters, instead it generates scan
codes. A scan code represents a button. The scan code representing the just
pressed button can be read from the keyboard's data I/O port:

~~~ {.c}
#include "io.h"

#define KBD_DATA_PORT   0x60

/** read_scan_code:
 *  Reads a scan code from the keyboard
 *
 *  @return The scan code (NOT an ASCII character!)
 */
unsigned char read_scan_code(void)
{
    return inb(KBD_DATA_PORT);
}
~~~

The next step is to write a function that translates a scan code to the
corresponding ASCII character. If you want to map the scan codes to the ASCII
characters as is done on an American keyboard, [@scancodes] shows how the scan
codes and keyboard buttons correlates.

Remember, since the keyboard interrupt is raised by the PIC, you must call
`pic_acknowledge` at the end of the keyboard interrupt handler!

## Further Reading

- The OSDev wiki has a great page on interrupts,
  <http://wiki.osdev.org/Interrupts>
- Chapter 6 of Intel Manual 3a [@intel3a] describes everything there is to know
  about interrupts.
