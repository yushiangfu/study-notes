> Note: browser extension [GitHub + Mermaid](https://chrome.google.com/webstore/detail/github-%20-mermaid/goiiopgdnkogdbjmncgedmgpoajilohe?hl=en)
> is needed to view flow chart of mermaid syntax

## Index

1. [Introduction](#introduction)
2. [TTY Types](#tty-types)
3. [TTY <-> UART](#tty-uart)
4. Boot & Kernel Boot Command
5. Mechanism of printk

## <a name="introduction"></a> Introduction

Console is the interface that users interact with systems through. 
In the old days, computers were expensive and many users shared one and accessed it by terminals, 
which were composed of small screens and teletypewriters (TTY). 
Nowadays it has become legacy but we can still see it on common OS. 
There are also pseudo TTYs shown when people access consoles in GUI desktop, or operate remote machines by SSH. 
On embedded systems, it's common that developers type commands through serial port interface (UART).

## <a name="tty-types"></a> TTY Types
Device Name       | Description
--                | --
tty1 ~ n          | Command line interface. On Raspberry Pi, Ctrl + Alt + F1 ~ F6 can switch to corresponding TTY 
ptmx & pts/0 ~ n  | Pseudo TTY, responsible for GUI and SSH consoles
ttyS0 ~ n         | It means serial console and each represents one UART within chip
ttyAMA0 ~ n       | ARM serial console?
ttyUSB0 ~ n       | Serial cable with USB interface. E.g. connect the USB-to-TTL cable into Raspberry Pi and you can see one
ttyprintk         | Not really related to TTY. Redirect message to this file and then it can be displayed by command 'dmesg'
tty               | (to be added)
console           | (to be added)

## <a name="tty-uart"></a> TTY <-> UART

```mermaid |
graph TD
   a>TTY -> UART]
   b(TTY Layer <br> e.g. tty_write)
   c(Line Discipline Layer <br> e.g. n_tty_write)
   d(UART Generic Layer <br> e.g. uart_write)
   e(UART Driver Layer <br> e.g. serial8250_start_tx)
   f(UART Driver ISR <br> e.g. serial8250_interrupt)
   g(UART Port Handler <br> e.g. serial8250_default_handle_irq <br> ->mem_serial_out)
   h>Output to Users]
   
   a-.-b
   b-->c
   c-->d
   d-->e
   e-.->|asynchronous|f
   f-->g
   g-.-h
   
   aa>UART -> TTY]
   bb>Input from Users]
   cc(UART Driver ISR <br> e.g. serial8250_interrupt)
   dd(UART Port Handler <br> e.g. serial8250_default_handle_irq <br> ->mem_serial_in)
   ee(UART Generic Layer <br> e.g. uart_insert_char)
   ff(Line Discipline Layer <br> e.g. n_tty_receive_buf)
   gg(TTY Layer <br> e.g. tty_read)
   
   aa-.-bb
   bb-.-cc
   cc-->dd
   dd-.->|Below is data direction instead of code flow|ee
   ee-.->ff
   ff-.->|Ready for read|gg
   
   
   
```

