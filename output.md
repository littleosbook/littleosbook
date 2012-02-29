# Output

After booting our very small operating system kernel, the next task will be to
be able to display text on the console. In this chapter we will create our
first _drivers_, that is, code that acts as an layer between our kernel and the
hardware, providing a nicer abstraction than speaking directly to the hardware.
The first part of this chapter creates a driver for the framebuffer
[@wiki:fb] to be able to display text to the user. The second part will
create a driver for the serial port, since BOCHS can store output from the
serial port in a file, effectively creating a logging mechanism for our
operating system!

## Different kinds of outputs
There are usually two different ways to "talk" to the hardware,
_memory-mapped I/O_ and _I/O ports_.

If the hardware uses memory-mapped I/O, then one
can write to a specific memory address and the hardware will be automatically
get the new values. One example of this is the framebuffer, which will be
discussed more in detail later. If you write the ASCII [@wiki:ascii] the value
`0x410F` to address `0x000B8000`, you will see the letter A in white color on a
black background (see the section [The framebuffer](#the-framebuffer) for more
details).

If the hardware uses I/O ports, then the assembly instructions `outb` and `inb`
must be used to communicate with the device. `out` takes two parameters, the
address of the device and the value to send to the device. `in` takes a single
parameter, simply address of the device. One can think of I/O ports as
communicating with the hardware the same as one communicate with a server using
sockets. For example, the cursor (the blinking rectangle) of the framebuffer is
controlled via I/O ports.

## The framebuffer
The framebuffer is a hardware device that is capable of displaying a buffer of
memory on the screen [@wiki:fb]. The framebuffer has 80 columns and 25 rows,
and their indices start at 0 (so rows are labelled 0 - 24).

### Writing text
Writing text to the console via the framebuffer is done by
memory-mapped I/O. The starting address of memory-mapped I/O for the
framebuffer is `0x000B8000`. The memory is divided into 16 bit cells, where the
16 bit determines both the character, the foreground color and the background
color. The highest 8 bits determines the character, bit 7 - 3 the background
and bit 3 - 0 the foreground, as can be seen in the following figure:

    Bit:     | 15 14 13 12 11 10 9 8 | 7 6 5 4 | 3 2 1 0 |
    Content: | ASCII                 | FG      | BG      |

The available colors are:

 Color Value       Color Value         Color Value          Color Value
------ ------ ---------- ------- ----------- ------ ------------- ------
 Black 0             Red 4         Dark grey 8          Light red 12
  Blue 1         Magenta 5        Light blue 9      Light magenta 13
 Green 2           Brown 6       Light green 10       Light brown 14
  Cyan 3      Light grey 7        Light cyan 11             White 15

The first cell corresponds to row zero, column zero on the console.  Using the
ASCII table, one can see that A corresponds to 65 or `0x41`. Therefore, to
write the character A with a green foreground (2) and dark grey background (8),
the following assembly instruction is used

~~~ {.nasm}
mov [0x000B8000], 0x4128
~~~

The second cell then corresponds to row zero, column one and it's address is

    0x000B8000 + 16 = 0x000B8010

This can all be done a lot easier in C by treating the address `0x000B8000` as
a char pointer, `char *fb = (char *) 0x000B8000`. Then, writing A to at place
(0,0) with green foreground and dark grey background becomes:

~~~ {.nasm}
fb[0] = 'A';
fb[1] = 0x28;
~~~

This can of course be wrapped into a nice function

~~~ {.c}
void fb_write_cell(unsigned int i, char c, unsigned char fg, unsigned char bg)
{
    fb[i] = c;
    fb[i + 1] = ((fg & 0x0F) << 4) | (bg & 0x0F)
}
~~~

which can then be called

~~~ {.c}
#define FB_GREEN     2
#define FB_DARK_GREY 8

fb_write_cell(0, 'A', FB_GREEN, FB_DARK_GREY);
~~~

### Moving the cursor

Moving the cursor of the framebuffer is done via two different I/O ports.
The cursor position is determined via a 16 bits integer, 0 means row zero,
column zero, 1 means row zero, column one, 80 means row one, column zero and so
on. Since the position is 16 bits large, and the `out` assembly instruction
only take 8 bits as data, the position must be sent in two turns, first 8 bits
then the next 8 bits. The has two I/O ports, one for accepting the data, and
one for describing the data being received. Port `0x3D4` is the command port
that describes the data and port `0x3D5` is for the data.

To set the cursor at row one, column zero (position `80 = 0x0050`), one would
use the following assembly instructions

~~~ {.nasm}
out 0x3D4, 14      ; 14 tells the framebuffer to expect the highest 8 bits
out 0x3D5, 0x00    ; sending the highest 8 bits of 0x0050
out 0x3D4, 15      ; 15 tells the framebuffer to expect the lowest 8 bits
out 0x3D5, 0x50    ; sending the lowest 8 bits of 0x0050
~~~

The `out` assembly instruction can't be done in C, therefore it's a good idea
to wrap it an assembly function which can be accessed from C via the cdecl
calling standard:

~~~ {.nasm}
global outb             ; make the label outb visible outside this file

; outb - send a byte to (out byte) to an I/O port
; stack: [esp + 8] the data byte
;        [esp + 4] the I/O port
;        [esp    ] return address
outb:
    mov al, [esp + 8]    ; move the data to be sent into the al register
    mov dx, [esp + 4]    ; move the address of the I/O port into the dx register
    out dx, al           ; send the data to the I/O port
    ret                  ; return to the calling function
~~~

By storing this function in a file called `io.s` and also creating a header
`io.h`, the `out` assembly instruction can now be conveniently accessed from C:

~~~ {.c}
#ifndef INCLUDE_IO_H
#define INCLUDE_IO_H

/** outb:
 *  Sends the given data to the given I/O port. Defined in io.s
 *
 *  @param port The I/O port to send the data to
 *  @param data The data to send to the I/O port
 */
void outb(unsigned short port, unsigned char data);

#endif /* INCLUDE_IO_H */
~~~

Moving the cursor can now be wrapped in a C function:

~~~ {.c}
#include "io.h"

/* The I/O ports */
#define FB_COMMAND_PORT         0x3D4
#define FB_DATA_PORT            0x3D5

/* The I/O port commands */
#define FB_HIGH_BYTE_COMMAND    14
#define FB_LOW_BYTE_COMMAND     15

void fb_move_cursor(unsigned short pos)
{
    outb(FB_COMMAND_PORT, FB_HIGH_BYTE_COMMAND);
    outb(FB_DATA_PORT,    ((pos >> 8) & 0x00FF));
    outb(FB_COMMAND_PORT, FB_LOW_BYTE_COMMAND);
    outb(FB_DATA_PORT,    pos & 0x00FF);
}
~~~

### The driver
Now that the basic functions, it's time to think about a driver interface for
the framebuffer. There is no right or wrong about what functionality the
interface should provide, but one suggestion is to have a `write` function with
the declaration:

    int write(char *buf, unsigned int len);

The `write` function would automatically advance the cursor after a character
has been written and also scroll the screen if necessary.

## The serial ports
The serial port [@wiki:serial] is an interface for communicating between
hardware devices, and it's usually not found on computer any longer. However,
the serial port is easy to use, and more importantly, can be used as a logging
utility together with BOCHS. A computer usually has several serial ports, but
we will only make use of one of the ports, since we only will use it for
logging. The serial ports are controlled completely via I/O ports.

### Configuring the serial port
The first data that needs to be sent to the serial port is some configuration
data. In order for two hardware devices to be able to talk each other, they
must agree upon some things. These things include:

- The speed used when sending data (bit or baud rate)
- If any error checking is used for the data (parity bit, stop bits)
- The number of bits that represents a unit of data (data bits)

### Configuring the line
Configuring the line means to configure how data is being sent over the line.
The serial port has an I/O port, the _line command port_ that is used for
configuring the line.

First, the speed for sending data will be set. The serial port has an internal
clock that runs at 115200 Hz. Setting the speed means sending a divisor to the
serial port, for example sending 2 results in a speed of `115200 / 2 = 57600`
Hz.

The divisor is a 16 bit number, but we can only send 8 bits at a time.
Therefore, we must first send an instruction telling the serial port to expect
first the highest 8 bits, then the lowest. This is done by sending `0x80` to
the send line command port. The code then becomes:

~~~ {.c}
#include "io.h" /* io.h is implement in the section "Moving the cursor" */

/* The I/O ports */

/* All the I/O ports are calculated relative to the data port. This is because
 * all serial ports (COM1, COM2, COM3, COM4) have their ports in the same
 * order, but they start at different values.
 /

#define SERIAL_COM1_BASE                0x3F8      /* COM1 base port */

#define SERIAL_DATA_PORT(base)          (base)
#define SERIAL_FIFO_COMMAND_PORT(base)  (base + 2)
#define SERIAL_LINE_COMMAND_PORT(base)  (base + 3)
#define SERIAL_MODEM_COMMAND_PORT(base) (base + 4)
#define SERIAL_LINE_STATUS_PORT(base)   (base + 5)

/* The I/O port commands */

/* SERIAL_LINE_ENABLE_DLAB:
 * Tells the serial port to expect first the highest 8 bits on the data port,
 * then the lowest 8 bits will follow
 */
#define SERIAL_LINE_ENABLE_DLAB         0x80

/** serial_configure_baud_rate:
 *  Sets the speed of the data being sent. The default speed of a serial
 *  port is 115200 bits/s. The argument is a divisor of that number, hence
 *  the resulting speed becomes (115200 / divisor) bits/s.
 *
 *  @param com      The COM port to configure
 *  @param divisor  The divisor
 */
void serial_configure_baud_rate(unsigned short com, unsigned short divisor)
{
    outb(SERIAL_LINE_COMMAND_PORT(com),
         SERIAL_LINE_ENABLE_DLAB);
    outb(SERIAL_DATA_PORT(com),
         (divisor >> 8) & 0x00FF);
    outb(SERIAL_DATA_PORT(com),
         divisor & 0x00FF);
}
~~~

Next, the way data is being sent must be configured, this is also done via the
line command port by sending 8 bits. The layout of the 8 bits are as follows

    Bit:     | 7 | 6 | 5 4 3 | 2 | 1 0 |
    Content: | d | b | prty  | s | exp |

The content is

 Name Description
----- ------------
    d Enables (`d = 1`) or disables (`d = 0`) DLAB
    b If break control is enabled (`b = 1`) or disabled (`b = 0`)
 prty The number of parity bits to use (`prty = 0, 1, 2 or 3`)
    s The number of stop bits to use (`s = 0` equals 1, `s = 1` equals 1.5 or 2)
  exp Describes the length of the data, the length is `2^exp` (`exp = 3` results in 8 bits)

We will use the mostly standard value `0x03` [@osdev:serial], meaning a length
of 8 bits, no parity bit, one stop bit and break control disabled. This is sent
to the line command port, resulting in

~~~ {.c}
/** serial_configure_line:
 *  Configures the line of the given serial port. The port is set to have a
 *  data length of 8 bits, no parity bits, one stop bit and break control
 *  disabled.
 *
 *  @param com  The serial port to configure
 */
void serial_configure_line(unsigned short com)
{
    /* Bit:     | 7 | 6 | 5 4 3 | 2 | 1 0 |
     * Content: | d | b | prty  | s | exp |
     * Value:   | 0 | 0 | 0 0 0 | 0 | 1 1 | = 0x03
     */
    outb(SERIAL_LINE_COMMAND_PORT(com), 0x03);
}
~~~

For a more in-depth explanation of the values, see [@osdev:serial].

### Configuring the FIFO queues
When data is transmitted via the serial port, it is placed in buffers, both
when receiving and sending data. This way, if you sending data to the serial
port faster than it can send it over the wire, it will be buffered. However, if
you send too much data too fast, the buffer will be full and then data will be
dropped. The FIFO queue configuration byte looks like

    Bit:     | 7 6 | 5  | 4 | 3   | 2   | 1   | 0 |
    Content: | lvl | 64 | r | dma | clt | clr | e |

The content is

 Name Description
----- ------------
  lvl How many bytes should be stored in the FIFO buffers
   64 If the buffers should 16 or 64 bytes large
    r Reserved
  dma How the serial port data should be accessed
  clt Clear the transmission FIFO buffer
  clr Clear the receiver FIFO buffer
    e If the FIFO buffer should be enabled or not

We use the value `0xC7 = 11000111` that:

    - Enables FIFO
    - Clear both receiver and transmission FIFO
    - Use 14 bytes as FIFO size

For a more in-depth explanation of the values, see [@wikibook:serial]

### Configuring the modem
The modem control register is used for very simple hardware flow control via
the Ready To Transmit (RTS) and Data Terminal Ready (DTR) pins. When
configuring the serial port before sending data, we want RTS and DTR to be 1.

The modem configuration byte looks like

    Bit:     | 7 | 6 | 5  | 4  | 3   | 2   | 1   | 0   |
    Content: | r | r | af | lb | ao2 | ao1 | rts | dtr |

The contents is

 Name Description
----- ------------
    r Reserved
   af Autoflow control enabled
   lb Loopback mode (used for debugging serial ports)
  ao2 Auxiliary output 2, used for receiving interrupts
  ao1 Auxiliary output 1
  rts Ready To Transmit
  dtr Data Terminal Ready

Since we don't need interrupts, since we won't handle the received data, we use 
the configuration value `0x03 = 00000011`, that is, RTS is set to 1 and DTR is
set to 1.

### Writing to the serial port

Writing to the serial port is done via the data I/O port. However, before
writing, we must first ensure that the transmit FIFO queue is empty (so that
all previous writes has been done). This is done by checking bit 5 of the line
status I/O port.

Reading the contents of an I/O port is done via the `int` assembly
instruction. There is no way to use the `int` instruction from C, so it has to
be wrapped (the same way as the `out` instruction):

~~~ {.nasm}
global inb

; inb - reads a byte from the given I/O port
; stack: [esp + 4] The address of the I/O port
;        [esp    ] The return address
inb:
    mov dx, [esp + 4]       ; move the address of the I/O port to the dx register
    in  al, dx              ; read a byte from the I/O port and store it in the al register
    ret                     ; return the read byte
~~~

~~~ {.c}
/* in file io.h */

/** inb:
 *  Read a byte from an I/O port.
 *
 *  @param port The address of the I/O port
 *  @return The read byte
 */
unsigned char inb(unsigned short port);
~~~

Checking if the transmit FIFO is empty is now trivial

~~~ {.c}
#include "io.h"

/** serial_is_transmit_fifo_empty:
 *  Checks whether the transmit FIFO queue is empty or not for the given COM
 *  port.
 *
 *  @param com The COM port
 *  @return 0 if the transmit FIFO queue is not empty
 *          1 if the transmit FIFO is empty
 */
int serial_is_transmit_fifo_empty(unsigned int com)
{
    /* 0x20 = 0001 0000 */
    return inb(SERIAL_LINE_STATUS_PORT(com)) & 0x20;
}
~~~

Now, the writing to serial port means spinning while the transmit FIFO queue
isn't empty, and then writing to the data port.

### The driver

We recommend that you try to write a `printf` like function, see section 7.3 in
[@knr]. We also recommend that you create some way of distinguish the
severeness of log message, for example by prepending the messages with `DEBUG`,
`INFO` or `ERROR`.

## Further reading
- The book "Serial programming" (available on WikiBooks) has a great section on
  programming the serial port,
  <http://en.wikibooks.org/wiki/Serial_Programming/8250_UART_Programming#UART_Registers>
- The OSDev wiki has a page with a lot of information about the serial ports,
  <http://wiki.osdev.org/Serial_ports>
