> Study case: Linux version 5.15.0 on OpenBMC

## Index

- [Introduction](#introduction)
- [Driver](#driver)
- [Pin Control](#pin-control)
- [System Startup](#system-startup)
- [Cheat Sheet](#cheat-sheet)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

Generic purpose input-output (GPIO) can work as output to control target device or as input to receive signals from outside. 
In modern design, these pins are usually multi-functional. One pin can be an SDA in SMBus protocol or a TX in UART with the proper configuration. 
Pin control (pinctrl) is the mechanism implemented in the kernel to properly configure those pins when we expect them to perform a specific function. 

## <a name="driver"></a> Driver



| GPIO | Line | Name           | Use            | Direction | Note |
| ---  | ---  | ---            | ---            | ---       | ---  |
| A4   | 4    |                | kernel         | input     |      |
| A5   | 5    |                | kernel         | input     |      |
| C0   | 16   |                | kernel         | input     |      |
| C1   | 17   |                | kernel         | input     |      |
| C2   | 18   |                | kernel         | input     |      |
| C3   | 19   |                | kernel         | input     |      |
| C4   | 20   |                | kernel         | input     |      |
| C5   | 21   |                | kernel         | input     |      |
| C6   | 22   |                | kernel         | input     |      |
| C7   | 23   |                | kernel         | input     |      |
| D3   | 27   | nic_func_mode0 | nic_func_mode0 | output    |      |
| D4   | 28   | nic_func_mode1 | nic_func_mode1 | output    |      |
| I3   | 67   | power-button   | sysfs          | input     |      |
| I4   | 68   |                | kernel         | input     |      |
| I5   | 69   |                | kernel         | input     |      |
| I6   | 70   |                | kernel         | input     |      |
| I7   | 71   |                | kernel         | input     |      |
| J1   | 73   |                | sysfs          | input     |      |
| J2   | 74   | checkstop      | checkstop      | input     |      |
| K0   | 80   |                | kernel         | input     |      |
| K1   | 81   |                | kernel         | input     |      |
| K2   | 82   |                | kernel         | input     |      |
| K3   | 83   |                | kernel         | input     |      |
| K4   | 84   |                | kernel         | input     |      |
| K5   | 85   |                | kernel         | input     |      |
| K6   | 86   |                | kernel         | input     |      |
| K7   | 87   |                | kernel         | input     |      |
| L0   | 88   |                | kernel         | input     |      |
| L1   | 89   |                | kernel         | input     |      |
| L2   | 90   |                | kernel         | input     |      |
| L3   | 91   |                | kernel         | input     |      |
| L4   | 92   |                | kernel         | input     |      |
| L5   | 93   |                | kernel         | input     |      |
| L6   | 94   |                | kernel         | input     |      |
| L7   | 95   |                | kernel         | input     |      |
| N0   | 104  |                | kernel         | input     |      |
| N1   | 105  |                | kernel         | input     |      |
| N2   | 106  | led-fault      | fault          | output    |      |
| N4   | 108  | led-identify   | identify       | output    |      |
| Q0   | 128  |                | kernel         | input     |      |
| Q1   | 129  |                | kernel         | input     |      |
| Q2   | 130  |                | kernel         | input     |      |
| Q3   | 131  |                | kernel         | input     |      |
| Q7   | 135  | id-button      | id-button      | input     |      |
| R3   | 139  |                | phosphor-hwmon | output    |      |
| R5   | 141  | led-power      | power          | output    |      |
| S7   | 151  | seq_cont       | seq_cont       | output    |      |
| T0   | 152  |                | kernel         | input     |      |
| T1   | 153  |                | kernel         | input     |      |
| T2   | 154  |                | kernel         | input     |      |
| T3   | 155  |                | kernel         | input     |      |
| U4   | 164  |                | kernel         | input     |      |
| U6   | 166  |                | kernel         | input     |      |
| U7   | 167  |                | kernel         | input     |      |
| V0   | 168  |                | kernel         | input     |      |
| V1   | 169  |                | kernel         | input     |      |


The 'GPIO Lib' framework in kernel space bridges the userspace requests and the real GPIO chip driver provided by the vendor, e.g., Aspeed.

```
                                                +--
                                                |   static const struct file_operations gpio_fileops = {
                                                |       .release = gpio_chrdev_release,
                                                |       .open = gpio_chrdev_open,
  user             +------+ e.g. /dev/gpiochip0 |       .poll = lineinfo_watch_poll,
  -----------------| cdev |------------------   |       .read = lineinfo_watch_read,
  kernel           +------+                     |       .owner = THIS_MODULE,
                       |                        |       .llseek = no_llseek,
                       v                        |       .unlocked_ioctl = gpio_ioctl,
             |--  +---------+                   |   };
    generic  |    | gpiolib |                   +--
             +--  +---------+
                       |
                       v
             |--  +--------+
   specific  |    | driver |
e.g. aspeed  +--  +--------+                                                                              
```

Nothing worths mentioning from the function **gpiolib_dev_init** in our study case. 
Just note that once **gpiolib_initialized** becomes true, the subsequent GPIO chip driver registration will set up the cdev file operations during initialization.

After kernel adds GPIO device based on device tree and initializes the GPIO driver, probe function triggers because the property **compatible** matches.

- Device descriptor in DTS

```
gpio@1e780000 {                  
    #gpio-cells = <0x02>;                  
    gpio-controller;                  
    compatible = "aspeed,ast2500-gpio";     <========== match
    reg = <0x1e780000 0x200>;                  
    interrupts = <0x14>;                  
    gpio-ranges = <0x0d 0x00 0x00 0xe8>;    <========== interpret as <pinctrl_phandle gpio_offset pin_offset pin#>
    clocks = <0x02 0x1a>;                  
    interrupt-controller;                  
    #interrupt-cells = <0x02>;                  
    gpio-line-names = [...
                       63 66 61 6d 2d 72 65 73 65 74 00 \               <========== GPIO_A0, "cfam-reset"
                       ... \
                       66 73 69 2d 6d 75 78 00 \                        <========== GPIO_A6, "fsi-mux"
                       ... \
                       66 73 69 2d 65 6e 61 62 6c 65 00 \               <========== GPIO_D0, "fsi-enable"
                       ... \
                       6e 69 63 5f 66 75 6e 63 5f 6d 6f 64 65 30 00 \   <========== GPIO_D3, "nic_func_mode0"
                       6e 69 63 5f 66 75 6e 63 5f 6d 6f 64 65 31 00 \   <========== GPIO_D4, "nic_func_mode1"
                       ... \
                       ... \
                       70 6f 77 65 72 2d 62 75 74 74 6f 6e 00 \         <========== GPIO_I3, "power-button"
                       ... \
                       63 68 65 63 6b 73 74 6f 70 00 \                  <========== GPIO_J2, "checkstop"
                       ... \
                       6c 65 64 2d 66 61 75 6c 74 00 \                  <========== GPIO_N2, "led-fault"
                       ... \
                       6c 65 64 2d 69 64 65 6e 74 69 66 79 00 \         <========== GPIO_N4, "led-identify"
                       ... \
                       69 64 2d 62 75 74 74 6f 6e 00 \                  <========== GPIO_Q7, "id-button"
                       ... \
                       66 73 69 2d 74 72 61 6e 73 00 \                  <========== GPIO_R2, "fsi-trans"
                       ... \
                       6c 65 64 2d 70 6f 77 65 72 00 \                  <========== GPIO_R5, "led-power"
                       ... \
                       73 65 71 5f 63 6f 6e 74 00 \                     <========== GPIO_S7, "seq_cont"
                       ... \
                       66 73 69 2d 63 6c 6f 63 6b 00 \                  <========== GPIO_AA0, "fsi-clock"
                       ... \
                       66 73 69 2d 64 61 74 61 00 \                     <========== GPIO_AA2, "fsi-data"
                       ...];
    phandle = <0x29>; 
```

- Table in driver

```
static const struct of_device_id aspeed_gpio_of_table[] = {
    { .compatible = "aspeed,ast2400-gpio", .data = &ast2400_config, },
    { .compatible = "aspeed,ast2500-gpio", .data = &ast2500_config, }, <========== match
    { .compatible = "aspeed,ast2600-gpio", .data = &ast2600_config, },
    {}
};
```

The probe function allocates vendor-specific structure, installs regular GPIO operations, and sets up the IRQ chip since it's also an interrupt controller. 
Then it calls **devm_gpiochip_add_data** to connect the generic layer (GPIO Lib) to the specific driver (Aspeed).

At the moment, we have the complete GPIO framework, and userspace utilities can send requests through the char device.

```
root@romulus:~# gpiodetect
gpiochip0 [1e780000.gpio] (232 lines)

root@romulus:~# gpioinfo 0 | head
gpiochip0 - 232 lines:
        line   0:      unnamed       unused   input  active-high 
        line   1: "cfam-reset"       unused   input  active-high 
        line   2:      unnamed       unused   input  active-high 
        line   3:      unnamed       unused   input  active-high 
        line   4:      unnamed       kernel   input  active-high [used]
        line   5:      unnamed       kernel   input  active-high [used]
        line   6:    "fsi-mux"       unused  output  active-high 
        line   7:      unnamed       unused   input  active-high 
        line   8:      unnamed       unused   input  active-high 

root@romulus:~# gpioget 0 2     # get value from gpio chip '0' line '2'
0

root@romulus:~# gpioset 0 2=1   # set value to gpio chip '0' line '2' to '1'

root@romulus:~# gpioget 0 2     # get value from gpio chip '0' line '2'
1

root@romulus:~# gpiomon 0 2     # monitor value change on gpio chip '0' line '2'
```

Here's the example of a flow chart showing how the code switches from generic layer (GPIO Lib) to chip vendor (Aspeed) implementation.

```
                                            |                                   
                                   gpio lib | aspeed driver                     
                                            |                                   
+----------------------+                    |                                   
| gpiod_request_commit |                    |                                   
+-----|----------------+                    |                                   
      |                                     |                                   
      |--> gc->request() -------------------|--- e.g., aspeed_gpio_request      
      |                                     |                                   
      |    +---------------------+          |                                   
      +--> | gpiod_get_direction |          |                                   
           +-----|---------------+          |                                   
                 |                          |                                   
                 +--> gc->get_direction() --|--- e.g., aspeed_gpio_get_direction
                                            |                                   
```

```
drivers/gpio/gpiolib-cdev.c

static const struct file_operations gpio_fileops = {
    .release = gpio_chrdev_release,
    .open = gpio_chrdev_open,
    .poll = lineinfo_watch_poll,
    .read = lineinfo_watch_read,
    .owner = THIS_MODULE,
    .llseek = no_llseek,
    .unlocked_ioctl = gpio_ioctl,
#ifdef CONFIG_COMPAT
    .compat_ioctl = gpio_ioctl_compat,
#endif
};
```

```
drivers/gpio/gpiolib-cdev.c                                                             
+------------------+                                                                     
| gpio_chrdev_open | : prepare notifier & register to chain, set up file priv & open_mode
+-|----------------+                                                                     
  |                                                                                      
  |--> alloc bitmap to represent all gpio lines                                          
  |                                                                                      
  |                     +-------------------------+                                      
  |--> prepare callback | lineinfo_changed_notify | set up chg_info, wake up waiting task
  |                     +-------------------------+                                      
  |    +----------------------------------+                                              
  |--> | blocking_notifier_chain_register | register to notifier chain of gpio_dev       
  |    +----------------------------------+                                              
  |                                                                                      
  |--> save private data in file                                                         
  |                                                                                      
  |    +------------------+                                                              
  +--> | nonseekable_open | set file open_mode                                           
       +------------------+                                                              
```

```
drivers/gpio/gpiolib-cdev.c                                       
+-------------------------+                                        
| lineinfo_changed_notify | : set up chg_info, wake up waiting task
+-|-----------------------+                                        
  |                                                                
  |--> return if the gpio_desc isn't one of our targets            
  |                                                                
  |--> set up chg_info                                             
  |                                                                
  |    +-----------------------+                                   
  |--> | gpio_desc_to_lineinfo | set up info based on gpip_desc    
  |    +-----------------------+                                   
  |    +--------------+                                            
  +--> | wake_up_poll |                                            
       +--------------+                                            
```

```
drivers/gpio/gpiolib-cdev.c                                               
+-----------------------+                                                  
| gpio_desc_to_lineinfo | : set up info based on gpip_desc                 
+-|---------------------+                                                  
  |    +---------------------------+                                       
  |--> | pinctrl_gpio_can_use_line | check if gpio pin is available for use
  |    +---------------------------+                                       
  |                                                                        
  |--> set up info .name and .consumer (label)                             
  |                                                                        
  +--> set up info.flags based on desc.flags                               
```

```
drivers/gpio/gpiolib-cdev.c                                   
+---------------------+                                        
| lineinfo_watch_read | : read data from kfifo and copy to user
+-|-------------------+                                        
  |                                                            
  +--> while we still have something to read                   
       |                                                       
       |--> if kfifo is empty                                  
       |    -                                                  
       |    +--> determine to return or wait event             
       |                                                       
       |--> determine event_size                               
       |                                                       
       |    +-----------+                                      
       |--> | kfifo_out | get data from fifo                   
       |    +-----------+                                      
       |    +--------------+                                   
       |--> | copy_to_user |                                   
       |    +--------------+                                   
       |                                                       
       +--> bytes_read += event_size                           
```

```
drivers/gpio/gpiolib-cdev.c                                                                                                     
+------------+                                                                                                                   
| gpio_ioctl | : ioctl for gpio operation                                                                                        
+-|----------+                                                                                                                   
  |                                                                                                                              
  |--> switch cmd                                                                                                                
  |--> case get_chip_info                                                                                                        
  |    -    +--------------+                                                                                                     
  |    +--> | chipinfo_get | prepare chip_info and copy to user                                                                  
  |         +--------------+                                                                                                     
  |--> case get_line_handle                                                                                                      
  |    -    +-------------------+                                                                                                
  |    +--> | linehandle_create | create line handle, prepare fd/file with private = line_handle, coyp fd info to user           
  |         +-------------------+                                                                                                
  |--> case get_line_event                                                                                                       
  |    -    +------------------+                                                                                                 
  |    +--> | lineevent_create | create line event, set gpio input, request threaded irq, prepare fd/file for le, copy fd to user
  |         +------------------+                                                                                                 
  |--> case get_line_info || get_line_info_watch                                                                                 
  |    -    +-----------------+                                                                                                  
  |    +--> | lineinfo_get_v1 | fill line_info based on gpip_desc, copy line_info to user                                        
  |         +-----------------+                                                                                                  
  |--> case v2_get_line_info || v2_get_line_info_watch                                                                           
  |    -    +--------------+                                                                                                     
  |    +--> | lineinfo_get | fill line_info based on gpip_desc, copy line_info to user                                           
  |         +--------------+                                                                                                     
  |--> case get_line                                                                                                             
  |    -    +----------------+                                                                                                   
  |    +--> | linereq_create | for each line: request gpio and set direction, prepare fd/file, copy fd to user                   
  |         +----------------+                                                                                                   
  +--> case get_line_info_unwatch                                                                                                
       -    +------------------+                                                                                                 
       +--> | lineinfo_unwatch | copy data from user, clear target bit from bitmap                                               
            +------------------+                                                                                                 
```

```
drivers/gpio/gpiolib-cdev.c                                                                                
+-------------------+                                                                                       
| linehandle_create | : create line handle, prepare fd/file with private = line_handle, coyp fd info to user
+-|-----------------+                                                                                       
  |    +----------------+                                                                                   
  |--> | copy_from_user | copy handle_request from user                                                     
  |    +----------------+                                                                                   
  |    +---------------------------+                                                                        
  |--> | linehandle_validate_flags | validate flags                                                         
  |    +---------------------------+                                                                        
  |                                                                                                         
  |--> alloc line_handle, and set up from handle_request                                                    
  |                                                                                                         
  |--> for each line                                                                                        
  |    |                                                                                                    
  |    |    +-------------------+                                                                           
  |    |--> | gpiochip_get_desc | get gpio_desc                                                             
  |    |    +-------------------+                                                                           
  |    |    +--------------------+                                                                          
  |    |--> | gpiod_request_user | set label, call ->request() and ->get_direction() if feasible            
  |    |    +--------------------+                                                                          
  |    |                                                                                                    
  |    |--> save gpio_desc in line_handle                                                                   
  |    |                                                                                                    
  |    |    +--------------------------------+                                                              
  |    |--> | linehandle_flags_to_desc_flags | given handle_request, set gpio_desc.flags                    
  |    |    +--------------------------------+                                                              
  |    |    +----------------------+                                                                        
  |    |--> | gpiod_set_transitory | label 'transitory' in desc, set config                                 
  |    |    +----------------------+                                                                        
  |    |                                                                                                    
  |    |--> set gpio direction accordingly                                                                  
  |    |                                                                                                    
  |    |    +------------------------------+                                                                
  |    +--> | blocking_notifier_call_chain | call notifier for 'line_changed_requested'                     
  |         +------------------------------+                                                                
  |    +---------------------+                                                                              
  |--> | get_unused_fd_flags | get an available fd                                                          
  |    +---------------------+                                                                              
  |    +--------------------+                                                                               
  |--> | anon_inode_getfile | prepare a file (name = gpio-linehandle, fops = linehandle_fileops)            
  |    +--------------------+                                                                               
  |                                                                                                         
  |--> save fd in handle_request                                                                            
  |                                                                                                         
  |    +--------------+                                                                                     
  |--> | copy_to_user | copy handle_request back to user                                                    
  |    +--------------+                                                                                     
  |    +------------+                                                                                       
  +--> | fd_install | table[fd] = fill                                                                      
       +------------+                                                                                       
```

```
drivers/gpio/gpiolib-cdev.c                                                                                           
+------------------+                                                                                                   
| lineevent_create | : create line event, set gpio input, request threaded irq, prepare fd/file for le, copy fd to user
+-|----------------+                                                                                                   
  |    +----------------+                                                                                              
  |--> | copy_from_user | copy event_req from user                                                                     
  |    +----------------+                                                                                              
  |    +-------------------+                                                                                           
  |--> | gpiochip_get_desc | get gpio_desc                                                                             
  |    +-------------------+                                                                                           
  |                                                                                                                    
  |--> alloc line_event                                                                                                
  |                                                                                                                    
  |    +--------------------+                                                                                          
  |--> | gpiod_request_user | set label, call ->request() and ->get_direction() if feasible                            
  |    +--------------------+                                                                                          
  |                                                                                                                    
  |--> save gpio_desc in line_event                                                                                    
  |                                                                                                                    
  |    +--------------------------------+                                                                              
  |--> | linehandle_flags_to_desc_flags | given handle_request, set gpio_desc.flags                                    
  |    +--------------------------------+                                                                              
  |    +-----------------------+                                                                                       
  |--> | gpiod_direction_input | set direction input                                                                   
  |    +-----------------------+                                                                                       
  |                                                                                                                    
  |--> get irq from gpio_desc, save in line_event                                                                      
  |                                                                                                                    
  |    +----------------------+                                                                                        
  |--> | request_threaded_irq | request a thread to read the events                                                    
  |    +----------------------+                                                                                        
  |    +---------------------+                                                                                         
  |--> | get_unused_fd_flags | get an available fd                                                                     
  |    +---------------------+                                                                                         
  |    +--------------------+                                                                                          
  |--> | anon_inode_getfile | prepare a file (name = gpio-event, fops = lineevent_fileops)                             
  |    +--------------------+                                                                                          
  |                                                                                                                    
  |--> save fd in event_req                                                                                            
  |                                                                                                                    
  |    +--------------+                                                                                                
  |--> | copy_to_user | copy event_req back to user                                                                    
  |    +--------------+                                                                                                
  |    +------------+                                                                                                  
  +--> | fd_install |                                                                                                  
       +------------+                                                                                                  
```

```
drivers/gpio/gpiolib-cdev.c                                                   
+-----------------+                                                            
| lineinfo_get_v1 | : fill line_info based on gpip_desc, copy line_info to user
+-|---------------+                                                            
  |    +----------------+                                                      
  |--> | copy_from_user | copy line info from user                             
  |    +----------------+                                                      
  |    +-------------------+                                                   
  |--> | gpiochip_get_desc |                                                   
  |    +-------------------+                                                   
  |                                                                            
  |--> get gpio_desc accordingly                                               
  |                                                                            
  |--> if arg 'watch' is set                                                   
  |    -                                                                       
  |    +--> return error if that gpio line is already watched                  
  |                                                                            
  |    +-----------------------+                                               
  |--> | gpio_desc_to_lineinfo | set up line_info_v2 based on gpio_desc        
  |    +-----------------------+                                               
  |    +-------------------------+                                             
  |--> | gpio_v2_line_info_to_v1 | set up line_info based on line_info_v2      
  |    +-------------------------+                                             
  |    +--------------+                                                        
  +--> | copy_to_user | coyp line_info to user                                 
       +--------------+                                                        
```

```
drivers/gpio/gpiolib-cdev.c                                                
+--------------+                                                            
| lineinfo_get | : fill line_info based on gpip_desc, copy line_info to user
+-|------------+                                                            
  |    +----------------+                                                   
  |--> | copy_from_user | copy line info from user                          
  |    +----------------+                                                   
  |    +-------------------+                                                
  |--> | gpiochip_get_desc | get gpio_desc                                  
  |    +-------------------+                                                
  |                                                                         
  |--> if arg 'watch' is set                                                
  |    -                                                                    
  |    +--> return error if that gpio line is already watched               
  |                                                                         
  |    +-----------------------+                                            
  |--> | gpio_desc_to_lineinfo | set up line_info_v2 based on gpio_desc     
  |    +-----------------------+                                            
  |    +--------------+                                                     
  +--> | copy_to_user | copy line info to user                              
       +--------------+                                                     
```

```
drivers/gpio/gpiolib-cdev.c                                                                        
+----------------+                                                                                  
| linereq_create | : for each line: request gpio and set direction, prepare fd/file, copy fd to user
+-|--------------+                                                                                  
  |    +----------------+                                                                           
  |--> | copy_from_user | copy user_line_req from user                                              
  |    +----------------+                                                                           
  |    +------------------------------+                                                             
  |--> | gpio_v2_line_config_validate | validate line config                                        
  |    +------------------------------+                                                             
  |                                                                                                 
  |--> alloc line_req                                                                               
  |                                                                                                 
  |--> for each line                                                                                
  |    |                                                                                            
  |    |    +-------------------+ +--------------------+                                            
  |    +--> | INIT_DELAYED_WORK | | debounce_work_func |                                            
  |         +-------------------+ +--------------------+                                            
  |                                                                                                 
  |--> determine line_req.buffer_size                                                               
  |                                                                                                 
  |--> for each line                                                                                
  |    |                                                                                            
  |    |    +-------------------+                                                                   
  |    |--> | gpiochip_get_desc | get gpio_desc                                                     
  |    |    +-------------------+                                                                   
  |    |    +--------------------+                                                                  
  |    |--> | gpiod_request_user | set label, call ->request() and ->get_direction() if feasible    
  |    |    +--------------------+                                                                  
  |    |    +----------------------+                                                                
  |    |--> | gpiod_set_transitory | label 'transitory' in desc, set config                         
  |    |    +----------------------+                                                                
  |    |                                                                                            
  |    |--> if flag specified 'output'                                                              
  |    |    |    +----------------------------------+                                               
  |    |    |--> | gpio_v2_line_config_output_value | get value for output                          
  |    |    |    +----------------------------------+                                               
  |    |    |    +------------------------+                                                         
  |    |    +--> | gpiod_direction_output | set direction 'output'                                  
  |    |         +------------------------+                                                         
  |    +--> else                                                                                    
  |         |    +-----------------------+                                                          
  |         |--> | gpiod_direction_input | set direction 'input'                                    
  |         |    +-----------------------+                                                          
  |         |    +---------------------+                                                            
  |         +--> | edge_detector_setup | request threaded irq                                       
  |              +---------------------+                                                            
  |                                                                                                 
  |--> prepare fd/file                                                                              
  |                                                                                                 
  +--> copy fd to user                                                                              
```

### libgpiod

```
bindings/cxx/line.cpp                                                                           
+------------------+                                                                             
| gpiod::find_line | : find name-matched line                                                    
+-|----------------+                                                                             
  |                     +-----------------------+                                                
  +--> for each iter in | gpiod::make_chip_iter |                                                
       |                +-----------------------+                                                
       |                prepare a iterator of that many gpio chips, get chip info and fill struct
       |                                                                                         
       |    +-----------------+                                                                  
       +--> | chip::find_line | find name-matched line                                           
            +-----------------+                                                                  
```

```
bindings/cxx/iter.cpp                                                                                  
+-----------------------+                                                                               
| gpiod::make_chip_iter | : prepare a iterator of that many gpio chips, get chip info and fill struct   
+-|---------------------+                                                                               
  |    +---------------------+                                                                          
  |--> | gpiod_chip_iter_new | prepare a iterator of that many gpio chips, get chip info and fill struct
  |    +---------------------+                                                                          
  |                                                                                                     
  +--> convert to chip iterator ?                                                                       
```

```
lib/iter.c                                                                                        
+---------------------+                                                                            
| gpiod_chip_iter_new | : prepare a iterator of that many gpio chips, get chip info and fill struct
+-|-------------------+                                                                            
  |                                                                                                
  |--> find /dev/gpiochip*                                                                         
  |                                                                                                
  |--> alloc iterator                                                                              
  |                                                                                                
  |--> alloc that many chip struct and save in iterator                                            
  |                                                                                                
  +--> for each gpio_chip                                                                          
       |                                                                                           
       |    +-------------------------+                                                            
       +--> | gpiod_chip_open_by_name | open file & get chip info, prepare such chip struct        
            +-------------------------+                                                            
```

```
+-------------------------+                                                      
| gpiod_chip_open_by_name | : open file & get chip info, prepare such chip struct
+-|-----------------------+                                                      
  |                                                                              
  |--> prepare file name                                                         
  |                                                                              
  |    +-----------------+                                                       
  +--> | gpiod_chip_open | open file & get chip info, prepare such chip struct   
       +-----------------+                                                       
```

```
lib/core.c                                                              
+-----------------+                                                      
| gpiod_chip_open | : open file & get chip info, prepare such chip struct
+-|---------------+                                                      
  |    +------+                                                          
  |--> | open |                                                          
  |    +------+                                                          
  |    +------------------+                                              
  |--> | is_gpiochip_cdev | check if it's valid                          
  |    +------------------+                                              
  |                                                                      
  |--> alloc chip                                                        
  |                                                                      
  |    +-------+                                                         
  |--> | ioctl | opt = get_chip_info                                     
  |    +-------+                                                         
  |                                                                      
  +--> set up chip (fd, lines, name, label)                              
```

```
bindings/cxx/chip.cpp                                
+-----------------+                                   
| chip::find_line | : find name-matched line          
+-|---------------+                                   
  |    +----------------------+                       
  +--> | gpiod_chip_find_line | find name-matched line
       +----------------------+                       
```

```
lib/helpers.c                                                              
+----------------------+                                                    
| gpiod_chip_find_line | : find name-matched line                           
+-|--------------------+                                                    
  |    +---------------------+                                              
  |--> | gpiod_line_iter_new | prepare line_iter (have each line info ready)
  |    +---------------------+                                              
  |                                                                         
  +--> for each line                                                        
       |                                                                    
       |--> get line name and compare with arg name                         
       |                                                                    
       +--> if found                                                        
            |                                                               
            |    +----------------------+                                   
            |--> | gpiod_line_iter_free |                                   
            |    +----------------------+                                   
            |                                                               
            +--> return                                                     
```

```
lib/iter.c                                                                  
+---------------------+                                                      
| gpiod_line_iter_new | : prepare line_iter (have each line info ready)      
+-|-------------------+                                                      
  |                                                                          
  |--> alloc line_iter                                                       
  |                                                                          
  |--> get line# from chip, alloc that many lines and save in line_iter      
  |                                                                          
  +--> for each line                                                         
       |                                                                     
       |    +---------------------+                                          
       +--> | gpiod_chip_get_line | ensure target line exists in chip, get it
            +---------------------+                                          
```

```
lib/core.c                                                        
+---------------------+                                            
| gpiod_chip_get_line | : ensure target line exists in chip, get it
+-|-------------------+                                            
  |                                                                
  |--> ensure chip->lines is allocated                             
  |                                                                
  |--> if chip->line[offset] doesn't exist yet                     
  |    -                                                           
  |    +--> alloc line and set up, save in chip                    
  |                                                                
  |    else                                                        
  |    -                                                           
  |    +--> get line from chip                                     
  |                                                                
  |    +-------------------+                                       
  +--> | gpiod_line_update | get line info and fill arg line       
       +-------------------+                                       
```

```
lib/core.c                                            
+-------------------+                                  
| gpiod_line_update | : get line info and fill arg line
+-|-----------------+                                  
  |                                                    
  |--> save offset in line struct                      
  |                                                    
  |    +-------+                                       
  |--> | ioctl | opt = get_line_info                   
  |    +-------+                                       
  |                                                    
  +--> set up line based on the info                   
```

```
bindings/cxx/line.cpp                                    
+---------------+                                         
| line::request | : given config, set target lines
+-|-------------+                                         
  |                                                       
  |--> prepare bulk                                       
  |                                                       
  |    +--------------------+                             
  +--> | line_bulk::request | given config, get line values or events
       +--------------------+                             
```

```
 bindings/cxx/line_bulk.cpp                                                 
+--------------------+                                                      
| line_bulk::request | : given config, get line values or events            
+-|------------------+                                                      
  |    +-------------------------+                                          
  |--> | line_bulk::to_line_bulk | add all lines to bulk struct             
  |    +-------------------------+                                          
  |                                                                         
  |--> set up config (consumer, type: direction or event, flags)            
  |                                                                         
  |    +-------------------------+                                          
  +--> | gpiod_line_request_bulk | given req type, get line values or events
       +-------------------------+                                          
```

```
bindings/cxx/line_bulk.cpp                               
+-------------------------+                               
| line_bulk::to_line_bulk | : add all lines to bulk struct
+-|-----------------------+                               
  |    +----------------------+                           
  |--> | gpiod_line_bulk_init | set line_num = 0          
  |    +----------------------+                           
  |                                                       
  +--> for each line: add to bulk struct                  
```

```
lib/core.c                                                            
+-------------------------+                                            
| gpiod_line_request_bulk | : given req type, get line values or events
+-|-----------------------+                                            
  |                                                                    
  |--> if request type is about direction                              
  |    |                                                               
  |    |    +---------------------+                                    
  |    +--> | line_request_values | get each line info                 
  |         +---------------------+                                    
  |                                                                    
  +--> elif request type is about event                                
       |                                                               
       |    +---------------------+                                    
       +--> | line_request_events | get each line info                 
            +---------------------+                                    
```

```
lib/core.c                                                           
+---------------------+                                               
| line_request_values | : get each line info                          
+-|-------------------+                                               
  |                                                                   
  |--> set up req_handle                                              
  |                                                                   
  |--> set direction in flags                                         
  |                                                                   
  |--> for each line in bulk                                          
  |    -                                                              
  |    +--> set up line_offset and default_value of line in req_handle
  |                                                                   
  |--> retrieve fd from bulk                                          
  |                                                                   
  |    +-------+                                                      
  |--> | ioctl | opt = get_line_handle (kernel saves target fd in req)
  |    +-------+                                                      
  |    +---------------------+                                        
  +--> | line_make_fd_handle | prepare line_fd                        
  |    +---------------------+                                        
  |                                                                   
  +--> for each line in bulk                                          
       |                                                              
       |--> set up line                                               
       |                                                              
       |    +-------------+                                           
       |--> | line_set_fd | save line_fd in line                      
       |    +-------------+                                           
       |    +-------------------+                                     
       +--> | gpiod_line_update | get line info and fill arg line     
            +-------------------+                                     
```

```
lib/core.c                                                   
+---------------------+                                       
| line_request_events | : get each line info                  
+-|-------------------+                                       
  |                                                           
  +--> for each line                                          
       |                                                      
       |    +---------------------------+                     
       +--> | line_request_event_single | get single line info
            +---------------------------+                     
```

```
lib/core.c                                                 
+---------------------------+                               
| line_request_event_single | : get single line info        
+-|-------------------------+                               
  |                                                         
  |--> set up event_req                                     
  |                                                         
  |--> set up flags (rising, falling, both)                 
  |                                                         
  |    +-------+                                            
  |--> | ioctl | opt = get_line_event                       
  |    +-------+                                            
  |    +---------------------+                              
  |--> | line_make_fd_handle | prepare line_fd              
  |    +---------------------+                              
  |                                                         
  |--> set up line                                          
  |                                                         
  |    +-------------------+                                
  +--> | gpiod_line_update | get line info and fill arg line
       +-------------------+                                
```

```
bindings/cxx/line.cpp                               
+--------------------+                               
| line::event_get_fd | : given line, get fd          
+-|------------------+                               
  |    +-------------------------+                   
  +--> | gpiod_line_event_get_fd | given line, get fd
       +-------------------------+                   
```

```
lib/core.c                                     
+-------------------------+                     
| gpiod_line_event_get_fd | : given line, get fd
+-|-----------------------+                     
  |    +-------------+                          
  +--> | line_get_fd | given line, get fd       
       +-------------+                          
```

```
bindings/cxx/line.cpp                                                          
+------------------+                                                            
| line::event_read | : read events from fd                                      
+-|----------------+                                                            
  |    +-----------------------+                                                
  |--> | gpiod_line_event_read | read events from fd, so as to set up arg events
  |    +-----------------------+                                                
  |    +-----------------------+                                                
  +--> | line::make_line_event | make line event                                
       +-----------------------+                                                
```

```
lib/core.c                                                                              
+-----------------------+                                                                
| gpiod_line_event_read | : read events from fd, so as to set up arg events              
+-|---------------------+                                                                
  |    +--------------------------------+                                                
  +--> | gpiod_line_event_read_multiple | read events from fd, so as to set up arg events
       +--------------------------------+                                                
```

```
lib/core.c                                                                                 
+--------------------------------+                                                          
| gpiod_line_event_read_multiple | : read events from fd, so as to set up arg events        
+-|------------------------------+                                                          
  |    +-------------------------+                                                          
  |--> | gpiod_line_event_get_fd | get fd                                                   
  |    +-------------------------+                                                          
  |    +-----------------------------------+                                                
  +--> | gpiod_line_event_read_fd_multiple | read events from fd, so as to set up arg events
       +-----------------------------------+                                                
```

```
lib/core.c                                                                            
+-----------------------------------+                                                  
| gpiod_line_event_read_fd_multiple | : read events from fd, so as to set up arg events
+-|---------------------------------+                                                  
  |    +------+                                                                        
  |--> | read | read events                                                            
  |    +------+                                                                        
  |                                                                                    
  +--> for each event                                                                  
       -                                                                               
       +--> set up arg event based on read event                                       
```

```
bindings/cxx/line.cpp                                
+-----------------------+                             
| line::make_line_event | : make line event           
+-|---------------------+                             
  |                                                   
  |--> set event type to rising or falling accordingly
  |                                                   
  +--> save timestamp and source in arg               
```

## <a name="pin-control"></a> Pin Control

Some GPIO pins are multi-functional; they can have one or two extra functionality through proper settings. 
For kernel to ensure these functions work as expected, it introduces the pin control (pinctrl) mechanism to set the required registers beforehand automatically. 
Let say I2C channels one and two are born to be I2C channels, and pinctrl is unnecessary before they work. 
However, I2C channel three is different, and proper settings are essential.

```
i2c-bus@c0 {
    #address-cells = <0x01>;
    #size-cells = <0x00>;
    #interrupt-cells = <0x01>;
    reg = <0xc0 0x40>;
    compatible = "aspeed,ast2500-i2c-bus";
    clocks = <0x02 0x1a>;
    resets = <0x02 0x07>;
    bus-frequency = <0x186a0>;
    interrupts = <0x02>;
    interrupt-parent = <0x1c>;
    pinctrl-names = "default";
    pinctrl-0 = <0x1d>; <========== refer to phandle 0x1d
    status = "okay";
};
```

And descriptor with phandle 0x1d is:

```
i2c3_default {      
    function = "I2C3";
    groups = "I2C3";
    phandle = <0x1d>;
};
```

From file **drivers/pinctrl/aspeed/pinctrl-aspeed-g5.c** we have the below definition:

```                                                                                                
 #define I2C3_DESC   SIG_DESC_SET(SCU90, 16) <========== setting to enable multi-function of the below pins
                                                                                                           
                                                                                                           
                          ball       setting                                                               
                            |           |                                                                  
 #define A11 128            v           v                                                                  
 SIG_EXPR_LIST_DECL_SINGLE(A11, SCL3, I2C3, I2C3_DESC);                                                    
 PIN_DECL_1(A11, GPIOQ0, SCL3);   ^              ^                                                         
                    ^      ^      |              |                                                         
                    |      |   signal         setting                                                      
                  gpio     |                                                                               
               (low prio)  |                                                                               
                           |                                                                               
                         func1                                                                             
                      (high prio)                                                                          
                                                                                                           
 #define A10 129                                                                                           
 SIG_EXPR_LIST_DECL_SINGLE(A10, SDA3, I2C3, I2C3_DESC);                                                    
 PIN_DECL_1(A10, GPIOQ1, SDA3);                                                                            
                                                                                                           
 FUNC_GROUP_DECL(I2C3, A11, A10) <========== declare A11, A10 as group I2C3                                
```

In the above declaration, ball **A11** can only be function 1 (SCL3, high priority) or other (GPIO, low priority). 
In some cases, it equips two extra functions instead of one. 
The I2C channel nine is an excellent example of this illustration.

```
 #define I2C9_DESC   SIG_DESC_SET(SCU90, 22) <========== setting to enable multi-function of the below pins                 
                                                                                                                            
                               signal                                                                                       
                                  |             settings to enable this function                                            
                          ball    |  function   +--------+                                                                  
                            |     |     |       |        |                                                                  
 #define C14 4              v     v     v       v        v                                                                  
 SIG_EXPR_LIST_DECL_SINGLE(C14, SCL9, I2C9, I2C9_DESC, COND1);                                                              
 SIG_EXPR_LIST_DECL_SINGLE(C14, TIMER5, TIMER5, SIG_DESC_SET(SCU80, 4), COND1);                                             
                            ^     ^     ^                  ^              ^                                                 
                            |     |     |                  |              |                                                 
                          ball    |  function              +--------------+                                                 
                                  |                        settings to enable this function                                 
                               signal                                                                                       
                                                                                                                            
                         func1                                                                                              
                      (high prio)                                                                                           
                           |                                                                                                
                           v                                                                                                
 PIN_DECL_2(C14, GPIOA4, SCL9, TIMER5);                                                                                     
                    ^             ^                                                                                         
                    |             |                                                                                         
                  other         func2                                                                                       
              (lowest prio)   (low prio)                                                                                    
                                                                                                                            
 FUNC_GROUP_DECL(TIMER5, C14);    <========== declare that group 'TIMER5' contains ball 'C14'                               
                                                                                                                            
 #define A13 5                                                                                                              
 SIG_EXPR_LIST_DECL_SINGLE(A13, SDA9, I2C9, I2C9_DESC, COND1);                                                              
 SIG_EXPR_LIST_DECL_SINGLE(A13, TIMER6, TIMER6, SIG_DESC_SET(SCU80, 5), COND1);                                             
 PIN_DECL_2(A13, GPIOA5, SDA9, TIMER6);                                                                                     
                                                                                                                            
 FUNC_GROUP_DECL(TIMER6, A13);    <========== declare that group 'TIMER6' contains ball 'A13'                               
                                                                                                                            
 FUNC_GROUP_DECL(I2C9, C14, A13); <========== declare that group 'I2C9' contains ball 'C14' and 'A13'                       
```

It's ok not to expand the above macro into scary expressions and complicated definitions. 
Knowing these relations between pins and functions is enough, and let's turn to the pinctrl initialization during kernel boot up. 
Pinctrl also has its descriptor in the device tree and triggers the probing after matching the driver.

```
Driver:

static const struct of_device_id aspeed_g5_pinctrl_of_match[] = {
    { .compatible = "aspeed,ast2500-pinctrl", }, <========== match
    { .compatible = "aspeed,g5-pinctrl", },
    { },
};
```

```
DTS:

pinctrl@80 {
    compatible = "aspeed,ast2500-pinctrl"; <========== match
    reg = <0x80 0x18 0xa0 0x10>;
    aspeed,external-nodes = <0x08 0x09>;
    phandle = <0x0d>;
```

It's better to register pinctrl device first, otherwise devices that need pinctrl will have to defer to a later probe.

```
+-------------------------+                                                                  
| aspeed_g5_pinctrl_probe |                                                                  
+------|------------------+                                                                  
       |                                                                                     
       |--> reassign the number of each pin in aspeed_g5_pins                                
       |                                                                                     
       |    +----------------------+                                                         
       +--> | aspeed_pinctrl_probe |                                                         
            +-----|----------------+                                                         
                  |    +-----------------------+                                             
                  |--> | syscon_node_to_regmap | get scu register base address               
                  |    +-----------------------+                                             
                  |    +------------------+                                                  
                  +--> | pinctrl_register | set up pin device and register all the pins to it
                       +------------------+                                                  
```

<details>
  <summary> Other code tracing </summary>

```
+------------------+                                                    
| pinctrl_register |                                                    
+----|-------------+                                                    
     |    +-------------------------+                                   
     +--> | pinctrl_init_controller |                                   
          +------|------------------+                                   
                 |                                                      
                 |--> allocate and set up pinctrl dev                   
                 |                                                      
                 |    +-----------------------+                         
                 +--> | pinctrl_register_pins |                         
                      +-----|-----------------+                         
                            |                                           
                            +--> for each pin                           
                                                                        
                                     +--------------------------+       
                                     | pinctrl_register_one_pin |       
                                     +------|-------------------+       
                                            |                           
                                            |--> allocate pin descriptor
                                            |                           
                                            +--> add to pinctrl dev     
```
  
```
              ball          function1                         
                |               |                             
                |               |                             
 #define B14 0  v               v                             
 SSSF_PIN_DECL(B14, GPIOA0, MAC1LINK, SIG_DESC_SET(SCU80, 0));
                       ^                         ^            
                       |                         |            
                       |                         |            
                     GPIO                     setting         
                                                for           
                                             function1                             
```

```
                                                          setting          
                                                            for            
                               function1                 function1         
                                   |                         |             
                                   |                         |             
                                   v                         v             
 SIG_EXPR_LIST_DECL_SINGLE(D13, SPI1CS1, SPI1CS1, SIG_DESC_SET(SCU80, 15));
 SIG_EXPR_LIST_DECL_SINGLE(D13, TIMER3, TIMER3, SIG_DESC_SET(SCU80, 2));   
                                   ^                         ^             
                                   |                         |             
                                   |                         |             
                               function2                  setting          
                                                            for            
                       (high prio)                       function2         
           ball         function1                                          
             |              |                                              
             |              |                                              
             v              v                                              
 PIN_DECL_2(D13, GPIOA2, SPI1CS1, TIMER3);                                 
                    ^                ^                                     
                    |                |                                     
                    |                |                                     
                  GPIO           function2                                 
              (lowest prio)      (low prio)                                
```

```
// MAC1LINK
static const struct aspeed_sig_desc sig_descs_MAC1LINK_MAC1LINK[] = {
    { ASPEED_IP_SCU, SCU80, BIT_MASK(0), 1, 0 }
};
static const struct aspeed_sig_expr sig_exprs_MAC1LINK_MAC1LINK = {
    .signal = "MAC1LINK",
    .function = "MAC1LINK",
    .ndescs = ARRAY_SIZE(sig_descs_MAC1LINK_MAC1LINK),
    .descs = &sig_descs_MAC1LINK_MAC1LINK[0],
};
static const struct aspeed_sig_expr *sig_exprs_MAC1LINK_MAC1LINK[] = {
    &sig_exprs_MAC1LINK_MAC1LINK, NULL
};
static const struct aspeed_sig_expr *
sig_exprs_0_MAC1LINK[ARRAY_SIZE(sig_exprs_MAC1LINK_MAC1LINK)]
__attribute__((alias(istringify(sig_exprs_MAC1LINK_MAC1LINK))));

// GPIOA0
static const struct aspeed_sig_desc sig_descs_GPIOA0_GPIOA0[] = {
};
static const struct aspeed_sig_expr sig_exprs_GPIOA0_GPIOA0 = {
    .signal = "GPIOA0",
    .function = "GPIOA0",
    .ndescs = ARRAY_SIZE(sig_descs_GPIOA0_GPIOA0),
    .descs = &sig_descs_GPIOA0_GPIOA0[0],
};
static const struct aspeed_sig_expr *sig_exprs_GPIOA0_GPIOA0[] = {
    &sig_exprs_GPIOA0_GPIOA0, NULL
};
static const struct aspeed_sig_expr *
sig_exprs_0_GPIOA0[ARRAY_SIZE(sig_exprs_GPIOA0_GPIOA0)]
__attribute__((alias(istringify(sig_exprs_GPIOA0_GPIOA0))));

// Both
static const struct aspeed_sig_expr **pin_exprs_0[] = {
    sig_exprs_0_MAC1LINK, sig_exprs_0_GPIOA0, NULL
};
static const struct aspeed_pin_desc pin_0 = {
    "0", &pin_exprs_0[0]
};
static const int group_pins_MAC1LINK[] = { 0 };
static const char *func_groups_MAC1LINK[] = { "MAC1LINK" };
```
  
</details>

When drivers and devices are busy matching each other, the pinctrl mechanism goes before the probing function to ensure the device is ready. 
As we mentioned earlier that I2C channel three needs pin control on **I2C3** for its default state. 
The below **create_pinctrl** looks into the device tree for the pinctrl properties, and later **pinctrl_select_state** operates on corresponding settings.

```
+-------------------+                                                                  
| pinctrl_bind_pins |                                                                  
+----|--------------+                                                                  
     |                                                                                 
     |--> allocate dev_pin_info for device                                             
     |                                                                                 
     |    +------------------+                                                         
     |--> | devm_pinctrl_get |                                                         
     |    +----|-------------+                                                         
     |         |                                                                       
     |         |--> allocate pinctrl                                                   
     |         |                                                                       
     |         |    +-------------+                                                    
     |         |--> | pinctrl_get |                                                    
     |         |    +---|---------+                                                    
     |         |        |    +--------------+                                          
     |         |        |--> | find_pinctrl | get pinctrl if it exists already         
     |         |        |    +--------------+                                          
     |         |        |    +----------------+                                        
     |         |        +--> | create_pinctrl | set up pinctrl, add map to pinctrl_maps
     |         |             +----------------+                                        
     |         |    +------------+                                                     
     |         +--> | devres_add | add pinctrl to device as resource                   
     |              +------------+                                                     
     |                                                                                 
     |--> get 'default' state                                                          
     |                                                                                 
     |--> get 'init' state (it's ok not to specify this in dts)                        
     |                                                                                 
     |    +----------------------+                                                     
     +--> | pinctrl_select_state | ensure pinctrl is in the specified state            
          +----------------------+                                                     
```

```
                +----------------+                  +----------------------+
                |can be 'default'|                  |nri1_default {        |
                |       'init'   |                  |    function = "NRI1";|
                |       'sleep'  |                  |    groups = "NRI1";  |
                |       'idle'   |                  |    phandle = <0x17>; |
                +----------------+                  |};                    |
                                                    +----------------------+
                       state                                                
 serial@1e783000 {       |                          config                  
     ...                 v                             |                    
     pinctrl-names = "default";                        v                    
     pinctrl-0 = <0x10 0x11 0x12 0x13 0x14 0x15 0x16 0x17>;                 
 };                                               ^                         
                                                  |                         
                                               config                       
                                                                            
                                      +-----------------------+             
                                      |ndcd1_default {        |             
                                      |    function = "NDCD1";|             
                                      |    groups = "NDCD1";  |             
                                      |    phandle = <0x16>;  |             
                                      |};                     |             
                                      +-----------------------+             
```

<details>
  <summary> Other code tracing </summary>

```
+----------------+                                                                                            
| create_pinctrl |                                                                                            
+---|------------+                                                                                            
    |                                                                                                         
    |--> allocate pinctrl                                                                                     
    |                                                                                                         
    |    +-------------------+                                                                                
    |--> | pinctrl_dt_to_map |                                                                                
    |    +----|--------------+                                                                                
    |         |                                                                                               
    |         +--> for each state ('default', 'init', 'sleep', 'idle')                                        
    |                                                                                                         
    |                  get property from DT                                                                   
    |                                                                                                         
    |                  return if not found                                                                    
    |                                                                                                         
    |                  for each config                                                                        
    |                                                                                                         
    |                      +-------------------------+                                                        
    |                      | of_find_node_by_phandle | find target node by config (handle)                    
    |                      +-------------------------+                                                        
    |                                                                                                         
    |                      +----------------------+                                                           
    |                      | dt_to_map_one_config | save info to map, add to pinctrl and pinctrl_maps (global)
    |                      +----------------------+                                                           
    |                                                                                                         
    |--> return if the device needs no pinctrl (most cases)                                                   
    |                                                                                                         
    |--> for each dev-name-matched map in pinctrl_maps                                                        
    |                                                                                                         
    |        +-------------+                                                                                  
    |        | add_setting | ensure 'state' exists and set up 'setting' based on 'map', and add to 'state'    
    |        +-------------+                                                                                  
    |                                                                                                         
    +--> add pinctrl to pinctrl_list     
```
  
```
+----------------------+                                                                                    
| dt_to_map_one_config | save info to map, add to pinctrl and pinctrl_maps (global)                         
+-----|----------------+                                                                                    
      |                                                                                                     
      |--> get pinctrl dev                                                                                  
      |                                                                                                     
      |--> call ->dt_node_to_map()                                                                          
      |          +------------------------------------+                                                     
      |    e.g., | pinconf_generic_dt_node_to_map_all | save mux/configs info of one DT node to map         
      |          +------------------------------------+                                                     
      |                                                                                                     
      |    +-------------------------+                                                                      
      +--> | dt_remember_or_free_map |                                                                      
           +------|------------------+                                                                      
                  |                                                                                         
                  |--> set up dt_map and add to pinctrl                                                     
                  |                                                                                         
                  |    +----------------------------+                                                       
                  +--> |  pinctrl_register_mappings | set up maps_node and register to pinctrl_maps (global)
                       +----------------------------+                                                       
```
  
```
+------------------------------------+                                                                     
| pinconf_generic_dt_node_to_map_all | save mux/configs info of one DT node to map                         
+--------|---------------------------+                                                                     
         |    +--------------------------------+                                                           
         +--> | pinconf_generic_dt_node_to_map |                                                           
              +-------|------------------------+                                                           
                      |    +-----------------------------------+                                           
                      |--> | pinconf_generic_dt_subnode_to_map |                                           
                      |    +--------|--------------------------+                                           
                      |             |                                                                      
                      |             |--> get property 'function'                                           
                      |             |                                                                      
                      |             |    +---------------------------------+                               
                      |             |--> | pinconf_generic_parse_dt_config |                               
                      |             |    +---------------------------------+                               
                      |             |                                                                      
                      |             +--> for each string in property (one string in our case)              
                      |                                                                                    
                      |                      if property 'function' exists                                 
                      |                                                                                    
                      |                          +---------------------------+                             
                      |                          | pinctrl_utils_add_map_mux | save mux info in map        
                      |                          +---------------------------+                             
                      |                                                                                    
                      |                      if pinconf node has config(s)                                 
                      |                                                                                    
                      |                          +-------------------------------+                         
                      |                          | pinctrl_utils_add_map_configs | save configs info in map
                      |                          +-------------------------------+                         
                      |                                                                                    
                      +-->  apply the same logic to the child nodes (no child in our case)                 
```

```                                                                                                                    
+-------------+                                                                                                               
| add_setting | ensure 'state' exists and set up 'setting' based on 'map', and add to 'state'                                 
+---|---------+                                                                                                               
    |    +------------+                                                                                                       
    |--> | find_state | find the state in pinctrl based on the map name                                                       
    |    +------------+                                                                                                       
    |                                                                                                                         
    |--> if not found                                                                                                         
    |                                                                                                                         
    |         +--------------+                                                                                                
    |         | create_state | create 'state' and add to pinctrl                                                              
    |         +--------------+                                                                                                
    |                                                                                                                         
    |--> allocate and set up 'setting'                                                                                        
    |                                                                                                                         
    |--> if map type is MUX_GROUP                                                                                             
    |                                                                                                                         
    |        +-----------------------+                                                                                        
    |        | pinmux_map_to_setting | save indexes of function and group name to 'setting'                                   
    |        +-----------------------+                                                                                        
    |                                                                                                                         
    |--> else if map type is CONFIGS_PIN or CONFIGS_GROUP                                                                     
    |                                                                                                                         
    |        +------------------------+                                                                                       
    |        | pinconf_map_to_setting | save pin number of matched name to 'setting', copy config info from 'map' to 'setting'
    |        +------------------------+                                                                                       
    |                                                                                                                         
    +--> add 'setting' to 'state'                                                                                             
```
  
```
+------------------------+                                                        
| pinconf_map_to_setting |                                                        
+-----|------------------+                                                        
      |                                                                           
      |--> if type is PIN                                                         
      |                                                                           
      |        +-------------------+                                              
      |        | pin_get_from_name |                                              
      |        +-------------------+                                              
      |                                                                           
      |        save pin number in 'setting'                                       
      |                                                                           
      |--> else if type is GROUP                                                  
      |                                                                           
      |        +----------------------------+                                     
      |        | pinctrl_get_group_selector | get pin number of matched group name
      |        +----------------------------+                                     
      |                                                                           
      |        save pin number in 'setting'                                       
      |                                                                           
      +--> copy config info from 'map' to 'setting'                               
```
  
```
+----------------------+                                                                            
| pinctrl_select_state |                                                                            
+-----|----------------+                                                                            
      |                                                                                             
      |--> return if it's already the specified state                                               
      |                                                                                             
      |    +----------------------+                                                                 
      +--> | pinctrl_commit_state |                                                                 
           +-----|----------------+                                                                 
                 |                                                                                  
                 |--> if pinctrl has old state                                                      
                 |                                                                                  
                 |        +------------------------+                                                
                 |        | pinmux_disable_setting |                                                
                 |        +------------------------+                                                
                 |                                                                                  
                 |--> for each setting on new state                                                 
                 |                                                                                  
                 |        +-----------------------+                                                 
                 |        | pinmux_enable_setting | request pin(s) and set mux to enable the setting
                 |        +-----------------------+                                                 
                 |                                                                                  
                 |--> for each setting on new state                                                 
                 |                                                                                  
                 |        +-----------------------+                                                 
                 |        | pinconf_apply_setting | configure pin(s)                                
                 |        +-----------------------+                                                 
                 |                                                                                  
                 +--> update pinctrl state                                                          
```
  
```
+-----------------------+                                                                                                             
| pinmux_enable_setting |                                                                                                             
+-----|-----------------+                                                                                                             
      |                                                                                                                               
      |--> get pinctrl ops from pinctrl dev                                                                                           
      |                                                                                                                               
      |--> get pinmux ops from pinctrl dev                                                                                            
      |                                                                                                                               
      |--> if pinctrl_ops->get_group_pins is prepared                                                                                 
      |                                                                                                                               
      |        all pinctrl_ops->get_group_pins()                                                                                      
      |              +-------------------------------+                                                                                
      |        e.g., | aspeed_pinctrl_get_group_pins | get a group of pins                                                            
      |              +-------------------------------+                                                                                
      |                                                                                                                               
      |--> for each pin in group                                                                                                      
      |                                                                                                                               
      |        +-------------+                                                                                                        
      |        | pin_request |                                                                                                        
      |        +---|---------+                                                                                                        
      |            |                                                                                                                  
      |            +--> get pin desc from pinctrl dev                                                                                 
      |                                                                                                                               
      |                 update owner of the pin desc                                                                                  
      |                                                                                                                               
      |                 call pinmux_ops->gpio_request_enable()                                                                        
      |                       +----------------------------+                                                                          
      |                 e.g., | aspeed_gpio_request_enable | disable all other high priority functions, so it's now a regular gpio pin
      |                       +----------------------------+                                                                          
      |                                                                                                                               
      |    call pinmux_ops->set_mux()                                                                                                 
      +-->       +-----------------------+                                                                                            
           e.g., | aspeed_pinmux_set_mux | disable higher priority functions, and enable mux                                          
                 +-----------------------+                                                                                            
```

```
+-----------------------+                                                                              
| pinconf_apply_setting |                                                                              
+-----|-----------------+                                                                              
      |                                                                                                
      |--> if setting type is PIN                                                                      
      |                                                                                                
      |        call ops->pin_config_set()                                                              
      |              +-----------------------+                                                         
      |        e.g., | aspeed_pin_config_set | for each config, update register based on config and map
      |              +-----------------------+                                                         
      |                                                                                                
      +--> if setting type is GROUP                                                                    
                                                                                                       
               call ops->pin_config_group_set()                                                        
                     +-----------------------------+                                                   
               e.g., | aspeed_pin_config_group_set | get a group of pins and config them               
                     +-----------------------------+                                                   
```
  
</details>

## <a name="system-startup"></a> System Startup

```
gpiolib_dev_init: register bus & driver, reserve dev#
gpiolib_sysfs_init: register 'gpio' class, for each entry@gpio_devices: create device and register to sysfs
gpiolib_debugfs_init: prepare '/sys/kernel/debug/gpio' with fops=gpiolib_fops
aspeed_gpio_probe: set up aspeed_gpio, install ops, prepare gpio line descriptors, register gpio chip
aspeed_sgpio_driver_init: (skip, no matched device)
gpio_clk_driver_init: (skip, no matched device)
gpio_keys_polled_driver_init: (skip, no matched device)
i2c_mux_gpio_driver_init: (skip, no matched device)
w1_gpio_driver_init: (skip, no matched device)
gpio_led_driver_init: register 'gpio_led_driver' to bus 'platform'
  gpio_led_probe: for each led, prepare gpio_desc and led_struct                         
fsi_master_gpio_driver_init: (skip, no matched device)
gpio_keys_init: register 'gpio_keys_device_driver' to bus 'platform'
  gpio_keys_probe: prepare pdate & input_dev & ddata, for each button: set up key, register input_dev
```

```
+------------------+                                                                                 
| gpiolib_dev_init | : register bus & driver, reserve dev#
+----|-------------+                                                                                 
     |    +--------------+                                                                           
     |--> | bus_register | register 'gpio_bus_type'                                                  
     |    +--------------+                                                                           
     |    +-----------------+                                                                        
     |--> | driver_register | register 'gpio_stub_drv' (not important in our case)                   
     |    +-----------------+                                                                        
     |    +---------------------+                                                                    
     |--> | alloc_chrdev_region | reserve a range (256) of minor numbers for gpio character devices  
     |    +---------------------+                                                                    
     |                                                                                               
     |--> gpiolib_initialized = true                                                                 
     |                                                                                               
     |    +---------------------+                                                                    
     +--> | gpiochip_setup_devs | register each gpiochip dev in 'gpio_devices' (empty in in our case)
          +---------------------+                                                                    
```

```                                                                                        
drivers/gpio/gpiolib-sysfs.c                                                                                   
+--------------------+                                                                                          
| gpiolib_sysfs_init | : register 'gpio' class, for each entry@gpio_devices: create device and register to sysfs
+-|------------------+                                                                                          
  |    +----------------+                                                                                       
  |--> | class_register | register 'gpio' class                                                                 
  |    +----------------+                                                                                       
  |                                                                                                             
  +--> for each entry on 'gpio_devices'                                                                         
       |                                                                                                        
       |    +-------------------------+                                                                         
       +--> | gpiochip_sysfs_register | create device and register to sysfs                                     
            +-------------------------+                                                                         
```

```
drivers/gpio/gpiolib.c                                                           
+----------------------+                                                          
| gpiolib_debugfs_init | : prepare '/sys/kernel/debug/gpio' with fops=gpiolib_fops
+----------------------+                                                          
```


```
drivers/gpio/gpio-aspeed.c
+-------------------+                                                                                                    
| aspeed_gpio_probe | : set up aspeed_gpio, install ops, prepare gpio line descriptors, register gpio chip
+----|--------------+                                                                                                    
     |                                                                                                                   
     |--> set up 'aspeed_gpio' and install gpio chip operations (in, out, get, set, ...)                                 
     |                                                                                                                   
     |--> set up 'irq_chip' of 'aspeed_gpio' if device tree specifies the irq#                                           
     |                                                                                                                   
     |    +------------------------+                                                                                     
     +--> | devm_gpiochip_add_data | prepare desc for each gpio line, set up irq desc, register gpio chip 
          +------------------------+ (connect the generic layer (gpio lib) to specific driver (aspeed))
```

```
include/linux/gpio/driver.h
+------------------------+
| devm_gpiochip_add_data | : prepare desc for each gpio line, set up irq desc, register gpio chip
+-----|------------------+
      |    +---------------------------------+
      +--> | devm_gpiochip_add_data_with_key | : prepare desc for each gpio line, set up irq desc, register gpio chip
           +--------|------------------------+
                    |    +----------------------------+
                    +--> | gpiochip_add_data_with_key | prepare desc for each gpio line, set up irq desc, register gpio chip
                    |    +----------------------------+
                    |    +--------------------------+
                    +--> | devm_add_action_or_reset | add action 'devm_gpio_chip_release' as device resource
                         +--------------------------+
```

```
drivers/gpio/gpiolib.c
+----------------------------+
| gpiochip_add_data_with_key | : prepare desc for each gpio line, set up irq desc, register gpio chip
+------|---------------------+
       |
       |--> set up 'gpio device'
       |
       |--> allocate 'descriptor' for each GPIO line
       |
       |    +---------------------+
       |--> | gpiodev_add_to_list | add 'gpio device' to the 'gpio_devices' list
       |    +---------------------+
       |    +-----------------+
       |--> | of_gpiochip_add | add pin range and set hog (default gpio behavior)
       |    +-----------------+
       |    +----------------------+
       |--> | gpiochip_add_irqchip | set up target 'irq desc' based on the 'gpio chip'
       |    +----------------------+
       |
       +--> if gpiolib_initialized is set (yes, in gpio lib init)
            |
            |    +--------------------+
            +--> | gpiochip_setup_dev | register cdev
                 +--------------------+
```

```
drivers/leds/leds-gpio.c                                                                   
+----------------+                                                                          
| gpio_led_probe | : for each led, prepare gpio_desc and led_struct                         
+-|--------------+                                                                          
  |                                                                                         
  |--> if led# > 0                                                                          
  |    |                                                                                    
  |    |    +--------------+                                                                
  |    |--> | devm_kzalloc | alloc private                                                  
  |    |    +--------------+                                                                
  |    |                                                                                    
  |    +--> for each led                                                                    
  |         |                                                                               
  |         |    +--------------------+                                                     
  |         |--> | gpio_led_get_gpiod | get gpio desc, add to device as resource            
  |         |    +--------------------+                                                     
  |         |    +-----------------+                                                        
  |         +--> | create_gpio_led | install ops, set gpio direction out, prepare led struct
  |              +-----------------+                                                        
  |                                                                                         
  +--> else                                                                                 
       |                                                                                    
       |    +------------------+                                                            
       +--> | gpio_leds_create | for each child, prepare gpio_desc and led_struct           
            +------------------+                                                            
```

```
drivers/leds/leds-gpio.c                                               
+--------------------+                                                  
| gpio_led_get_gpiod | : get gpio desc, add to device as resource       
+-|------------------+                                                  
  |    +----------------------+                                         
  +--> | devm_gpiod_get_index | get gpio desc, add to device as resource
       +----------------------+                                         
```

```
drivers/gpio/gpiolib-devres.c                                                 
+----------------------+                                                       
| devm_gpiod_get_index | : get gpio desc, add to device as resource            
+-|--------------------+                                                       
  |    +-----------------+                                                     
  |--> | gpiod_get_index | get gpio desc, request and set direction accordingly
  |    +-----------------+                                                     
  |    +--------------+                                                        
  |--> | devres_alloc | alloc a ptr                                            
  |    +--------------+                                                        
  |                                                                            
  |--> have the ptr points to gpio desc                                        
  |                                                                            
  |    +------------+                                                          
  +--> | devres_add |                                                          
       +------------+                                                          
```

```
drivers/gpio/gpiolib.c                                                               
+-----------------+                                                                   
| gpiod_get_index | : get gpio desc, request and set direction accordingly            
+-|---------------+                                                                   
  |                                                                                   
  |--> if of node exists (our case)                                                   
  |    |                                                                              
  |    |    +--------------+                                                          
  |    +--> | of_find_gpio |                                                          
  |         +--------------+                                                          
  |                                                                                   
  |--> else                                                                           
  |    |                                                                              
  |    |    +----------------+                                                        
  |    +--> | acpi_find_gpio |                                                        
  |         +----------------+                                                        
  |                                                                                   
  |--> if the desc is still not found                                                 
  |    |                                                                              
  |    |    +------------+                                                            
  |    +--> | gpiod_find |                                                            
  |         +------------+                                                            
  |                                                                                   
  |--> return if no gpio consumer (desc) found                                        
  |                                                                                   
  |    +---------------+                                                              
  |--> | gpiod_request | set label, call ->request() and ->get_direction() if feasible
  |    +---------------+                                                              
  |    +-----------------------+                                                      
  |--> | gpiod_configure_flags | set gpio direction accordingly                       
  |    +-----------------------+                                                      
  |                                                                                   
  +--> call notifier (gpio_line_changed_requested)                                    
```

```
drivers/gpio/gpiolib.c                                                                      
+---------------+                                                                            
| gpiod_request | : set label, call ->request() and ->get_direction() if feasible            
+-|-------------+                                                                            
  |                                                                                          
  |--> get gpio_dev from desc                                                                
  |                                                                                          
  |    +----------------------+                                                              
  +--> | gpiod_request_commit | set label, call ->request() and ->get_direction() if feasible
       +----------------------+                                                              
```

```
drivers/gpio/gpiolib.c                                                                 
+----------------------+                                                                
| gpiod_request_commit | : set label, call ->request() and ->get_direction() if feasible
+-|--------------------+                                                                
  |    +----------------+                                                               
  |--> | desc_set_label |                                                               
  |    +----------------+                                                               
  |                                                                                     
  |--> if gpio_chip->request() exists                                                   
  |    |                                                                                
  |    |    +------------------+                                                        
  |    |--> | gpio_chip_hwgpio | get desc offset                                        
  |    |    +------------------+                                                        
  |    |                                                                                
  |    +--> if the offset is valid                                                      
  |         -                                                                           
  |         +--> call ->request()                                                       
  |                                                                                     
  +--> if gpio_chip->request() exists                                                   
       |                                                                                
       |    +---------------------+                                                     
       +--> | gpiod_get_direction | call ->get_direction() and save result in desc flags
            +---------------------+                                                     
```

```
drivers/gpio/gpiolib.c                                                       
+---------------------+                                                       
| gpiod_get_direction | : call ->get_direction() and save result in desc flags
+-|-------------------+                                                       
  |                                                                           
  |--> get gpio_chip from desc                                                
  |                                                                           
  |    +------------------+                                                   
  |--> | gpio_chip_hwgpio | get desc offset                                   
  |    +------------------+                                                   
  |                                                                           
  |--> call ->get_direction()                                                 
  |                                                                           
  +--> save direction in desc flags                                           
```

```
drivers/gpio/gpiolib.c                                                     
+-----------------------+                                                   
| gpiod_configure_flags | : set gpio direction accordingly                  
+-|---------------------+                                                   
  |                                                                         
  |--> given lflags, set bits in desc flags                                 
  |                                                                         
  |    +----------------------+                                             
  |--> | gpiod_set_transitory | label 'transitory' in desc, set config      
  |    +----------------------+                                             
  |                                                                         
  |--> return if dflag has no direction set                                 
  |                                                                         
  |--> if direction is out                                                  
  |    |                                                                    
  |    |    +------------------------+                                      
  |    +--> | gpiod_direction_output | write hw reg to set direction = 'out'
  |         +------------------------+                                      
  |                                                                         
  +--> else                                                                 
       |                                                                    
       |    +-----------------------+                                       
       +--> | gpiod_direction_input | write hw reg to set direction = 'in'  
            +-----------------------+                                       
```

```
+----------------------+                                                                          
| gpiod_set_transitory | : label 'transitory' in desc, set config                                 
+-|--------------------+                                                                          
  |                                                                                               
  |--> set 'transitory' bit in desc flags                                                         
  |                                                                                               
  |    +----------------------------------------+                                                 
  +--> | gpio_set_config_with_argument_optional | prepare config, call ->set_config() if it exists
       +----------------------------------------+                                                 
```

```
+----------------------------------------+                                                   
| gpio_set_config_with_argument_optional | : prepare config, call ->set_config() if it exists
+-|--------------------------------------+                                                   
  |    +------------------+                                                                  
  |--> | gpio_chip_hwgpio | get gpio (offset) from desc                                      
  |    +------------------+                                                                  
  |    +-------------------------------+                                                     
  +--> | gpio_set_config_with_argument | prepare config, call ->set_config() if it exists    
       +-------------------------------+                                                     
```

```
drivers/gpio/gpiolib.c                                                             
+-------------------------------+                                                   
| gpio_set_config_with_argument | : prepare config, call ->set_config() if it exists
+-|-----------------------------+                                                   
  |                                                                                 
  |--> get gpio_dev from desc                                                       
  |                                                                                 
  |    +--------------------------+                                                 
  |--> | pinconf_to_config_packed | config = make pair (mode, argument)             
  |    +--------------------------+                                                 
  |    +--------------------+                                                       
  +--> | gpio_do_set_config | call ->set_config() if it exists                      
       +--------------------+                                                       
```

```
drivers/gpio/gpiolib.c                                                           
+------------------------+                                                        
| gpiod_direction_output | : write hw reg to set direction = 'out'                
+-|----------------------+                                                        
  |                                                                               
  |--> adjust value                                                               
  |                                                                               
  |--> if open_drain is set in flags                                              
  |    |                                                                          
  |    |    +-----------------+                                                   
  |    +--> | gpio_set_config | set config 'open_drain'                           
  |         +-----------------+                                                   
  |                                                                               
  |--> elif open_source is set in flags                                           
  |    |                                                                          
  |    |    +-----------------+                                                   
  |    +--> | gpio_set_config | set config 'open_source'                          
  |         +-----------------+                                                   
  |--> else                                                                       
  |    |                                                                          
  |    |    +-----------------+                                                   
  |    +--> | gpio_set_config | set config 'push_pull'                            
  |         +-----------------+                                                   
  |    +---------------+                                                          
  |--> | gpio_set_bias | based on desc.flags, determine bias and set config       
  |    +---------------+                                                          
  |    +-----------------------------------+                                      
  +--> | gpiod_direction_output_raw_commit | write hw reg to set direction = 'out'
       +-----------------------------------+                                      
```

```
drivers/gpio/gpiolib.c                                                                           
+---------------+                                                                                 
| gpio_set_bias | : based on desc.flags, determine bias and set config                            
+-|-------------+                                                                                 
  |                                                                                               
  |--> determine bias_bit and arg based on desc.flags                                             
  |                                                                                               
  |    +----------------------------------------+                                                 
  +--> | gpio_set_config_with_argument_optional | prepare config, call ->set_config() if it exists
       +----------------------------------------+                                                 
```

```
drivers/gpio/gpiolib.c                                                      
+-----------------------------------+                                        
| gpiod_direction_output_raw_commit | : write hw reg to set direction = 'out'
+-|---------------------------------+                                        
  |                                                                          
  |--> if gpio_chip has ->direction_output                                   
  |    -                                                                     
  |    +--> call it, e.g.,                                                   
  |         +---------------------+                                          
  |         | aspeed_gpio_dir_out | write hw reg to set direction = 'out'    
  |         +---------------------+                                          
  |                                                                          
  |--> else                                                                  
  |    -                                                                     
  |    +--> (ignore)                                                         
  |                                                                          
  +--> label 'out' on desc.flags                                             
```

```
drivers/gpio/gpio-aspeed.c                                    
+---------------------+                                        
| aspeed_gpio_dir_out | : write hw reg to set direction = 'out'
+-|-------------------+                                        
  |                                                            
  |--> read hw reg and set target bit                          
  |                                                            
  |    +---------------------------+                           
  |--> | aspeed_gpio_copro_request | request the access (?)    
  |    +---------------------------+                           
  |    +-------------------+                                   
  |--> | __aspeed_gpio_set | set or clear bit, write to hw reg 
  |    +-------------------+                                   
  |                                                            
  |--> write to hw reg (different from the above?)             
  |                                                            
  +--> if co-processor is involved (?)                         
       |                                                       
       |    +---------------------------+                      
       +--> | aspeed_gpio_copro_release | (skip)               
            +---------------------------+                      
```

```
drivers/gpio/gpio-aspeed.c                                                       
+---------------------------+                                                     
| aspeed_gpio_copro_request | : request the access (?)                            
+-|-------------------------+                                                     
  |                                                                               
  |--> return if no co-processor involved?                                        
  |                                                                               
  |--> call ->request_access(), e.g.,                                             
  |    +-----------------------------+                                            
  |    | fsi_master_acf_gpio_request | write hw reg to request the access         
  |    +-----------------------------+                                            
  |                                                                               
  |    +-------------------------------+                                          
  |--> | aspeed_gpio_change_cmd_source | write hw reg to change source back to arm
  |    +-------------------------------+                                          
  |                                                                               
  +--> update gpio cache                                                          
```

```
drivers/gpio/gpio-aspeed.c                              
+-------------------+                                    
| __aspeed_gpio_set | : set or clear bit, write to hw reg
+-|-----------------+                                    
  |                                                      
  |--> given offset, get cached reg value                
  |                                                      
  |--> set or clear bit                                  
  |                                                      
  |--> update cache                                      
  |                                                      
  +--> write back to hw reg                              
```

```
drivers/leds/leds-gpio.c                                                                   
+-----------------+                                                                         
| create_gpio_led | : install ops, set gpio direction out, prepare led struct               
+-|---------------+                                                                         
  |                                                                                         
  |--> install func for ->brightness_set()                                                  
  |                                                                                         
  |--> install func for ->blink_set()                                                       
  |                                                                                         
  |--> given template, set flags                                                            
  |                                                                                         
  |    +------------------------+                                                           
  |--> | gpiod_direction_output | write hw reg to set direction = 'out'                     
  |    +------------------------+                                                           
  |                                                                                         
  |--> if template has 'name' set                                                           
  |    |                                                                                    
  |    |--> save name in led_dat                                                            
  |    |                                                                                    
  |    |    +----------------------------+                                                  
  |    +--> | devm_led_classdev_register | prepare led struct, add to device as resource    
  |         +----------------------------+                                                  
  |                                                                                         
  +--> else                                                                                 
       |                                                                                    
       |    +--------------------------------+                                              
       +--> | devm_led_classdev_register_ext | prepare led struct, add to device as resource
            +--------------------------------+                                              
```

```
include/linux/leds.h                                                                                      
+----------------------------+                                                                             
| devm_led_classdev_register | : prepare led struct, add to device as resource                             
+-|--------------------------+                                                                             
  |    drivers/leds/led-class.c                                                                            
  |    +--------------------------------+                                                                  
  +--> | devm_led_classdev_register_ext | : prepare led struct, add to device as resource                  
       +-|------------------------------+                                                                  
         |    +--------------+                                                                             
         |--> | devres_alloc | alloc a ptr                                                                 
         |    +--------------+                                                                             
         |    +---------------------------+                                                                
         |--> | led_classdev_register_ext | determine name, init led work & timer, find the matched trigger
         |    +---------------------------+                                                                
         |                                                                                                 
         |--> have the ptr point to led_cdev                                                               
         |                                                                                                 
         |    +------------+                                                                               
         +--> | devres_add |                                                                               
              +------------+                                                                               
```

```
drivers/leds/led-class.c                                                                      
+---------------------------+                                                                  
| led_classdev_register_ext | : determine name, init led work & timer, find the matched trigger
+-|-------------------------+                                                                  
  |                                                                                            
  |--> determine name                                                                          
  |                                                                                            
  |    +------------------------+                                                              
  |--> | led_classdev_next_name | ensure it's unique                                           
  |    +------------------------+                                                              
  |    +---------------------------+                                                           
  |--> | device_create_with_groups | create files in sysfs                                     
  |    +---------------------------+                                                           
  |                                                                                            
  |--> add led_cdev to list (leds_list)                                                        
  |                                                                                            
  |    +-----------------------+                                                               
  |--> | led_update_brightness | get brightness value and save in struct                       
  |    +-----------------------+                                                               
  |    +---------------+                                                                       
  |--> | led_init_core | init led work and timer                                               
  |    +---------------+                                                                       
  |    +-------------------------+                                                             
  +--> | led_trigger_set_default | find matched trigger from list, save in led_cdev            
       +-------------------------+                                                             
```

```
drivers/leds/led-core.c                                                    
+---------------+                                                           
| led_init_core | : init led work and timer                                 
+-|-------------+                                                           
  |    +-----------+ +------------------------+                             
  |--> | INIT_WORK | | set_brightness_delayed | set brightness              
  |    +-----------+ +------------------------+                             
  |    +-------------+ +--------------------+                               
  +--> | timer_setup | | led_timer_function | set brightness, add timer back
       +-------------+ +--------------------+                               
```

```
drivers/leds/led-core.c                                     
+------------------------+                                   
| set_brightness_delayed | : set brightness                  
+-|----------------------+                                   
  |                                                          
  |--> if 'disable' is set, clear it                         
  |    |                                                     
  |    |    +-------------------------+                      
  |    +--> | led_stop_software_blink | delete blinking timer
  |         +-------------------------+                      
  |    +----------------------+                              
  +--> | __led_set_brightness | set brightness               
       +----------------------+                              
```

```
drivers/leds/led-core.c                                      
+--------------------+                                         
| led_timer_function | : set brightness, add timer back        
+-|------------------+                                         
  |    +--------------------+                                  
  |--> | led_get_brightness |                                  
  |    +--------------------+                                  
  |                                                            
  |--> if it off now                                           
  |    -                                                       
  |    +--> get brightness and delay for switch-on preparation 
  |                                                            
  |--> else (on)                                               
  |    -                                                       
  |    +--> get brightness and delay for switch-off preparation
  |                                                            
  |    +----------------------------+                          
  |--> | led_set_brightness_nosleep |                          
  |    +----------------------------+                          
  |    +-----------+                                           
  +--> | mod_timer | add timer back to framework               
       +-----------+                                           
```

```
drivers/leds/led-triggers.c                                                  
+-------------------------+                                                   
| led_trigger_set_default | : find matched trigger from list, save in led_cdev
+-|-----------------------+                                                   
  |                                                                           
  +--> for each trigger on list (trigger_list)                                
       -                                                                      
       +--> if matched trigger found                                          
            |                                                                 
            |    +-----------------+                                          
            |--> | led_trigger_set | save trigger in led_cdev                 
            |    +-----------------+                                          
            |                                                                 
            +--> break                                                        
```

```
drivers/leds/led-triggers.c                              
+-----------------+                                       
| led_trigger_set | : save trigger in led_cdev            
+-|---------------+                                       
  |                                                       
  |--> if led_cdev has an existing trigger                
  |    |                                                  
  |    |--> remove led_cdev from where it is              
  |    |                                                  
  |    |    +------------------+                          
  |    |--> | cancel_work_sync | cancel led work          
  |    |    +------------------+                          
  |    |    +-------------------------+                   
  |    |--> | led_stop_software_blink | delete blink timer
  |    |    +-------------------------+                   
  |    |                                                  
  |    |--> if trigger->deactivate() exists, call it      
  |    |                                                  
  |    +--> reset led_cdev                                
  |                                                       
  +--> if arg trigger is provided                         
       |                                                  
       |--> add led_cdev to list (trigger)                
       |                                                  
       +--> if trigger->activate() exists, call it        
```

```
drivers/leds/leds-gpio.c                                                              
+------------------+                                                                   
| gpio_leds_create | : for each child, prepare gpio_desc and led_struct                
+-|----------------+                                                                   
  |    +-----------------------------+                                                 
  |--> | device_get_child_node_count | count number of child node                      
  |    +-----------------------------+                                                 
  |                                                                                    
  +--> for each child                                                                  
       |                                                                               
       |    +----------------------------------+                                       
       |--> | devm_fwnode_get_gpiod_from_child | get gpio desc from fw node            
       |    +----------------------------------+                                       
       |    +----------------------------+                                             
       |--> | led_init_default_state_get | determine led default state                 
       |    +----------------------------+                                             
       |    +-----------------+                                                        
       |--> | create_gpio_led | install ops, set gpio direction out, prepare led struct
       |    +-----------------+                                                        
       |    +-------------------------+                                                
       +--> | gpiod_set_consumer_name | set consumer name on gpio_desc                 
            +-------------------------+                                                
```

```
drivers/input/keyboard/gpio_keys.c
+-----------------+
| gpio_keys_probe | : prepare pdate & input_dev & ddata, for each button: set up key, register input_dev
+-|---------------+
  |            +-----------------------------+
  |--> pdata = | gpio_keys_get_devtree_pdata | alloc buffer, save property values of child nodes
  |            +-----------------------------+
  |
  |--> alloc ddata (?)
  |
  |--> alloc keymap
  |
  |            +----------------------------+
  |--> input = | devm_input_allocate_device | prepare input_dev, add to dev as resource
  |            +----------------------------+
  |
  |--> save pdata and input in ddata
  |
  |--> further set up input (name, ops, id, ...)
  |
  |--> for each button
  |    |
  |    |    +---------------------+
  |    +--> | gpio_keys_setup_key | get gpio_desc, set capability, request irq
  |         +---------------------+
  |    +-----------------------+
  |--> | input_register_device | set up input_dev, register it
  |    +-----------------------+
  |    +--------------------+
  +--> | device_init_wakeup | (skip)
       +--------------------+
```

```
drivers/input/keyboard/gpio_keys.c                                                
+-----------------------------+                                                    
| gpio_keys_get_devtree_pdata | : alloc buffer, save property values of child nodes
+-|---------------------------+                                                    
  |    +-----------------------------+                                             
  |--> | device_get_child_node_count | count number of child node                  
  |    +-----------------------------+                                             
  |                                                                                
  |--> alloc buffer  = pdata + button * n                                          
  |                                                                                
  +--> for each child node                                                         
       |                                                                           
       |--> button.code = value of property "linux,code"                           
       |                                                                           
       |--> button.desc = value of property "label"                                
       |                                                                           
       +--> button.xxx = value of property "ooo"                                   
```

```
drivers/input/input.c                                                    
+----------------------------+                                            
| devm_input_allocate_device | : prepare input_dev, add to dev as resource
+-|--------------------------+                                            
  |    +--------------+                                                   
  |--> | devres_alloc | alloc input_dev_resource                          
  |    +--------------+                                                   
  |    +-----------------------+                                          
  |--> | input_allocate_device | alloc input_dev and set it up            
  |    +-----------------------+                                          
  |                                                                       
  |--> save input_dev in resource                                         
  |                                                                       
  |    +------------+                                                     
  +--> | devres_add |                                                     
       +------------+                                                     
```

```
drivers/input/input.c                                  
+-----------------------+                                
| input_allocate_device | : alloc input_dev and set it up
+-|---------------------+                                
  |                                                      
  +--> alloc input_dev                                   
  |                                                      
  |--> set up device type & class                        
  |                                                      
  |    +-------------+                                   
  |--> | timer_setup | set up a null timer               
  |    +-------------+                                   
  |    +--------------+                                  
  +--> | dev_set_name | "input%lu"                       
       +--------------+                                  
```

```
drivers/input/keyboard/gpio_keys.c                                                                     
+---------------------+                                                                                 
| gpio_keys_setup_key | : get gpio_desc, set capability, request irq                                    
+-|-------------------+                                                                                 
  |                                                                                                     
  |--> if child is provided                                                                             
  |    |                                                                                                
  |    |    +-----------------------+                                                                   
  |    +--> | devm_fwnode_gpiod_get | get gpio desc                                                     
  |         +-----------------------+                                                                   
  |                                                                                                     
  |--> elif button has valid gpio#                                                                      
  |    |                                                                                                
  |    |    +-----------------------+                                                                   
  |    +--> | devm_gpio_request_one | request a gpio line and add to dev as resource                    
  |         +-----------------------+                                                                   
  |                                                                                                     
  |--> if gpio_desc is preppared                                                                        
  |    |                                                                                                
  |    |--> set up button_data                                                                          
  |    |                  +--------------------+                                                        
  |    +--> determine isr | gpio_keys_gpio_isr |                                                        
  |                       +--------------------+                                                        
  |--> else                                                                                             
  |    |                                                                                                
  |    |--> set up button_data                                                                          
  |    |                  +-------------------+                                                         
  |    +--> determine isr | gpio_keys_irq_isr |                                                         
  |                       +-------------------+                                                         
  |    +----------------------+                                                                         
  |--> | input_set_capability | label capabilities on input_dev                                         
  |    +----------------------+                                                                         
  |    +-----------------+                                                                              
  |--> | devm_add_action | add action to dev as resource                                                
  |    +-----------------+                                                                              
  |    +------------------------------+                                                                 
  +--> | devm_request_any_context_irq | request irq                                                     
       +------------------------------+                                                                 
```

## <a name="cheat-sheet"></a> Cheat Sheet

```
cd /sys/class/gpio
echo 123 > export
cd gpio123
cat direction # check current direction
echo out > direction
cat value
```

```
cat /sys/kernel/debug/gpio
```

## <a name="reference"></a> Reference

- [Linux kernel GPIO user space interface](https://embeddedbits.org/new-linux-kernel-gpio-user-space-interface/)
