> Study case: Linux version 5.15.69 on OpenBMC

## Index

- [Introduction](#introduction)
- [TTY to UART](#tty-to-uart)
- [Console](#console)
- [System Startup](#system-startup)
- [Cheat Sheet](#cheat-sheet)
- [Reference](#reference)

1. [Introduction](#introduction)
2. [TTY Types](#tty-types)
3. [TTY <-> UART](#tty-uart)
5. [Mechanism of printk](#mechanism-of-printk)
6. [Kernel Boot Arguments](#kernel-boot-arguments)
7. [Conclusion](#conclusion)

## <a name="introduction"></a> Introduction

```
root@romulus:/dev# ls -l tty* console
crw-------    1 root     root        5,   1 Jan 31 04:06 console
crw-rw-rw-    1 root     tty         5,   0 Jan 31 04:05 tty
crw-rw----    1 root     dialout     4,  64 Jan 31 04:05 ttyS0
crw-rw----    1 root     dialout     4,  65 Jan 31 04:05 ttyS1
crw-rw----    1 root     dialout     4,  66 Jan 31 04:05 ttyS2
crw-rw----    1 root     dialout     4,  67 Jan 31 04:05 ttyS3
crw-------    1 root     tty         4,  68 Feb  1 06:36 ttyS4
crw-rw----    1 root     dialout     4,  69 Jan 31 04:05 ttyS5
lrwxrwxrwx    1 root     root             5 Jan 31 04:05 ttyVUART0 -> ttyS5
```

```
static const struct file_operations tty_fops = {
    .llseek     = no_llseek,
    .read_iter  = tty_read,
    .write_iter = tty_write,
    .splice_read    = generic_file_splice_read,
    .splice_write   = iter_file_splice_write,
    .poll       = tty_poll,
    .unlocked_ioctl = tty_ioctl,
    .compat_ioctl   = tty_compat_ioctl,
    .open       = tty_open,
    .release    = tty_release,
    .fasync     = tty_fasync,
    .show_fdinfo    = tty_show_fdinfo,
};
```

```
drivers/tty/tty_io.c                                                                    
+----------+                                                                             
| tty_open | : ready tty/ldisc/tty_port(xmit buffer, isr, ...), wait                     
+-|--------+                                                                             
  |    +------------------+                                                              
  |--> | nonseekable_open | remove flags to make the file non-seekable                   
  |    +------------------+                                                              
  |    +----------------+                                                                
  |--> | tty_alloc_file | alloc private and save in file                                 
  |    +----------------+                                                                
  |    +----------------------+                                                          
  |--> | tty_open_current_tty | get tty from current->signal, ensure it has ldisc        
  |    +----------------------+                                                          
  |                                                                                      
  |--> if no tty, which means its dev# isn't (5, 0)                                      
  |    |                                                                                 
  |    |    +--------------------+                                                       
  |    +--> | tty_open_by_driver | look up driver, ensure tty and ldisc are ready        
  |         +--------------------+                                                       
  |    +--------------+                                                                  
  |--> | tty_add_file | add file_priv to tty                                             
  |    +--------------+                                                                  
  |                                                                                      
  +--> call ->open, e.g.,                                                                
       +-----------+                                                                     
       | uart_open | ensure tty_port is initialized (xmit buffer, port set up, isr), wait
       +-----------+                                                                     
```

```
drivers/tty/tty_io.c                                                       
+----------------------+                                                    
| tty_open_current_tty | : get tty from current->signal, ensure it has ldisc
+-|--------------------+                                                    
  |                                                                         
  |--> if dev# isn't (5, 0), return null (let caller handle it)             
  |                                                                         
  |    +-----------------+                                                  
  |--> | get_current_tty | get tty from current->signal                     
  |    +-----------------+                                                  
  |    +------------+                                                       
  +--> | tty_reopen | ensure tty has ldisc                                  
       +------------+                                                       
```

```
drivers/tty/tty_io.c                                                                                
+--------------------+                                                                               
| tty_open_by_driver | : look up driver, ensure tty and ldisc are ready                              
+-|------------------+                                                                               
  |    +-------------------+                                                                         
  |--> | tty_lookup_driver | look up tty driver based on dev#                                        
  |    +-------------------+                                                                         
  |    +-----------------------+                                                                     
  |--> | tty_driver_lookup_tty | get tty from driver->ttys[index]                                    
  |    +-----------------------+                                                                     
  |                                                                                                  
  |--> if tty exists                                                                                 
  |    |                                                                                             
  |    |    +------------+                                                                           
  |    +--> | tty_reopen | ensure tty has ldisc                                                      
  |         +------------+                                                                           
  |                                                                                                  
  +--> else                                                                                          
       |                                                                                             
       |    +--------------+                                                                         
       +--> | tty_init_dev | prepare tty and save in driver, ensure it has port, open line discipline
            +--------------+                                                                         
```

```
drivers/tty/tty_io.c                                                                                          
+-------------------+                                                                                          
| tty_lookup_driver | : look up tty driver based on dev#                                                       
+-|-----------------+                                                                                          
  |                                                                                                            
  |--> switch dev#                                                                                             
  |--> case (5, 1)                                                                                             
  |    |                                                                                                       
  |    |    +----------------+                                                                                 
  |    +--> | console_device | traverse console_drivers, get the 1st console with driver, return the driver    
  |         +----------------+                                                                                 
  |                                                                                                            
  +--> default                                                                                                 
       |                                                                                                       
       |    +----------------+                                                                                 
       +--> | get_tty_driver | traverse tty_drivers to find the driver that governs arg dev#, return the driver
            +----------------+                                                                                 
```

```
kernel/printk/printk.c                                                                          
+----------------+                                                                               
| console_device | : traverse console_drivers, get the 1st console with driver, return the driver
+-|--------------+                                                                               
  |                                                                                              
  +--> for each console in console_drivers                                                       
       |                                                                                         
       |--> if ->device() doesn't exist, continue                                                
       |                                                                                         
       |--> call ->device(), e.g.,                                                               
       |    +---------------------+                                                              
       |    | uart_console_device | return index & tty_driver                                    
       |    +---------------------+                                                              
       |                                                                                         
       +--> if driver found, break                                                               
```

```
drivers/tty/tty_io.c                                                                      
+--------------+                                                                           
| tty_init_dev | : prepare tty and save in driver, ensure it has port, open line discipline
+-|------------+                                                                           
  |    +------------------+                                                                
  |--> | alloc_tty_struct | alloc and set up tty                                           
  |    +------------------+                                                                
  |    +------------------------+                                                          
  |--> | tty_driver_install_tty | install tty to driver                                    
  |    +------------------------+                                                          
  |                                                                                        
  |--> ensure tty has port                                                                 
  |                                                                                        
  |    +-----------------+                                                                 
  +--> | tty_ldisc_setup | open tty's line discipline                                      
       +-----------------+                                                                 
```

```
drivers/tty/tty_io.c                                                
+------------------------+                                           
| tty_driver_install_tty | : install tty to driver                   
+-|----------------------+                                           
  |                                                                  
  |--> if ->install() exists                                         
  |    -                                                             
  |    +--> call ->install(), e.g.,                                  
  |         +--------------+                                         
  |         | uart_install | save state in tty, install tty to driver
  |         +--------------+    
  |         +--------------------+                                         
  |         | pty_unix98_install | set up pty pair
  |         +--------------------+   
  |                                                                  
  +--> else                                                          
       |                                                             
       |    +----------------------+                                 
       +--> | tty_standard_install | install tty to driver           
            +----------------------+                                 
```

```
drivers/tty/tty_io.c                                              
+-----------------+                                                
| tty_ldisc_setup | : open tty's line discipline                   
+-|---------------+                                                
  |    +----------------+                                          
  |--> | tty_ldisc_open | open a line discipline                   
  |    +----------------+                                          
  |                                                                
  +--> if o_tty is provided                                        
       |                                                           
       |    +----------------+                                     
       +--> | tty_ldisc_open | open o_tty's line discipline as well
            +----------------+                                     
```

```
drivers/tty/tty_ldisc.c                                 
+----------------+                                       
| tty_ldisc_open | : open a line discipline              
+-|--------------+                                       
  |                                                      
  +--> if ->open() exists                                
       -                                                 
       +--> call it, e.g.,                               
            +------------+                               
            | n_tty_open | alloc and set up ldisc for tty
            +------------+                               
```

```
drivers/tty/serial/serial_core.c                                                            
+-----------+                                                                                
| uart_open | : ensure tty_port is initialized (xmit buffer, port set up, isr), wait         
+-|---------+                                                                                
  |    +---------------+                                                                     
  +--> | tty_port_open | ensure tty_port is initialized (xmit buffer, port set up, isr), wait
       +---------------+                                                                     
```

```
drivers/tty/tty_port.c                                                                                        
+---------------+                                                                                              
| tty_port_open | : ensure tty_port is initialized (xmit buffer, port set up, isr), wait                       
+-|-------------+                                                                                              
  |    +------------------+                                                                                    
  |--> | tty_port_tty_set | save tty in port                                                                   
  |    +------------------+                                                                                    
  |                                                                                                            
  |--> if the port isn't initialized yet                                                                       
  |    |                                                                                                       
  |    |--> if ->activate() exisits                                                                            
  |    |    -                                                                                                  
  |    |    +--> call it, e.g.,                                                                                
  |    |         +--------------------+                                                                        
  |    |         | uart_port_activate | prepare transmit buffer, set up 8250/uart ports, register isr for tx/rx
  |    |         +--------------------+                                                                        
  |    |    +--------------------------+                                                                       
  |    +--> | tty_port_set_initialized | set 'initialized' flag                                                
  |         +--------------------------+                                                                       
  |    +--------------------------+                                                                            
  +--> | tty_port_block_til_ready | waiting logic for tty open                                                 
       +--------------------------+                                                                            
```

```
drivers/tty/serial/serial_core.c                                                               
+--------------------+                                                                          
| uart_port_activate | : prepare transmit buffer, set up 8250/uart ports, register isr for tx/rx
+-|------------------+                                                                          
  |    +--------------+                                                                         
  +--> | uart_startup | prepare transmit buffer, set up 8250/uart ports, register isr for tx/rx 
       +--------------+                                                                         
```

```
drivers/tty/serial/serial_core.c                                                                   
+--------------+                                                                                    
| uart_startup | : prepare transmit buffer, set up 8250/uart ports, register isr for tx/rx          
+-|------------+                                                                                    
  |                                                                                                 
  |--> if tty_port is initialized already, return                                                   
  |                                                                                                 
  |    +-------------------+                                                                        
  +--> | uart_port_startup | prepare transmit buffer, set up 8250/uart ports, register isr for tx/rx
       +-------------------+                                                                        
```

```
drivers/tty/serial/serial_core.c                                                                     
+-------------------+                                                                                 
| uart_port_startup | : prepare transmit buffer, set up 8250/uart ports, register isr for tx/rx       
+-|-----------------+                                                                                 
  |                                                                                                   
  |--> ensure the uart_state has transmit buffer                                                      
  |                                                                                                   
  +--> call ->startup(), e.g.,                                                                        
       +--------------------+                                                                         
       | serial8250_startup | set up 8250/uart ports, clear fifo and interrupt, register isr for tx/rx
       +--------------------+                                                                         
```

```
drivers/tty/serial/8250/8250_port.c                                                                     
+--------------------+                                                                                   
| serial8250_startup | : set up 8250/uart ports, clear fifo and interrupt, register isr for tx/rx        
+-|------------------+                                                                                   
  |                                                                                                      
  |--> if uart_port has ->startup()                                                                      
  |    -                                                                                                 
  |    +--> call it, and return                                                                          
  |                                                                                                      
  |    +-----------------------+                                                                         
  +--> | serial8250_do_startup | set up 8250/uart ports, clear fifo and interrupt, register isr for tx/rx
       +-----------------------+                                                                         
```

```
drivers/tty/serial/8250/8250_port.c                                                                
+-----------------------+                                                                           
| serial8250_do_startup | : set up 8250/uart ports, clear fifo and interrupt, register isr for tx/rx
+-|---------------------+                                                                           
  |                                                                                                 
  |--> set up 8250_port and uart_port                                                               
  |                                                                                                 
  |    +------------------------+                                                                   
  |--> | serial8250_clear_fifos | clear fifo buffer (hw level)                                      
  |    +------------------------+                                                                   
  |                                                                                                 
  |--> clear interrupt registers                                                                    
  |                                                                                                 
  |--> test 'thre' if appropriate                                                                   
  |                                                                                                 
  +--> call ->setup_irq(), e.g.,                                                                    
       +--------------------+                                                                       
       | univ8250_setup_irq | prepare timer or isr to handle tx/rx                                  
       +--------------------+                                                                       
```

```
drivers/tty/serial/8250/8250_core.c                                                             
+--------------------+                                                                           
| univ8250_setup_irq | : prepare timer or isr to handle tx/rx                                    
+-|------------------+                                                                           
  |                                                                                              
  |--> if uart_port has no irq                                                                   
  |    |                                                                                         
  |    |    +-----------+                                                                        
  |    +--> | mod_timer |                                                                        
  |         +-----------+                                                                        
  |                                                                                              
  +--> else                                                                                      
       |                                                                                         
       |    +-----------------------+                                                            
       +--> | serial_link_irq_chain | add 8250_port's irq_info to chain, ensure isr is registered
            +-----------------------+                                                            
```

```
drivers/tty/serial/8250/8250_core.c                                                   
+-----------------------+                                                              
| serial_link_irq_chain | : add 8250_port's irq_info to chain, ensure isr is registered
+-|---------------------+                                                              
  |                                                                                    
  |--> ensure 8250_port has its irq_info in irq_lists                                  
  |                                                                                    
  |--> if irq_info has head (someone's there already)                                  
  |    -                                                                               
  |    +--> add 8250_port to it                                                        
  |                                                                                    
  +--> else                                                                            
       |                                                                               
       |--> set 8250_port as its head                                                  
       |                                                                               
       |    +-------------+                                                            
       +--> | request_irq | register isr                                               
            +-------------+ +----------------------+                                   
                            | serial8250_interrupt |                                   
                            +----------------------+                                   
                            for each irq_info in list: handle irq (perform rx/tx)      
```

```
drivers/tty/serial/8250/8250_core.c                                                             
+----------------------+                                                                         
| serial8250_interrupt | : for each irq_info in list: handle irq (perform rx/tx)                 
+-|--------------------+                                                                         
  |                                                                                              
  +--> for each irq_info in list                                                                 
       |                                                                                         
       |--> get its uart_port                                                                    
       |                                                                                         
       +--> call ->handle_irq(), e.g.,                                                           
            +-------------------------------+                                                    
            | serial8250_default_handle_irq | read line status register, handle rx/tx accordingly
            +-------------------------------+                                                    
```

```
drivers/tty/serial/8250/8250_port.c                                                   
+-------------------------------+                                                      
| serial8250_default_handle_irq | : read line status register, handle rx/tx accordingly
+-|-----------------------------+                                                      
  |    +----------------+                                                              
  |--> | serial_port_in | read interrupt id register                                   
  |    +----------------+                                                              
  |    +-----------------------+                                                       
  +--> | serial8250_handle_irq | read line status register, handle rx/tx accordingly   
       +-----------------------+                                                       
```

```
drivers/tty/serial/8250/8250_port.c                                                  
+-----------------------+                                                             
| serial8250_handle_irq | : read line status register, handle rx/tx accordingly       
+-|---------------------+                                                             
  |    +----------------+                                                             
  |--> | serial_port_in | read line status register                                   
  |    +----------------+                                                             
  |                                                                                   
  |--> if status has flag 'dr' or 'bi' set                                            
  |    |                                                                              
  |    |    +---------------------+                                                   
  |    +--> | serial8250_rx_chars | read chars from register, place in tty read buffer
  |         +---------------------+                                                   
  |                                                                                   
  +--> if status has flag 'thre' set                                                  
       |                                                                              
       |    +---------------------+                                                   
       +--> | serial8250_tx_chars | for chars in transmit buffer, output to register  
            +---------------------+                                                   
```

```
drivers/tty/tty_io.c                                       
+----------+                                                
| tty_read | : wait for ldisc, read data into arg iterator  
+-|--------+                                                
  |    +--------------------+                               
  |--> | tty_ldisc_ref_wait | wait for tty ldisc            
  |    +--------------------+                               
  |                                                         
  +--> if ldisc has ->read()                                
       |                                                    
       |    +------------------+                            
       +--> | iterate_tty_read | read data into arg iterator
            +------------------+                            
```

```
drivers/tty/tty_io.c                             
+------------------+                              
| iterate_tty_read | : read data into arg iterator
+-|----------------+                              
  |                                               
  +--> while still valid                          
       |                                          
       |--> call ->read(), e.g.,                  
       |    +------------+                        
       |    | n_tty_read | read data into arg kbuf
       |    +------------+                        
       |                                          
       |    +--------------+                      
       |--> | copy_to_iter | copy to arg iterator 
       |    +--------------+                      
       |                                          
       +--> update offset and remaining count     
```

```
drivers/tty/tty_io.c                                                               
+-----------+                                                                       
| tty_write | : copy data from iter to tty_write_buf, call tty-level write          
+-|---------+                                                                       
  |    +----------------+                                                           
  +--> | file_tty_write | copy data from iter to tty_write_buf, call tty-level write
       +----------------+                                                           
```

```
drivers/tty/tty_io.c                                                             
+----------------+                                                                
| file_tty_write | : copy data from iter to tty_write_buf, call tty-level write   
+-|--------------+                                                                
  |    +--------------------+                                                     
  |--> | tty_ldisc_ref_wait | wait for tty ldisc                                  
  |    +--------------------+                                                     
  |    +--------------+                                                           
  +--> | do_tty_write | copy data from iter to tty_write_buf, call tty-level write
       +--------------+                                                           
```

```
drivers/tty/tty_io.c                                                        
+--------------+                                                             
| do_tty_write | : copy data from iter to tty_write_buf, call tty-level write
+-|------------+                                                             
  |                                                                          
  +--> endless loop                                                          
       |                                                                     
       |    +----------------+                                               
       |--> | copy_from_iter | copy data from iterator to tty write buf      
       |    +----------------+                                               
       |                                                                     
       |--> cal arg write(), e.g.,                                           
       |    +-------------+                                                  
       |    | n_tty_write | tty-level write                                  
       |    +-------------+                                                  
       |                                                                     
       |--> update written and remaining count                               
       |                                                                     
       +--> break if nothing left to write                                   
```

```
drivers/tty/tty_io.c                                                           
+-------------+                                                                 
| tty_release | : delete file, release ldisc, flush work, remove tty from driver
+-|-----------+                                                                 
  |    +--------------+                                                         
  |--> | __tty_fasync | (skip)                                                  
  |    +--------------+                                                         
  |                                                                             
  |--> if ->close() exists                                                      
  |    -                                                                        
  |    +--> call it, e.g.,                                                      
  |         +------------+                                                      
  |         | uart_close |                                                      
  |         +------------+                                                      
  |    +--------------+                                                         
  +--> | tty_del_file | delete file from tty (free file private)                
  |    +--------------+                                                         
  |    +--------------------+                                                   
  +--> | tty_release_struct | release ldisc, flush work, remove tty from driver 
       +--------------------+                                                   
```

## <a name="tty-to-uart"></a> TTY to UART

(TBD)

### Virtual UART

### Net Cconsole (PTY?)

### Line Discipline



## <a name="tty-types"></a> TTY Types

Device Name       | Description
--                | --
tty1 ~ n          | Command line interface. On Raspberry Pi, Ctrl + Alt + F1 ~ F6 can switch to corresponding TTY 
ptmx & pts/0 ~ n  | Pseudo TTY, responsible for GUI and SSH consoles
ttyS0 ~ n         | It means serial console and each represents one UART within chip
ttyAMA0 ~ n       | ARM serial console?
ttyUSB0 ~ n       | Serial cable with USB interface. E.g. Plug in the USB-to-TTL cable to Raspberry Pi and you can see one
ttyprintk         | Redirect message to this file and then it can be displayed by command 'dmesg'
tty               | Conceptually it's like a link that always points to the TTY device of current task
console           | In terms of results, it represents the one specified by the kernel boot argument 'console=', which is ttyS4 in my case

## <a name="tty-uart"></a> TTY <-> UART

By default, each user space task has the first three file descriptors used for stdin/stdout/stderr. Issue below command to inspect which TTY devices are used.
```
ls -l /proc/$$/fd
```

```
In serial console:
root@romulus:~# ls -l /proc/$$/fd
lrwx------    1 root     root            64 Jul 27 15:25 0 -> /dev/ttyS4
lrwx------    1 root     root            64 Jul 27 15:25 1 -> /dev/ttyS4
lrwx------    1 root     root            64 Jul 27 15:25 2 -> /dev/ttyS4
lrwx------    1 root     root            64 Jul 27 15:25 255 -> /dev/ttyS4

In SSH console:
root@romulus:~# ls -l /proc/$$/fd
lrwx------    1 root     root            64 Jul 27 15:26 0 -> /dev/pts/0
lrwx------    1 root     root            64 Jul 27 15:26 1 -> /dev/pts/0
lrwx------    1 root     root            64 Jul 27 15:26 2 -> /dev/pts/0
lrwx------    1 root     root            64 Jul 27 15:26 255 -> /dev/pts/0
```

Below are the flowcharts showing how data is output to or input from users.

```mermaid
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

## <a name="system-startup"></a> System Startup

```
param_setup_earlycon:            set up early console
console_setup:                   parse 'console=xxx,yyy', add as preferred console 
proc_consoles_init:              create /proc/consoles
chr_dev_init:                    (don't care)
   tty_init:                     init char devices for tty and console
aspeed_uart_routing_driver_init: register 'aspeed_uart_routing_driver' to bus 'platform'
pty_init:                           
serial8250_init:                 init and register 'seerial8250_ports', prepare tty driver, register platform dev/drv (serial8250)
   serial8250_probe:             for each valid port in dev data: register a 8250 port
aspeed_vuart_driver_init:        register platform driver 'aspeed_vuart_driver'
   aspeed_vuart_probe:           prepare port (ops), install handle_irq, register 8250 ports, set enabled  
of_platform_serial_driver_init:  register platform driver 'of_platform_serial_driver'
   of_platform_serial_probe:     get iomem/irq, prepare info, register 8250 ports
mctp_serial_init:                (mctp over serial, skip)
usb_serial_init:                 prepare tty driver, register bus 'usb-serial'
usb_serial_module_init:          register usb intf driver, register arg drivers to bus 'usb-serial'
init_netconsole:                 register netdevice notifer, register net console
univ8250_console_init:           init serial8250 ports, register console (univ8250)          
```

```
drivers/tty/serial/earlycon.c                                       
+----------------------+                                             
| param_setup_earlycon | : set up early console                      
+-|--------------------+                                             
  |                                                                  
  |--> if no buf (it is 'earlycon' alone)                            
  |    |                                                             
  |    |    +----------------------------------+                     
  |    |--> | early_init_dt_scan_chosen_stdout | set up early console
  |    |    +----------------------------------+                     
  |    |                                                             
  |    +--> return                                                   
  |                                                                  
  |    +----------------+                                            
  +--> | setup_earlycon | （skip, it is for case of 'earlycon=???')   
       +----------------+                                            
```

```
drivers/of/fdt.c                                                                                                     
+----------------------------------+                                                                                  
| early_init_dt_scan_chosen_stdout | ： set up early console                                                           
+-|--------------------------------+                                                                                  
  |                                                                                                                   
  |--> get node '/chosen'                                                                                             
  |                                                                                                                   
  |--> get property 'stdout-path'                                                                                     
  |                                                                                                                   
  |--> based on property value, get node, e.g., 'serial@1e784000'                                                     
  |                                                                                                                   
  +--> for each entry in '__earlycon_table' (drivers/tty/serial/8250/8250_early.c)                                    
       |                                                                                                              
       |--> if entry has no 'compatible' field, continue                                                              
       |                                                                                                              
       |    +---------------------------+                                                                             
       |--> | fdt_node_check_compatible | check if entry can handle the earlycon                                      
       |    +---------------------------+                                                                             
       |                                                                                                              
       |--> if not, continue                                                                                          
       |                                                                                                              
       |    +-------------------+                                                                                     
       +--> | of_setup_earlycon | iomap, handle dt properties, set up uart/ops, register console, print out queued log
            +-------------------+          
            
chosen {
   stdout-path = "/ahb/apb/serial@1e784000"; <----------
   bootargs = "console=ttyS4,115200 earlycon";
};

serial@1e784000 {
    compatible = "ns16550a"; <----------
    reg = <0x1e784000 0x20>;
    reg-shift = <0x02>;
    interrupts = <0x0a>;
    clocks = <0x02 0x0f>;
    no-loopback-test;
    status = "okay";
};

drivers/tty/serial/8250/8250_early.c
   EARLYCON_DECLARE(uart8250, early_serial8250_setup);
   EARLYCON_DECLARE(uart, early_serial8250_setup);
   OF_EARLYCON_DECLARE(ns16550, "ns16550", early_serial8250_setup);
   OF_EARLYCON_DECLARE(ns16550a, "ns16550a", early_serial8250_setup); <----------
   OF_EARLYCON_DECLARE(uart, "nvidia,tegra20-uart", early_serial8250_setup);
   OF_EARLYCON_DECLARE(uart, "snps,dw-apb-uart", early_serial8250_setup);
```

```
drivers/tty/serial/earlycon.c                                                                              
+-------------------+                                                                                       
| of_setup_earlycon | : iomap, handle dt properties, set up uart/ops, register console, print out queued log
+-|-----------------+                                                                                       
  |    +------------------------------+                                                                     
  |--> | of_flat_dt_translate_address | translate reg addr from given node, e.g., 'serial@1e784000'         
  |    +------------------------------+                                                                     
  |                                                                                                         
  |--> handle other dt properties                                                                           
  |                                                                                                         
  |--> if options (baud rate) is provided, save it in early_console_dev                                     
  |                                                                                                         
  |    +---------------+                                                                                    
  |--> | earlycon_init | parse string to determine idx                                                      
  |    +---------------+                                                                                    
  |                                                                                                         
  |--> call ->setup(), e.g.,                                                                                
  |    +------------------------+                                                                           
  |    | early_serial8250_setup | set up uart and install ops                                               
  |    +------------------------+                                                                           
  |                                                                                                         
  |    +---------------------+                                                                              
  |--> | earlycon_print_info | print, e.g., "earlycon: ns16550a0 at MMIO 0x1e784000 (options '')"           
  |    +---------------------+                                                                              
  |    +------------------+                                                                                 
  +--> | register_console | add console to 'console_drivers', print out queued log                          
       +------------------+                                                                                 
```

```
drivers/tty/serial/8250/8250_early.c                                         
+------------------------+                                                    
| early_serial8250_setup | : set up uart and install ops                      
+-|----------------------+                                                    
  |                                                                           
  |--> if dev hasn't specified baud rate                                      
  |    -                                                                      
  |    +--> read/write hw reg to setup uart                                   
  |                                                                           
  |--> else (not our case)                                                    
  |    |                                                                      
  |    |    +-----------+                                                     
  |    +--> | init_port | read/write hw reg to config baud rate and setup uart
  |         +-----------+                                                     
  |                                                                           
  +--> install ops                                                            
       +------------------------+                                             
       | early_serial8250_write | write each char in string to hw reg         
       +-----------------------++                                             
       | early_serial8250_read | (null bc of disabled config)                 
       +-----------------------+                                              
```

```
drivers/tty/serial/8250/8250_early.c                                              
+------------------------+                                                         
| early_serial8250_write | : write each char in string to hw reg                   
+-|----------------------+                                                         
  |    +--------------------+                                                      
  +--> | uart_console_write | : write each char in string to hw reg                
       +-|------------------+                                                      
         |                                                                         
         +--> for each char in string                                              
              -                                                                    
              +--> call arg putchar(), e.g.,                                       
                   +-------------+                                                 
                   | serial_putc | write 'c' to tx reg, wait till status shows done
                   +-------------+                                                 
```

```
kernel/printk/printk.c                                                                                          
+------------------+                                                                                             
| register_console | : add console to 'console_drivers', print out queued log                                    
+-|----------------+                                                                                             
  |                                                                                                              
  |--> ensure we have a preferred console                                                                        
  |                                                                                                              
  |    +------------------------+                                                                                
  |--> | try_enable_new_console | try to match new_con with 'console_cmdline', enable it if matched              
  |    +------------------------+                                                                                
  |                                                                                                              
  |--> if fail, try again with loosen condition                                                                  
  |                                                                                                              
  |    +--------------+                                                                                          
  |--> | console_lock |                                                                                          
  |    +--------------+                                                                                          
  |                                                                                                              
  |--> add new_con to the head of 'console_drivers'                                                              
  |                                                                                                              
  |    +----------------+                                                                                        
  |--> | console_unlock | for each valid record: prepend timestamp and ask console drivers to print out          
  |    +----------------+                                                                                        
  |    +----------------------+                                                                                  
  |--> | console_sysfs_notify | (skip)                                                                           
  |    +----------------------+                                                                                  
  |                                                                                                              
  |--> print, e.g., 'printk: bootconsole [ns16550a0] enabled'                                                    
  |                                                                                                              
  +--> if we have boot console already, and new one is real console                                              
       -                                                                                                         
       +--> for each console in 'console_drivers'                                                                
            -                                                                                                    
            +--> if it's boot console                                                                            
                 |                                                                                               
                 |    +--------------------+                                                                     
                 +--> | unregister_console | remove console from list, clear 'enabled' flag, print out queued log
                      +--------------------+                                                                     
```

```
kernel/printk/printk.c                                                                       
+------------------------+                                                                    
| try_enable_new_console | : try to match new_con with 'console_cmdline', enable it if matched
+-|----------------------+                                                                    
  |                                                                                           
  +--> for each console in 'console_cmdline' (e.g., added by console_setup)                   
       |                                                                                      
       |--> if attributes 'user_specified' mismatch, continue                                 
       |                                                                                      
       |--> if new_con doesn't have ->match() or fail to match                                
       |    |                                                                                 
       |    |--> try other attributes: name, index, and ->setup()                             
       |    |                                                                                 
       |    +--> if it still fails, continue or return error                                  
       |                                                                                      
       |--> label 'enabled' on new_con                                                        
       |                                                                                      
       |--> if it's the preferred console                                                     
       |    |                                                                                 
       |    |--> label 'cons_dev' on new_con                                                  
       |    |                                                                                 
       |    +--> has_preferred_console = true                                                 
       |                                                                                      
       +--> return                                                                            
```

```
kernel/printk/printk.c                                                                           
+----------------+                                                                                
| console_unlock | : for each valid record: prepend timestamp and ask console drivers to print out
+-|--------------+                                                                                
  |    +-----------------+                                                                        
  |--> | prb_rec_init_rd | init record with info and text                                         
  |    +-----------------+                                                                        
  |again:                                                                                         
  |--> endless loop                                                                               
  |    |                                                                                          
  |    |    +----------------+                                                                    
  |    |--> | prb_read_valid | read a record, update arg seq#                                     
  |    |    +----------------+                                                                    
  |    |                                                                                          
  |    |--> if fail to read, break                                                                
  |    |                                                                                          
  |    |--> ensure console_seq == r.info->seq                                                     
  |    |                                                                                          
  |    |    +-------------------+                                                                 
  |    |--> | record_print_text | prepend prefix and append '\0' to text body                     
  |    |    +-------------------+                                                                 
  |    |                                                                                          
  |    |--> console_seq++                                                                         
  |    |                                                                                          
  |    |    +----------------------+                                                              
  |    +--> | call_console_drivers | for each driver in 'console_drivers': call ->write()         
  |         +----------------------+                                                              
  |                                                                                               
  |--> next_seq = console_seq                                                                     
  |                                                                                               
  |    +----------------+                                                                         
  |--> | prb_read_valid | next_seq                                                                
  |    +----------------+                                                                         
  |                                                                                               
  +--> if next_seq is valid (somewhere fills it while we're performing the above actions)         
       -                                                                                          
       +--> go to 'again'                                                                         
```

```
kernel/printk/printk_ringbuffer.c                               
+----------------+                                               
| prb_read_valid | : read a record, update arg seq#              
+-|--------------+                                               
  |    +-----------------+                                       
  +--> | _prb_read_valid | : read a record, update arg seq#      
       +-|---------------+                                       
         |          +----------+                                 
         |--> while | prb_read |                                 
         |    |     +----------+                                 
         |    |     copy text data from ring buffer to arg record
         |    |                                                  
         |    |    +---------------+                             
         |    |--> | prb_first_seq | get seq# of tail descriptor 
         |    |    +---------------+                             
         |    |                                                  
         |    |--> if seq# < tail                                
         |    |    -                                             
         |    |    +--> seq# = tail (catch up and try again)     
         |    |                                                  
         |    +--> else                                          
         |         -                                             
         |         +--> return false (no existent record)        
         |                                                       
         +--> return true                                        
```

```
kernel/printk/printk_ringbuffer.c                                         
+----------+                                                               
| prb_read | : copy text data from ring buffer to arg record               
+-|--------+                                                               
  |                                                                        
  |--> extract descriptor id                                               
  |                                                                        
  |    +-------------------------+                                         
  |--> | desc_read_finalized_seq | copy desc to desc_out                   
  |    +-------------------------+                                         
  |                                                                        
  |--> if r->info is provided, copy info to it                             
  |                                                                        
  |    +-----------+                                                       
  |--> | copy_data |                                                       
  |    +-----------+                                                       
  |    +-------------------------+                                         
  +--> | desc_read_finalized_seq | to ensure it's still a finalized record?
       +-------------------------+                                         
```

```
kernel/printk/printk_ringbuffer.c                 
+-------------------------+                        
| desc_read_finalized_seq | : copy desc to desc_out
+-|-----------------------+                        
  |    +-----------+                               
  +--> | desc_read | : copy desc to desc_out       
       +-|---------+                               
         |                                         
         |--> copy desc to desc_out                
         |                                         
         +--> save state in desc_out               
```

```
kernel/printk/printk.c
+-------------------+                                                        
| record_print_text | : prepend prefix and append '\0' to text body          
+-|-----------------+                                                        
  |    +-------------------+                                                 
  |--> | info_print_prefix | add prefix (syslog/timestamp/caller) is required
  |    +-------------------+                                                 
  |                                                                          
  |--> endless loop (to handle multiple lines)                               
  |    |                                                                     
  |    |--> determine line_len                                               
  |    |                                                                     
  |    |--> assemble prefix and text                                         
  |    |                                                                     
  |    +--> append '\n' and break                                            
  |                                                                          
  +--> append '\0'                                                           
```

```
kernel/printk/printk.c                                                 
+-------------------+                                                   
| info_print_prefix | : add prefix (syslog/timestamp/caller) is required
+-|-----------------+                                                   
  |                                                                     
  |--> if arg syslog is specified                                       
  |    |                                                                
  |    |    +--------------+                                            
  |    +--> | print_syslog | add "<%u>" to buffer                       
  |         +--------------+                                            
  |                                                                     
  |--> if arg time is specified                                         
  |    |                                                                
  |    |    +------------+                                              
  |    +--> | print_time | add timestamp to buffer                      
  |         +------------+                                              
  |    +--------------+                                                 
  +--> | print_caller | print caller?                                   
       +--------------+                                                 
```

```
kernel/printk/printk.c                                                        
+----------------------+                                                       
| call_console_drivers | : for each driver in 'console_drivers': call ->write()
+-|--------------------+                                                       
  |                                                                            
  +--> for each driver in 'console_drivers'                                    
       -                                                                       
       +--> call driver->write(), e.g.,                                        
            +------------------------+                                         
            | early_serial8250_write | write each char in string to hw reg     
            +------------------------+                                         
```

```
kernel/printk/printk.c                                                                                
+--------------------+                                                                                 
| unregister_console | : remove console from list, clear 'enabled' flag, print out queued log          
+-|------------------+                                                                                 
  |                                                                                                    
  +--> print, e.g., "printk: bootconsole [ns16550a0] disabled"                                         
  |                                                                                                    
  |    +--------------+                                                                                
  |--> | console_lock |                                                                                
  |    +--------------+                                                                                
  |                                                                                                    
  |--> remove arg console from 'console_drivers'                                                       
  |                                                                                                    
  |--> remove flag 'enabled' from arg console                                                          
  |                                                                                                    
  |    +----------------+                                                                              
  |--> | console_unlock | for each valid record: prepend timestamp and ask console drivers to print out
  |    +----------------+                                                                              
  |                                                                                                    
  +--> if ->exit() exists, call it                                                                     
```

```
kernel/printk/printk.c                                                                   
+---------------+                                                                         
| console_setup | : parse 'console=xxx,yyy', add as preferred console                     
+-|-------------+                                                                         
  |                                                                                       
  |--> parse arg 'str, e.g., buf=ttyS4, options=115200                                    
  |                                                                                       
  |    +-------------------------+                                                        
  |--> | __add_preferred_console | ensure preferred console is set up in 'console_cmdline'
  |    +-------------------------+                                                        
  |                                                                                       
  +--> console_set_on_cmdline = 1                                                         
```

```
kernel/printk/printk.c                                                                 
+-------------------------+                                                             
| __add_preferred_console | ： ensure preferred console is set up in 'console_cmdline'   
+-|-----------------------+                                                             
  |                                                                                     
  |--> for each entry in 'console_cmdline'                                              
  |    |                                                                                
  |    |--> find name- and index-matched console_cmd                                    
  |    |                                                                                
  |    +--> if found                                                                    
  |         |                                                                           
  |         |--> preferred_console = i                                                  
  |         |                                                                           
  |         +--> return                                                                 
  |                                                                                     
  |--> preferred_console = i                                                            
  |                                                                                     
  +--> set up first empty entry in 'console_cmdline' (name/options/user_specified/index)
```

```
drivers/tty/tty_io.c                                                     
+----------+                                                              
| tty_init | : init char devices for tty and console                      
+-|--------+                                                              
  |    +-----------------+                                                
  |--> | tty_sysctl_init | (skip)                                         
  |    +-----------------+                                                
  |    +-----------+                                                      
  |--> | cdev_init | init cdev and install ops (tty_fops)                 
  |    +-----------+                                                      
  |    +----------+                                                       
  |--> | cdev_add | register cdev (tty_cdev)                              
  |    +----------+                                                       
  |    +------------------------+                                         
  |--> | register_chrdev_region | reserve the specified dev# range        
  |    +------------------------+                                         
  |    +---------------+                                                  
  |--> | device_create | given params (tty), create device                
  |    +---------------+                                                  
  |    +-----------+                                                      
  |--> | cdev_init | init cdev and install ops (console_fops)             
  |    +-----------+                                                      
  |    +----------+                                                       
  |--> | cdev_add | register cdev (console_cdev)                          
  |    +----------+                                                       
  |    +------------------------+                                         
  |--> | register_chrdev_region | reserve the specified dev# range        
  |    +------------------------+                                         
  |    +---------------------------+                                      
  +--> | device_create_with_groups | given params (console), create device
       +---------------------------+                                      
```

```
drivers/tty/pty.c                                                                                       
+----------+                                                                                             
| pty_init |                                                                                             
+-|--------+                                                                                             
  |    +-----------------+                                                                               
  |--> | legacy_pty_init | (do nothing bc of disabled config)                                            
  |    +-----------------+                                                                               
  |    +-----------------+                                                                               
  +--> | unix98_pty_init | alloc and register tty drivers for ptm/pts, prepare and register cdev for ptmx
       +-----------------+                                                                               
```

```
drivers/tty/pty.c                                                                                  
+-----------------+                                                                                 
| unix98_pty_init | : alloc and register tty drivers for ptm/pts, prepare and register cdev for ptmx
+-|---------------+                                                                                 
  |    +------------------+                                                                         
  |--> | tty_alloc_driver | prepare tty_driver (e.g., uart case: alloc ttys/termios/ports)          
  |    +------------------+                                                                         
  |    +------------------+                                                                         
  |--> | tty_alloc_driver | the above is for ptm, this one is for pts                               
  |    +------------------+                                                                         
  |                                                                                                 
  |--> set up ptm (major = 128, install ptm_unix98_ops)                                             
  |                                                                                                 
  |--> set up pts (major = 136, install pty_unix98_ops)                                             
  |                                                                                                 
  |    +---------------------+                                                                      
  |--> | tty_register_driver | reserve dev# range, add tty_driver to list, create proc file(s)      
  |    +---------------------+                                                                      
  |    +---------------------+                                                                      
  |--> | tty_register_driver | the above is for ptm, this one is for pts                            
  |    +---------------------+                                                                      
  |    +------------------+                                                                         
  |--> | tty_default_fops | ptmx_fops = tty_fops                                                    
  |    +------------------+                                                                         
  |                                                                                                 
  |--> overwrite ->open() = ptmx_open()                                                             
  |                         +-----------+                                                           
  |                         | ptmx_open |                                                           
  |                         +-----------+                                                           
  |    +-----------+                                                                                
  |--> | cdev_init | init cdev and install ops (ptmx_fops)                                          
  |    +-----------+                                                                                
  |    +----------+                                                                                 
  |--> | cdev_add | register cdev (ptmx_cdev)                                                       
  |    +----------+                                                                                 
  |    +------------------------+                                                                   
  |--> | register_chrdev_region |                                                                   
  |    +------------------------+                                                                   
  |    +---------------+                                                                            
  +--> | device_create |                                                                            
       +---------------+                                                                            
```

```
static const struct tty_operations ptm_unix98_ops = {
    .lookup = ptm_unix98_lookup,
    .install = pty_unix98_install,
    .remove = pty_unix98_remove,
    .open = pty_open, ------------------------ set flags
    .close = pty_close, ---------------------- set flags
    .write = pty_write, ---------------------- copy data to tty buffer, and flush to ldisc
    .write_room = pty_write_room,
    .flush_buffer = pty_flush_buffer,
    .unthrottle = pty_unthrottle,
    .ioctl = pty_unix98_ioctl,
    .compat_ioctl = pty_unix98_compat_ioctl,
    .resize = pty_resize,
    .cleanup = pty_cleanup,
    .show_fdinfo = pty_show_fdinfo,
};                                                                                      
```

```
drivers/tty/pty.c                                                                           
+-----------+                                                                                
| pty_write | : copy data to tty buffer, and flush to ldisc                                  
+-|---------+                                                                                
  |    +----------------------------------------+                                            
  +--> | tty_insert_flip_string_and_push_buffer | copy data to tty buffer, and flush to ldisc
       +----------------------------------------+                                            
```

```
drivers/tty/tty_buffer.c                                                                 
+----------------------------------------+                                                
| tty_insert_flip_string_and_push_buffer | : copy data to tty buffer, and flush to ldisc  
+-|--------------------------------------+                                                
  |    +------------------------+                                                         
  |--> | tty_insert_flip_string | copy data to tty buffer                                 
  |    +------------------------+                                                         
  |    +------------------------+                                                         
  |--> | tty_flip_buffer_commit | data is ready to be flushed to ldisc                    
  |    +------------------------+                                                         
  |    +------------+ +----------------+                                                  
  +--> | queue_work | | flush_to_ldisc |                                                  
       +------------+ +----------------+                                                  
                      receive data to tty read buffer till no more, wake up task if needed
```

```
include/linux/tty_flip.h                                             
+------------------------+                                            
| tty_insert_flip_string | : copy data to tty buffer                  
+-|----------------------+                                            
  |    +-----------------------------------+                          
  +--> | tty_insert_flip_string_fixed_flag | : copy data to tty buffer
       +-|---------------------------------+                          
         |    +---------------------------+                           
         |--> | __tty_buffer_request_room | grow tty buffer if needed 
         |    +---------------------------+                           
         |                                                            
         +--> copy data to tty buffer                                 
```

```
drivers/tty/pty.c                                                                              
+-----------+                                                                                   
| ptmx_open | : open a unix98 pty master (create inode for pts)                                 
+-|---------+                                                                                   
  |    +----------------+                                                                       
  |--> | tty_alloc_file | alloc private and save in file                                        
  |    +----------------+                                                                       
  |    +----------------+                                                                       
  |--> | devpts_acquire | get fs_info of sb                                                     
  |    +----------------+                                                                       
  |    +------------------+                                                                     
  |--> | devpts_new_index | find an unused index                                                
  |    +------------------+                                                                     
  |    +--------------+                                                                         
  |--> | tty_init_dev | prepare tty and save in driver, ensure it has port, open line discipline
  |    +--------------+                                                                         
  |    +--------------+                                                                         
  |--> | tty_add_file | add file_priv to tty                                                    
  |    +--------------+                                                                         
  |    +----------------+                                                                       
  |--> | devpts_pty_new | create (inode, dentry) for pts                                        
  |    +----------------+                                                                       
  |                                                                                             
  +--> call ->open(), e.g.,                                                                     
       +----------+                                                                             
       | pty_open | set flags on tty                                                            
       +----------+                                                                             
```

```
drivers/tty/pty.c                                     
+----------------+                                     
| devpts_pty_new | : create (inode, dentry) for pts    
+-|--------------+                                     
  |    +-----------+                                   
  |--> | new_inode |                                   
  |    +-----------+                                   
  |    +--------------------+                          
  |--> | init_special_inode | char dev for slave       
  |    +--------------------+                          
  |    +--------------+                                
  |--> | d_alloc_name | alloc child dentry of sb dentry
  |    +--------------+                                
  |    +-------+                                       
  +--> | d_add | add to hash table                     
       +-------+                                       
```

```

```

```
drivers/tty/serial/8250/8250_core.c                                                                                     
+-----------------+                                                                                                      
| serial8250_init | : init and register 'seerial8250_ports', prepare tty driver, register platform dev/drv (serial8250)  
+-|---------------+                                                                                                      
  |    +---------------------------+                                                                                     
  |--> |serial8250_isa_init_ports  | init each port in 'serial8250_ports'                                                
  |    +---------------------------+                                                                                     
  |                                                                                                                      
  |--> print, e.g., Serial: "8250/16550 driver, 6 ports, IRQ sharing enabled"                                            
  |                                                                                                                      
  |    +----------------------+                                                                                          
  |--> | uart_register_driver | prepare tty driver (ops, cdeev, ...) for uart driver                                     
  |    +----------------------+                                                                                          
  |    +-----------------------+                                                                                         
  |--> | platform_device_alloc | prepare platform_object (name = "serial8250")                                           
  |    +-----------------------+                                                                                         
  |    +---------------------+                                                                                           
  |--> | platform_device_add | add device to bus 'platform'                                                              
  |    +---------------------+                                                                                           
  |    +---------------------------+                                                                                     
  |--> | serial8250_register_ports | for each port in 'serial8250_ports': config port, prepare device/cdev, register them
  |    +---------------------------+                                                                                     
  |    +--------------------------+                                                                                      
  +--> | platform_driver_register | register platform driver 'serial8250_isa_driver'                                     
       +--------------------------+                                                                                      
```

```
drivers/tty/serial/8250/8250_core.c
+---------------------------+
| serial8250_isa_init_ports | : init each port in 'serial8250_ports'
+-|-------------------------+
  |
  |--> for each port in 'serial8250_ports'
  |    |
  |    |    +----------------------+
  |    |--> | serial8250_init_port | init port and install ops (serial8250_pops)
  |    |    +----------------------+
  |    |
  |    |--> overwrite ops with 'univ8250_port_ops' (which is later duplicated from serial8250_pops)
  |    |
  |    |    +-------------+ +--------------------+
  |    |--> | timer_setup | | serial8250_timeout |
  |    |    +-------------+ +--------------------+
  |    |
  |    |--> install uart ops (univ8250_driver_ops)
  |    |
  |    |    +-------------------------+
  |    +--> | serial8250_set_defaults | install default ops for up and p
  |         +-------------------------+
  |
  +--> univ8250_driver_ops = base_ops, e.g., serial8250_pops
```

```
drivers/tty/serial/8250/8250_port.c                                 
+-------------------------+                                          
| serial8250_set_defaults | : install default ops for up and p       
+-|-----------------------+                                          
  |    +------------------+                                          
  |--> | set_io_from_upio | install ops for up, install ops/isr for p
  |    +------------------+                                          
  |                                                                  
  +--> if specified, install dma ops                                 
```

```
drivers/tty/serial/serial_core.c                                                             
+----------------------+                                                                      
| uart_register_driver | : prepare tty driver (ops, cdeev, ...) for uart driver               
+-|--------------------+                                                                      
  |                                                                                           
  |--> alloc state                                                                            
  |                                                                                           
  |    +------------------+                                                                   
  |--> | tty_alloc_driver | : prepare tty_driver (tty, ports, cdevs)                          
  |    +------------------+                                                                   
  |                                                                                           
  |--> save tty_driver in uart_driver                                                         
  |                                                                                           
  |--> set up tty_driver and install ops (uart_ops)                                           
  |                                                                                           
  |--> for each uart state                                                                    
  |    |                                                                                      
  |    |--> get tty_port from state                                                           
  |    |                                                                                      
  |    |    +---------------+                                                                 
  |    +--> | tty_port_init | init tty_port and install ops (tty_port_default_client_ops)     
  |         +---------------+                                                                 
  |    +---------------------+                                                                
  +--> | tty_register_driver | reserve dev# range, add tty_driver to list, create proc file(s)
       +---------------------+                                                                
```

```
include/linux/tty_driver.h                                                                   
+------------------+                                                                          
| tty_alloc_driver | : prepare tty_driver (e.g., uart case: alloc ttys/termios/ports)         
+-|----------------+                                                                          
  |    +--------------------+                                                                 
  +--> | __tty_alloc_driver | : prepare tty_driver (e.g., uart case: alloc ttys/termios/ports)
       +-|------------------+                                                                 
         |                                                                                    
         |--> alloc and set up 'driver' struct (magic, lines, ...)                            
         |                                                                                    
         |--> if no 'devpts_mem' flag (e.g., uart case)                                       
         |    -                                                                               
         |    +--> alloc ttys and termios for 'driver'                                        
         |                                                                                    
         |--> if no 'dynamic_alloc' flag (e.g., uart case)                                    
         |    -                                                                               
         |    +--> alloc ports for 'driver'                                                   
         |                                                                                    
         +--> alloc cdevs for 'driver'                                                        
                                                                                             
                                                                                              
     uart                          pty                                                        
     TTY_DRIVER_REAL_RAW           TTY_DRIVER_RESET_TERMIOS                                   
     TTY_DRIVER_DYNAMIC_DEV        TTY_DRIVER_REAL_RAW                                        
                                   TTY_DRIVER_DYNAMIC_DEV                                     
                                   TTY_DRIVER_DEVPTS_MEM                                      
                                   TTY_DRIVER_DYNAMIC_ALLOC                                   
```

```
drivers/tty/tty_io.c                                                                      
+---------------------+                                                                    
| tty_register_driver | : reserve dev# range, add tty_driver to list, create proc file(s)  
+-|-------------------+                                                                    
  |                                                                                        
  |--> alloc or register char dev# range                                                   
  |                                                                                        
  |--> if driver has specified 'dynamic_alloc' (not our case)                              
  |    |                                                                                   
  |    |    +--------------+                                                               
  |    +--> | tty_cdev_add | prepare cdev and install ops (tty_fops), add to slot, add cdev
  |         +--------------+                                                               
  |                                                                                        
  |--> add tty_driver to 'tty_drivers'                                                     
  |                                                                                        
  |--> if driver hasn't specified 'dynamic_dev' (not our case)                             
  |    -                                                                                   
  |    +--> for each dev governed by driver                                                
  |         |                                                                              
  |         |    +---------------------+                                                   
  |         +--> | tty_register_device |                                                   
  |              +---------------------+                                                   
  |    +--------------------------+                                                        
  |--> | proc_tty_register_driver | create proc file(s)                                    
  |    +--------------------------+                                                        
  |                                                                                        
  +--> label 'installed' on driver                                                         
```

```
drivers/tty/serial/8250/8250_core.c                                                                                
+---------------------------+                                                                                       
| serial8250_register_ports | : for each port in 'serial8250_ports': config port, prepare device/cdev, register them
+-|-------------------------+                                                                                       
  |                                                                                                                 
  +--> for each port in 'serial8250_ports'                                                                          
       |                                                                                                            
       |--> save dev in it                                                                                          
       |                                                                                                            
       |    +-------------------+                                                                                   
       +--> | uart_add_one_port | config port, save port in driver, alloc device/cdev for tty and register them     
            +-------------------+                                                                                   
```

```
drivers/tty/serial/8250/8250_core.c                                                                            
+-------------------+                                                                                           
| uart_add_one_port | : config port, save port in driver, alloc device/cdev for tty and register them           
+-|-----------------+                                                                                           
  |                                                                                                             
  |--> given line as index, get target state and port                                                           
  |                                                                                                             
  |--> link uport and state                                                                                     
  |                                                                                                             
  |--> set up uport                                                                                             
  |                                                                                                             
  |    +----------------------+                                                                                 
  |--> | tty_port_link_device | save port in driver                                                             
  |    +----------------------+                                                                                 
  |    +---------------------+                                                                                  
  |--> | uart_configure_port | low-level config port, set mctrl, ensure the port console is registered          
  |    +---------------------+                                                                                  
  |    +--------------------------------------+                                                                 
  +--> | tty_port_register_device_attr_serdev | save port in driver, alloc device/cdev for tty and register them
       +--------------------------------------+                                                                 
```

```
drivers/tty/serial/serial_core.c                                                                                                    
+---------------------+                                                                                                              
| uart_configure_port | : low-level config port, set mctrl, ensure the port console is registered                                    
+-|-------------------+                                                                                                              
  |                                                                                                                                  
  |--> if port has specified boot_auto_config                                                                                        
  |     -                                                                                                                            
  |     +--> call ->config_port(), e.g.,                                                                                             
  |          +------------------------+                                                                                              
  |          | serial8250_config_port | request mem region as resource, config port type and reset uart                              
  |          +------------------------+                                                                                              
  |                                                                                                                                  
  +--> if port type isn't unknown                                                                                                    
       |                                                                                                                             
       |    +------------------+                                                                                                     
       |--> | uart_report_port | print, e.g., "1e783000.serial: ttyS0 at MMIO 0x1e783000 (irq = 32, base_baud = 1500000) is a 16550A"
       |    +------------------+                                                                                                     
       |                                                                                                                             
       |--> call ->set_mctrl(), e.g.,                                                                                                
       |    +-----------------------+                                                                                                
       |    | serial8250_set_mctrl  | set mctrl                                                                                      
       |    +-----------------------+                                                                                                
       |                                                                                                                             
       +--> if the port has an unregistered console                                                                                  
            |                                                                                                                        
            |    +------------------+                                                                                                
            +--> | register_console | add console to 'console_drivers', print out queued log                                         
                 +------------------+                                                                                                
```

```
drivers/tty/serial/8250/8250_port.c                                                        
+------------------------+                                                                  
| serial8250_config_port | : request mem region as resource, config port type and reset uart
+-|----------------------+                                                                  
  |    +---------------------------------+                                                  
  |--> | serial8250_request_std_resource | request mem region as resource                   
  |    +---------------------------------+                                                  
  |                                                                                         
  |--> if io type changes                                                                   
  |    |                                                                                    
  |    |    +------------------+                                                            
  |    +--> | set_io_from_upio | install ops for up, install ops/isr for p                  
  |         +------------------+                                                            
  |                                                                                         
  +--> if there is 'config_type' flag                                                       
       |                                                                                    
       |    +------------+                                                                  
       +--> | autoconfig | find out port type and config if needed, reset uart              
            +------------+                                                                  
```

```
drivers/tty/serial/8250/8250_port.c                                
+------------+                                                      
| autoconfig | : find out port type and config if needed, reset uart
+-|----------+                                                      
  |                                                                 
  |--> if port has no 'buggy_uart' flag                             
  |    -                                                            
  |    +--> do existence test                                       
  |                                                                 
  |--> if port has no 'skip_test' flag                              
  |    -                                                            
  |    +--> see if uart is really there?                            
  |                                                                 
  |--> find out port type and config it if needed                   
  |                                                                 
  +--> reset uart                                                   
```

```
drivers/tty/tty_port.c                                                                                    
+--------------------------------------+                                                                   
| tty_port_register_device_attr_serdev | : save port in driver, alloc device/cdev for tty and register them
+-|------------------------------------+                                                                   
  |    +----------------------+                                                                            
  |--> | tty_port_link_device | save port in driver                                                        
  |    +----------------------+                                                                            
  |    +--------------------------+                                                                        
  |--> | serdev_tty_port_register | (do nothing bc of disabled config)                                     
  |    +--------------------------+                                                                        
  |    +--------------------------+                                                                        
  +--> | tty_register_device_attr | alloc device for tty and register it, prepare its cdev as well         
       +--------------------------+                                                                        
```

```
drivers/tty/tty_io.c                                                                                    
+--------------------------+                                                                             
| tty_register_device_attr | : alloc device for tty and register it, prepare its cdev as well            
+-|------------------------+                                                                             
  |                                                                                                      
  |--> determine name based on type (pty or tty)                                                         
  |                                                                                                      
  |--> alloc device and set up (tty class)                                                               
  |                                                                                                      
  |    +-----------------+                                                                               
  |--> | device_register |                                                                               
  |    +-----------------+                                                                               
  |                                                                                                      
  +--> if driver has no 'dynamic_alloc' flag (our case)                                                  
       |                                                                                                 
       |    +--------------+                                                                             
       +--> | tty_cdev_add | prepare cdev and install ops (tty_fops), save in driver and add to framework
            +--------------+                                                                             
```

```
drivers/tty/serial/8250/8250_core.c                                                           
+------------------+                                                                           
| serial8250_probe | : for each valid port in dev data: register a 8250 port                   
+-|----------------+                                                                           
  |                                                                                            
  |--> get port(s) from device data                                                            
  |                                                                                            
  +--> for each valid port                                                                     
       |                                                                                       
       |--> given the port, set up a tmp uart port                                             
       |                                                                                       
       |    +-------------------------------+                                                  
       +--> | serial8250_register_8250_port | find matched uport from 'serial8250_ports',      
            +-------------------------------+ set it up based on arg uport, add it to framework
```

```
drivers/tty/serial/8250/8250_core.c                                                                                             
+-------------------------------+                                                                                                
| serial8250_register_8250_port | : find matched uport from 'serial8250_ports', set it up based on arg uport, add it to framework
+-|-----------------------------+                                                                                                
  |    +---------------------------------+                                                                                       
  |--> | serial8250_find_match_or_unused | try to find a matched or available uart port from 'serial8250_ports'                  
  |    +---------------------------------+                                                                                       
  |                                                                                                                              
  +--> if uart port is found                                                                                                     
       |                                                                                                                         
       |--> if the uart port has dev                                                                                             
       |    |                                                                                                                    
       |    |    +----------------------+                                                                                        
       |    +--> | uart_remove_one_port | label 'dead' on port, unregister device/cdev and console                               
       |         +----------------------+                                                                                        
       |                                                                                                                         
       |--> set up the uart_port by arg uart_port                                                                                
       |                                                                                                                         
       |    +-------------------------+                                                                                          
       |--> | serial8250_set_defaults | install default ops for up and p                                                         
       |    +-------------------------+                                                                                          
       |                                                                                                                         
       |--> install ops to the uart_port based on arg uart_port                                                                  
       |                                                                                                                         
       |    +-------------------+                                                                                                
       +--> | uart_add_one_port | config port, save port in driver, alloc device/cdev for tty and register them                  
            +-------------------+                                                                                                
```

```
drivers/tty/serial/serial_core.c                                                                      
+----------------------+                                                                               
| uart_remove_one_port | : label 'dead' on port, unregister device/cdev and console                    
+-|--------------------+                                                                               
  |                                                                                                    
  |--> set 'dead' flag to prevent other opens                                                          
  |                                                                                                    
  |    +----------------------------+                                                                  
  |--> | tty_port_unregister_device | destroy device/cdev, remove cdev from driver                     
  |    +----------------------------+                                                                  
  |    +------------------+                                                                            
  |--> | tty_port_tty_get | get tty from port                                                          
  |    +------------------+                                                                            
  |                                                                                                    
  |--> if the port is used as console                                                                  
  |    |                                                                                               
  |    |    +--------------------+                                                                     
  |    +--> | unregister_console | remove console from list, clear 'enabled' flag, print out queued log
  |         +--------------------+                                                                     
  |                                                                                                    
  |--> if ->release_port() exists, call it                                                             
  |                                                                                                    
  +--> set port_type = unknown to indicate it's not there                                              
```

```
drivers/tty/tty_port.c                                                      
+----------------------------+                                               
| tty_port_unregister_device | : destroy device/cdev, remove cdev from driver
+-|--------------------------+                                               
  |    +----------------------------+                                        
  |--> | serdev_tty_port_unregister | (do nothing bc of disabled config)     
  |    +----------------------------+                                        
  |    +-----------------------+                                             
  +--> | tty_unregister_device | destroy device/cdev, remove cdev from driver
       +-----------------------+                                             
```

```
drivers/tty/serial/8250/8250_aspeed_vuart.c                                                     
+--------------------+                                                                           
| aspeed_vuart_probe | : prepare port (ops), install handle_irq, register 8250 ports, set enabled
+-|------------------+                                                                           
  |                                                                                              
  |--> alloc vuart                                                                               
  |                                                                                              
  |    +-------------+ +-----------------------------+                                           
  |--> | timer_setup | | aspeed_vuart_unthrottle_exp | control throttling                        
  |    +-------------+ +-----------------------------+                                           
  |                                                                                              
  |--> set up port and install ops                                                               
  |                                                                                              
  |    +--------------------+                                                                    
  |--> | sysfs_create_group | create files under /sys                                            
  |    +--------------------+                                                                    
  |                                                                                              
  |--> handlel dt properties                                                                     
  |                           +-------------------------+                                        
  +--> parse irq# and install | aspeed_vuart_handle_irq |                                        
  |                           +-------------------------+                                        
  |                           rx data, and tx data if needed                                     
  |                                                                                              
  |    +-------------------------------+                                                         
  |--> | serial8250_register_8250_port | find matched uport from 'serial8250_ports',             
  |    +-------------------------------+ set it up based on arg uport, add it to framework       
  |                                                                                              
  |    +------------------------------+                                                          
  |--> | aspeed_vuart_set_lpc_address | set lpc addr in hw reg                                   
  |    +------------------------------+                                                          
  |    +-----------------------+                                                                 
  |--> | aspeed_vuart_set_sirq | set sirq in hw reg                                              
  |    +-----------------------+                                                                 
  |    +--------------------------------+                                                        
  |--> | aspeed_vuart_set_sirq_polarity | set sirq polarity in hw reg                            
  |    +--------------------------------+                                                        
  |    +--------------------------+                                                              
  |--> | aspeed_vuart_set_enabled |                                                              
  |    +--------------------------+                                                              
  |    +----------------------------------+                                                      
  +--> | aspeed_vuart_set_host_tx_discard |                                                      
       +----------------------------------+                                                      
```

```
drivers/tty/serial/8250/8250_aspeed_vuart.c
+-------------------------+
| aspeed_vuart_handle_irq | : rx data, and tx data if needed
+-|-----------------------+
  |
  |--> read reg 'lsr'
  |
  |--> if bit 'dr' or 'bi is set
  |    |
  |    |    +------------------------+
  |    |--> | tty_buffer_space_avail | get unused buffer space
  |    |    +------------------------+
  |    |
  |    |--> if no space left
  |    |    |
  |    |    |    +-----------------------------+
  |    |    |--> | __aspeed_vuart_set_throttle |
  |    |    |    +-----------------------------+
  |    |    |
  |    |    +--> schedule the timer for later unthrottle
  |    |
  |    +--> else
  |         |
  |         |--> while available
  |         |    |
  |         |    |    +----------------------+
  |         |    +--> | serial8250_read_char | serial in a char, add to uart/tty layer
  |         |         +----------------------+
  |         |    +----------------------+
  |         +--> | tty_flip_buffer_push | queue work to receive data to tty read buffer, wake up task if needed
  |              +----------------------+
  |
  +--> if transmit_hold_register is empty
       |
       |    +---------------------+
       +--> | serial8250_tx_chars | send out data from transmit buffer
            +---------------------+
```

```
drivers/tty/serial/8250/8250_port.c                                   
+----------------------+                                               
| serial8250_read_char | : serial in a char, add to uart/tty layer     
+-|--------------------+                                               
  |         +-----------+                                              
  |--> ch = | serial_in |                                              
  |         +-----------+                                              
  |    +------------------+                                            
  +--> | uart_insert_char | add ch to uart layer, further to tty buffer
       +------------------+                                            
```

```
drivers/tty/tty_buffer.c                                                                                    
+----------------------+                                                                                     
| tty_flip_buffer_push | : queue work to receive data to ldisc buffer, wake up task if needed             
+-|--------------------+                                                                                     
  |    +------------------------+                                                                            
  |--> | tty_flip_buffer_commit | (skip, not our concern)                                                    
  |    +------------------------+                                                                            
  |    +------------+ +----------------+                                                                     
  +--> | queue_work | | flush_to_ldisc | receive data to ldisc buffer till no more, wake up task if needed
       +------------+ +----------------+                                                                     
```

```
drivers/tty/tty_buffer.c                                                                             
+----------------+                                                                                    
| flush_to_ldisc | : receive data to ldisc buffer till no more, wake up task if needed             
+-|--------------+                                                                                    
  |                                                                                                   
  +--> endless loop                                                                                   
       |                                                                                              
       |--> calculate how much to read (count)                                                        
       |                                                                                              
       |    +-------------+                                                                           
       |--> | receive_buf | receive data to tty read buffer, wake up task if needed, clear data source
       |    +-------------+                                                                           
       |                                                                                              
       |--> if receive nothing, break                                                                 
       |                                                                                              
       +--> reschedule if needed                                                                      
```

```
drivers/tty/tty_buffer.c                                                                              
+-------------+                                                                                        
| receive_buf | : receive data and put to ldisc buffer, wake up task if needed, clear data source           
+-|-----------+                                                                                        
  |                                                                                                    
  |--> get ptr to the data head                                                                        
  |                                                                                                    
  |--> call ->receive_buf(), e.g.,                                                                     
  |    +------------------------------+                                                                
  |    | tty_port_default_receive_buf | receive data and put to ldisc buffer, wake up task if needed
  |    +------------------------------+                                                                
  |                                                                                                    
  +--> memset data source                                                                              
```

```
drivers/tty/tty_buffer.c                                                                         
+-----------------------+                                                                         
| tty_ldisc_receive_buf | : receive data and put to ldisc buffer, wake up task if needed
+-|---------------------+                                                                         
  |                                                                                               
  |--> if ld has ->receive_buf2()                                                                 
  |    -                                                                                          
  |    +--> call ->receive_buf2(), e.g.,                                                          
  |         +--------------------+                                                                
  |         | n_tty_receive_buf2 | receive data and put to tty read buffer, wake up task if needed
  |         +--------------------+                                                                
  |                                                                                               
  +--> elif ->receive_buf() exists                                                                
       -                                                                                          
       +--> call ->receive_buf()                                                                  
```

```
drivers/tty/n_tty.c                                                                                 
+--------------------+                                                                               
| n_tty_receive_buf2 | : receive data and put to tty read buffer, wake up task if needed             
+-|------------------+                                                                               
  |    +--------------------------+                                                                  
  +--> | n_tty_receive_buf_common | : receive data and put to tty read buffer, wake up task if needed
       +-|------------------------+                                                                  
         |                                                                                           
         |--> while flag 'changing' is not set                                                       
         |    |                                                                                      
         |    |--> min(count, room)                                                                  
         |    |                                                                                      
         |    +--> break if no count or no room                                                      
         |    |                                                                                      
         |    |    +---------------+                                                                 
         |    |--> | __receive_buf | receive data and put to tty read buffer, wake up task if needed 
         |    |    +---------------+                                                                 
         |    |                                                                                      
         |    +--> update cp/count/rcvd                                                              
         |                                                                                           
         |    +----------------------+                                                               
         +--> | n_tty_check_throttle |                                                               
              +----------------------+                                                               
```

```
drivers/tty/n_tty.c                                                                     
+---------------+                                                                        
| __receive_buf | : receive data and put to tty read buffer, wake up task if needed      
+-|-------------+                                                                        
  |--> if blabla                                                                         
  |    -    +----------------------------+                                               
  |    +--> | n_tty_receive_buf_real_raw | (skip)                                        
  |         +----------------------------+                                               
  |--> elif blabla                                                                       
  |    -    +-----------------------+                                                    
  |    +--> | n_tty_receive_buf_raw | (skip)                                             
  |         +-----------------------+                                                    
  |--> elif blabla                                                                       
  |    -    +---------------------------+                                                
  |    +--> | n_tty_receive_buf_closing | (skip)                                         
  |         +---------------------------+                                                
  |--> else                                                                              
  |    |    +----------------------------+                                               
  |    |--> | n_tty_receive_buf_standard | add each char in arg buffer to tty read buffer
  |    |    +----------------------------+                                               
  |    |    +--------------+                                                             
  |    |--> | flush_echoes |                                                             
  |    |    +--------------+                                                             
  |    +--> if ->flush_chars() exists                                                    
  |         -                                                                            
  |         +--> call it, e.g.,                                                          
  |              +------------------+                                                    
  |              | uart_flush_chars | start uart                                         
  |              +------------------+                                                    
  +--> if ldata has read cnt                                                             
       |    +-------------+                                                              
       |--> | kill_fasync | (skip)                                                       
       |    +-------------+                                                              
       |    +----------------------------+                                               
       +--> | wake_up_interruptible_poll | wake up task                                  
            +----------------------------+                                               
```

```
                                                                               
 drivers/tty/n_tty.c                                                           
+----------------------------+                                                 
| n_tty_receive_buf_standard | : add each char in arg buffer to tty read buffer
+-|--------------------------+                                                 
  |                                                                            
  +--> while count > 0                                                         
       |                                                                       
       |--> if data has next                                                   
       |    |                                                                  
       |    |    +--------------------------+                                  
       |    +--> | n_tty_receive_char_lnext | add char to tty read buffer      
       |    |    +--------------------------+                                  
       |    |                                                                  
       |    +--> continue                                                      
       |                                                                       
       |--> if c is in ldata's char_map                                        
       |    |                                                                  
       |    |    +----------------------------+                                
       |    +--> | n_tty_receive_char_special | (skip)                         
       |         +----------------------------+                                
       |                                                                       
       +--> else                                                               
            |                                                                  
            |    +--------------------+                                        
            +--> | n_tty_receive_char | add char to tty read buffer            
                 +--------------------+                                        
```

```
drivers/tty/n_tty.c                                       
+--------------------------+                               
| n_tty_receive_char_lnext | : add char to tty read buffer 
+-|------------------------+                               
  |    +--------------------+                              
  +--> | n_tty_receive_char | : add char to tty read buffer
       +-|------------------+                              
         |                                                 
         |--> other handling                               
         |                                                 
         |    +---------------+                            
         +--> | put_tty_queue | add char to tty read buffer
              +---------------+                            
```

```
drivers/tty/serial/8250/8250_port.c                         
+---------------------+                                      
| serial8250_tx_chars | : send out data from transmit buffer 
+-|-------------------+                                      
  |                                                          
  +--> while --count > 0                                     
       |                                                     
       |    +------------+                                   
       +--> | serial_out | send out data from transmit buffer
            +------------+                                   
```

```
drivers/tty/serial/8250/8250_of.c                                                        
+--------------------------+                                                              
| of_platform_serial_probe | : get iomem/irq, prepare info, register 8250 ports           
+-|------------------------+                                                              
  |    +--------------------------+                                                       
  |--> | of_platform_serial_setup | handle dt properties, get iomem/irq, do reset control 
  |    +--------------------------+                                                       
  |                                                                                       
  |--> handle dt properties                                                               
  |                                                                                       
  |--> alloc 'info'                                                                       
  |                                                                                       
  |    +-------------------------------+                                                  
  |--> | serial8250_register_8250_port | find matched uport from 'serial8250_ports',      
  |    +-------------------------------+ set it up based on arg uport, add it to framework
  |                                                                                       
  +--> save port_type and line# in 'info'                                                 
```

```
drivers/tty/serial/8250/8250_of.c                                                     
+--------------------------+                                                           
| of_platform_serial_setup | : handle dt properties, get iomem/irq, do reset control   
+-|------------------------+                                                           
  |                                                                                    
  |--> handle dt properties                                                            
  |                                                                                    
  |--> get iomem resource from dt and save in port                                     
  |                                                                                    
  |    +------------+                                                                  
  |--> | of_irq_get | get hwirq, prepare virq, create mapping of them and add to domain
  |    +------------+                                                                  
  |    +----------------------------------------+                                      
  |--> | devm_reset_control_get_optional_shared | (reset control, skip)                
  |    +----------------------------------------+                                      
  |    +------------------------+                                                      
  +--> | reset_control_deassert | (reset control, skip)                                
       +------------------------+                                                      
```

```
drivers/usb/serial/usb-serial.c
+-----------------+                                                    
| usb_serial_init | : prepare tty driver, register bus 'usb-serial'    
+-|---------------+                                                    
  |    +------------------+                                            
  |--> | tty_alloc_driver | prepare tty driver                         
  |    +------------------+                                            
  |                                                                    
  |--> assign to 'usb_serial_tty_driver'                               
  |                                                                    
  |    +--------------+                                                
  |--> | bus_register | 'usb_serial_bus_type'                          
  |    +--------------+                                                
  |                                                                    
  |--> further set up 'usb_serial_tty_driver'                          
  |                                                                    
  |    +--------------------+                                          
  |--> | tty_set_operations | 'serial_ops'                             
  |    +--------------------+                                          
  |    +-----------------------------+                                 
  +--> | usb_serial_generic_register | do nothing bc of disabled config
       +-----------------------------+                                 
```

```
serial/pl2303.c                                                                                                    
+------------------------+                                                                                           
| usb_serial_module_init | : register usb intf driver, register arg drivers to bus 'usb-serial'                      
+-|----------------------+                                                                                           
  |    +-----------------------------+                                                                               
  +--> | usb_serial_register_drivers | : register usb intf driver, register arg drivers to bus 'usb-serial'          
       +-|---------------------------+                                                                               
         |                                                                                                           
         |--> alloc usb_driver                                                                                       
         |                                                                                                           
         |--> set up driver and install ops                                                                          
         |                                                                                                           
         |    +--------------+                                                                                       
         |--> | usb_register | register usb interface driver                                                         
         |    +--------------+                                                                                       
         |                                                                                                           
         |--> for each driver in arg 'serial_drivers'                                                                
         |                                                                                                           
         |------> save the newly allocated usb_driver ptr                                                            
         |                                                                                                           
         |        +---------------------+                                                                            
         |------> | usb_serial_register | add driver to 'usb_serial_driver_list', register driver to bus 'usb-serial'
         |        +---------------------+                                                                            
         |                                                                                                           
         |--> set id_table for match                                                                                 
         |                                                                                                           
         |    +---------------+                                                                                      
         +--> | driver_attach | try match device and probe (but I don't see the 'bus' set anywhere)                  
              +---------------+                                                                                      
```

```
drivers/usb/serial/usb-serial.c
+---------------------+                                                                              
| usb_serial_register | : add driver to 'usb_serial_driver_list', register driver to bus 'usb-serial'
+-|-------------------+                                                                              
  |    +----------------------------+                                                                
  |--> | usb_serial_operations_init | reset some fields                                              
  |    +----------------------------+                                                                
  |                                                                                                  
  |--> add driver to 'usb_serial_driver_list'                                                        
  |                                                                                                  
  |    +-------------------------+                                                                   
  +--> | usb_serial_bus_register | set driver bus type to 'usb-serial', register driver              
       +-------------------------+                                                                   
```

```
drivers/net/netconsole.c                                                              
+-----------------+                                                                    
| init_netconsole | : register netdevice notifer, register net console                 
+-|---------------+                                                                    
  |                                                                                    
  |--> handle config (it's empty in our case)                                          
  |                                                                                    
  |    +-----------------------------+                                                 
  |--> | register_netdevice_notifier | .notifier_call  = netconsole_netdev_event       
  |    +-----------------------------+                                                 
  |    +-------------------------+                                                     
  |--> | dynamic_netconsole_init | (do nothing bc of disabled config)                  
  |    +-------------------------+                                                     
  |                                                                                    
  |--> if netconsole_ext has specified 'enabled' (not our case)                        
  |    |                                                                               
  |    |    +------------------+                                                       
  |    +--> | register_console | add console to 'console_drivers', print out queued log
  |         +------------------+ (console = external net console)                      
  |    +------------------+                                                            
  |--> | register_console | add console to 'console_drivers', print out queued log     
  |    +------------------+ (console = net console)                                    
  |                                                                                    
  +--> print "netconsole: network logging started"                                     
```

```
drivers/net/netconsole.c                               
+-------------------------+                             
| netconsole_netdev_event |                             
+-|-----------------------+                             
  |                                                     
  |--> for each entry on target_list (empty in our case)
  |    -                                                
  |    +--> (skip)                                      
  |                                                     
  +--> if stopped (false in our case)                   
       -                                                
       +--> (skip)                                      
```

```
drivers/tty/serial/8250/8250_core.c                                              
+-----------------------+                                                         
| univ8250_console_init | : init serial8250 ports, register console (univ8250)    
+-|---------------------+                                                         
  |    +---------------------------+                                              
  |--> | serial8250_isa_init_ports | init each port in 'serial8250_ports'         
  |    +---------------------------+                                              
  |    +------------------+                                                       
  +--> | register_console | add console to 'console_drivers', print out queued log
       +------------------+ (console = univ8250_console)                          
```






Here we list a few functions that are related to our topic and we'll introduce them one by one.
```
init calls
  └─ of_platform_default_populate_ini()
  └─ of_platform_serial_driver_init()
```
  
- of_platform_serial_driver_init

  It registers driver 'of_serial' and probes device '1e783000.serial' & '1e784000.serial' sequentially. Here we have to introduce the 'port' concept of uart.
  Take AST2500 for example, it might be equipped with up to 5 regular UARTs (leave virtual UART alone).
  Each UART component is regarded as 'port' in the code, e.g. ttyS4 is the port 5 of UART.
  Back to the probe, it registers the UART port to the framework for each of the matched devices.

  ```
  of_platform_serial_driver_init()
    └─ register driver 'of_serial'
         └─ probe device 1e783000.serial
              └─ set up the UART port and register it
              └─ register console but fail (it's not our preferred console, which is set by kernel boot command)
         └─ probe device 1e784000.serial
              └─ set up the UART port and register it              
              └─ register console and pass (preferred console)
  ```
  
  Registered consoles are for kernel space to print out messages.
  
## <a name="mechanism-of-printk"></a> Mechanism of printk()

Unlike user space processes have TTY device to direct input/output/error, kernel threads output messages by printk and below flowchart shows how it works.

```mermaid
graph TD
   a(printk)
   b(vprintk_func)
   c(vsnprintf)
   d(log_output)
   e(console_unlock)
   
   a-->b
   b-->c 
   c-->d
   d-->e
```

- printk(), aggregate arguments
- vprintk_func(), deal with different contexts, e.g. NMI, safe, ...
- vsnprintf(), replace specifiers with real data
- log_output(), commit string to the circular buffer of printk as 'record'
- console_unlock(), for each valid record, add prefix and write to each console

```
console_unlock()
  └─ record_print_text(), insert timestamp to the beginning of string
  └─ call_console_drivers(), write out the committed record to each registered console
```

So, before registering any console, printk() can still work since it just commits the message into the ring buffer.
Once there's an available console, all pending records are handled sequentially.

## <a name="kernel-boot-arguments"></a> Kernel Boot Arguments

## <a name="conclusion"></a> Conclusion

User space tasks interact with TTY devices to control message direction, 
and the underlying destination can be local desktop application, remote machine, or serial port. 
Meanwhile, logs emitted by kernel space threads or services are committed to the circular buffer, 
which is further written out to each valid console if there's any.

There are still other interesting topics worth digging into, such as pseudo TTY and serial over LAN. Hope this note helps, thanks!

```
grep '0x' /sys/class/tty/ttyS*/iomem_base  
```

[What are TTY, serial, and UART lines?](https://subscription.packtpub.com/book/hardware-&-creative/9781786461803/7/ch07lvl1sec37/what-are-tty-serial-and-uart-lines)


