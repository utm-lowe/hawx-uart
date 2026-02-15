# HAWX Assignment: UART Driver with Ports

## Introduction

In this assignment, you will implement the UART driver for the HAWX
kernel. This is our first true *hardware* subsystem: code that interacts
with an external device (external to the CPU/RAM bus) using
**memory-mapped I/O**.

In HAWX, device drivers do not talk directly to user programs. Instead,
they connect hardware to the rest of the kernel through the **ports
system**, which acts as a message buffer / stream abstraction inside the
kernel.

The UART driver sits between:

-   **The UART hardware** (16550-compatible device in QEMU), and
-   **The HAWX ports subsystem**, specifically:
    -   `PORT_CONSOLEOUT`: outbound characters destined for the terminal
    -   `PORT_CONSOLEIN`: inbound characters received from the terminal

Conceptually:

       (keyboard input)                         (screen output)
             |                                         ^
             v                                         |
          UART HW  <---- MMIO registers ---->  UART driver
             |                                         ^
             v                                         |
      PORT_CONSOLEIN                           PORT_CONSOLEOUT
             |                                         ^
             v                                         |
       rest of kernel                           printf/panic/etc.


## What You Should Read

You will benefit from skimming:

-   xv6 `kernel/uart.c`
-   The 16550 UART reference:
    https://uart16550.readthedocs.io/en/latest/index.html

The UART manual is a reference for register meanings. It becomes useful
once you have a code-level question like "what does bit 5 in LSR mean
again?"


## Part 0: Confirm the Symptom

Run the kernel as-is:

``` sh
make qemu
```

It will likely boot into silence.

That's expected: without a working UART, the kernel cannot print. Poor thing
can't even panic!


## Part I: Bring the UART to Life (Polling)

To bootstrap the system, you will first implement enough of the UART
that the kernel can print reliably *without interrupts*.

### Implement

In `kernel/uart.c`:

-   `uartinit`
-   `uartputc`
-   `uartflush`

### Notes

-   `uartputc` is the "emergency" path: it writes directly to the UART
    hardware using polling (busy-waiting).
-   `uartflush` bridges the **ports system** to the UART by repeatedly
    reading bytes from `PORT_CONSOLEOUT` and sending them with
    `uartputc`.

### Expected Output

After these functions are correct:

    HAWX kernel is booting

    UART initialization test...PASSED
    UART flush test...PASSED

Then the system will hang on the next test, because interrupt-driven
operation is not finished yet.


## Part II: Interrupt-Driven Output (PORT_CONSOLEOUT → UART)

Now you will make `PORT_CONSOLEOUT` work "the right way":

-   output is buffered in a port,
-   the UART transmits when ready,
-   and the interrupt handler keeps the transmission going byte by byte.

### Implement

-   `uartstart`
-   `uartintr` (output portion)

### The `uartstart` Contract (Very Important)

In HAWX, `uartstart` is responsible for sending the **next** output
character, but only if both conditions are true:

1.  `PORT_CONSOLEOUT` has data waiting\
    (check `ports[PORT_CONSOLEOUT].count` --- do *not* consume yet)
2.  the UART transmitter can accept a byte\
    (`LSR_TX_IDLE` / THR empty)

If both are true:

-   `port_read(PORT_CONSOLEOUT, &c, 1)`
-   write `c` to `THR`

If either is false, return immediately.

### Expected Output

After interrupt-driven output is correct:

    Interrupt driven output test...PASSED
    Type the word "PASSED" and press enter:

You will still not be able to type successfully yet.


## Part III: Interrupt-Driven Input (UART → PORT_CONSOLEIN)

Now we complete the "inbound" direction.

When the user types:

-   the UART raises an interrupt,
-   the driver reads characters from the UART,
-   places them into `PORT_CONSOLEIN`,
-   and echoes them by writing them into `PORT_CONSOLEOUT`.

### Implement

-   `uartgetc`
-   input portion of `uartintr`

### Input Requirements

In `uartintr`:

1.  Repeatedly call `uartgetc()` until it returns `-1`
2.  For each character:
    -   convert carriage return `\r` to newline `\n`
    -   write to input buffer:
        -   `port_write(PORT_CONSOLEIN, &c, 1)`
    -   echo by writing to output buffer:
        -   `port_write(PORT_CONSOLEOUT, &c, 1)`
3.  After processing input, call:
    -   `uartstart()`\
        to kick off the echo transmission process.

### Expected Final Output

When all UART functionality is complete:

    HAWX kernel is booting

    UART initialization test...PASSED
    UART flush test...PASSED
    Interrupt driven output test...PASSED
    Type the word "PASSED" and press enter: PASSED
    Interrupt driven input test...PASSED
    panic: All done! For now...


## Extra Credit: Backspace

Implement correct backspace behavior in `uartintr`.

A good solution:

-   updates the input port appropriately, and
-   echoes the backspace in a way terminals understand
