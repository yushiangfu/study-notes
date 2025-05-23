> The note is based on Linux version 5.15.0 in OpenBMC.

## Index

- [Introduction](#introduction)
- [I2C and SMBus](#i2c-and-smbus)
- [Mux](#mux)
- [SMBus System Interface (SSIF)](#ssif)
- [Intelligent Platform Management Bus (IPMB)](#ipmb)
- [System Startup](#system-startup)
- [Cheat Sheet](#cheat-sheet)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

(TBD)

## <a name="i2c-and-smbus"></a> I2C and SMBus

The I2C protocol serves as a means for the master to communicate with devices on the bus. 
The layout of buses and devices is determined by motherboard designers, ensuring that each device on a bus has a unique address. 
When the master wants to communicate with a specific device, it broadcasts the unique address associated with that device, and only the targeted device will respond with an acknowledgment (ack), while others remain silent.

From the application's perspective, the process involves preparing a message and passing it to the device file. 
The driver then takes control and performs the necessary operations on the hardware. 
Messages can be of two types: read or write.

- For a read-type message, the following parameters are typically included:
  - Slave: Specifies the address of the device from which to read.
  - Flags: Indicates a read operation (1).
  - Len: Specifies the number of bytes to read.
  - Buf: Specifies the memory location where the data will be stored before being returned to the application.
- On the other hand, a write-type message includes the following parameters:
  - Slave: Specifies the address of the device to which to write.
  - Flags: Indicates a write operation (0).
  - Len: Specifies the number of bytes to write.
  - Buf: Specifies the memory location from which the data will be taken and copied to the I2C controller register.

<p align="center"><img src="images/i2c/framework.png" /></p>

<details><summary> More Details </summary>

```
                                                                                                                   
                                              addr=++           addr=??           addr=##           addr=@@         
                                                                                                                   
                                            +--------+        +--------+        +--------+        +--------+       
                                            | slave  |        | slave  |        | slave  |        | slave  |       
                                            +--------+        +--------+        +--------+        +--------+       
                                              |    |            |    |            |    |            |    |         
                                              |    |            |    |            |    |            |    |         
                                              |    |            |    |            |    |            |    |         
                                              |    |            |    |            |    |            |    |         
+---------+                                   |    |            |    |            |    |            |    |         
|      i2c-0              +-     sda  -------------|-----------------|-----------------|-----------------|---------
|         |          i2c  |   (data)               |                 |                 |                 |         
|      i2c-1  -----  bus  |                        |                 |                 |                 |         
|         |               +-     scl  -----------------------------------------------------------------------------
|         |                  (clock)                                                                               
| AST2500 |                                                                                                        
|         |                                                                                                        
|         |                                                                                                        
|      i2c-12                                                                                                      
|         |                                                                                                        
|      i2c-13                                                                                                      
+---------+                                                                                                        
```

```
                                                              +--------+                          +-------+
                                                              | master |                          | slave |
                                                              +--------+                          +-------+
                                                                                                           
                                                                       +--   +-+                           
                                                                       |     |s|           ---->           
                                                                       |     +-+                           
                                                                       |     +--------+-+                  
                                                                       |     |  addr  |w|  ---->           
                                                                       |     +--------+-+                  
                                                                       |                                   
                                                         i2c_msg[0]    |                     +-+           
                                                        +----------+   |     <----           |a|           
                                                        |  addr    |   |                     +-+           
                    +-----+                             |  flags   |   |                                   
                    | app |                             |  len     |   |     +----------+                  
                    +-----+                             |  *buf -------|->   |   data   |  ---->           
                     |  ｜                               +----------+   |     +----------+                  
                     | +-----+                                         |                     +-+           
                     | | lib | libi2c.so                               |     <----           |a|           
                     | +-----+                                         |                     +-+           
                     |  |                                              |               -                   
            I2C_RDWR |  | I2C_SMBUS                                    |               -                   
                   +------+                                            |               -                   
                   | cdev |                                            |     +-+                           
                   +------+                                            |     |p|           ---->           
                      |                                                +--   +-+                           
        user          |                                                                                    
        --------------|-------------                                   +--   +-+                           
        kernel        |                                                |     |s|           ---->           
                      |                                                |     +-+                           
                 +---------+                                           |     +--------+-+                  
 aspeed_i2c_algo | adapter |                                           |     |  addr  |r|  ---->           
                 +---------+                                           |     +--------+-+                  
                                                                       |                                   
                                                         i2c_msg[1]    |                     +-+           
                                                        +----------+   |     <----           |a|           
                                                        |  addr    |   |                     +-+           
                                                        |  flags   |   |                                   
                                                        |  len     |   |             +----------+          
                                                        |  *buf <------|--   <----   |   data   |          
                                                        +----------+   |             +----------+          
                                                                       |     +-+                           
                                                                       |     |a|           ---->           
                                                                       |     +-+       -                   
                                                                       |               -                   
                                                                       |               -                   
                                                                       |     +-+                           
                                                                       |     |p|           ---->           
                                                                       +--   +-+                           
```

```
static const struct file_operations i2cdev_fops = {
    .owner      = THIS_MODULE,
    .llseek     = no_llseek,
    .read       = i2cdev_read,
    .write      = i2cdev_write,
    .unlocked_ioctl = i2cdev_ioctl,
    .compat_ioctl   = compat_i2cdev_ioctl,
    .open       = i2cdev_open,
    .release    = i2cdev_release,
};
```

```
+-------------+                                                         
| i2cdev_open | : alloc i2c_client, relate adapter, and set to file priv
+---|---------+                                                         
    |    +-----------------+                                            
    |--> | i2c_get_adapter | get adapter from minor (adapter nr)        
    |    +-----------------+                                            
    |                                                                   
    |--> alloc i2c_client                                               
    |                                                                   
    +--> relate file priv, client, and adapter                          
```

```
+--------------+                                                                                              
| i2cdev_ioctl | : i2c ioctl                                                                                  
+---|----------+                                                                                              
    |                                                                                                         
    |--> get i2c client from file priv                                                                        
    |                                                                                                         
    |--> switch cmd                                                                                           
    |                                                                                                         
    |--> case slave                                                                                           
    +--> case slave_force                                                                                     
    |                                                                                                         
    |------> client addr = arg                                                                                
    |                                                                                                         
    |--> case ten bit                                                                                         
    |                                                                                                         
    |------> set or clear the flag of client                                                                  
    |                                                                                                         
    |--> case pec                                                                                             
    |                                                                                                         
    |------> set or clear the flag of client                                                                  
    |                                                                                                         
    |--> case funcs                                                                                           
    |                                                                                                         
    |------> get adapter functionalities                                                                      
    |                                                                                                         
    |--> case read write                                                                                      
    |                                                                                                         
    |        +-------------+                                                                                  
    |------> | memdup_user | alloc buffer and copy msg meta from user space                                   
    |        +-------------+                                                                                  
    |        +-------------------+                                                                            
    |------> | i2cdev_ioctl_rdwr | copy msg data from user, transfer i2c packet(s), copy msg data to user     
    |        +-------------------+                                                                            
    |                                                                                                         
    |--> case smbus                                                                                           
    |                                                                                                         
    |------> copy info from user space                                                                        
    |                                                                                                         
    |        +--------------------+                                                                           
    |------> | i2cdev_ioctl_smbus | copy smbus data from user, transfer i2c packet(s), copy smbus data to user
    |        +--------------------+                                                                           
    |                                                                                                         
    |--> case retries                                                                                         
    |                                                                                                         
    |------> config adapter->retries                                                                          
    |                                                                                                         
    |--> case timeout                                                                                         
    |                                                                                                         
    +------> config adapter->timeout                                                                          
```

```
+-------------------+                                                                         
| i2cdev_ioctl_rdwr | : copy msg data from user, transfer i2c packet(s), copy msg data to user
+----|--------------+                                                                         
     |                                                                                        
     |--> alloc buffer and copy msg data from user space                                      
     |                                                                                        
     |    +--------------+                                                                    
     |--> | i2c_transfer | lock bus, transfer i2c packet, unlock bus                          
     |    +--------------+                                                                    
     |                                                                                        
     +--> if flag specifies 'read', copy msg data to user space                               
```
  
```
+----------------+                                         
| __i2c_transfer | : transfer i2c packet                   
+---|------------+                                         
    |                                                      
    |--> while we can still retry                          
    |                                                      
    +------> call ->master_xfer(), e.g.,                   
    |        +------------------------+                    
    |        | aspeed_i2c_master_xfer | transfer i2c packet
    |        +------------------------+                    
    |                                                      
    +------> break if it's not 'again' error               
```
  
```
tatic const struct i2c_algorithm aspeed_i2c_algo = {
    .master_xfer    = aspeed_i2c_master_xfer,
    .functionality  = aspeed_i2c_functionality,
    .reg_slave  = aspeed_i2c_reg_slave,
    .unreg_slave    = aspeed_i2c_unreg_slave,
};
```
  
```
+------------------------+                                                            
| aspeed_i2c_master_xfer | : transfer i2c packet                                      
+-----|------------------+                                                            
      |                                                                               
      |--> if the bus is busy && it's a single master bus                             
      |                                                                               
      |        +------------------------+                                             
      |------> | aspeed_i2c_recover_bus | try to recover                              
      |        +------------------------+                                             
      |                                                                               
      |--> set up bus (msg, cnt, idx, err)                                            
      |                                                                               
      |    +---------------------+                                                    
      |--> | aspeed_i2c_do_start | prepare cmd, write slave addr & cmd to hw registers
      |    +---------------------+                                                    
      |    +----------------------------+                                             
      +--> | wait_for_completion_timeout|                                             
           +----------------------------+                                             
```

```
+---------------------+                                                      
| aspeed_i2c_do_start | : prepare cmd, write slave addr & cmd to hw registers
+-----|---------------+                                                      
      |                                                                      
      |--> set cmd to 'start | tx'                                           
      |                                                                      
      |--> set master state to 'master start'                                
      |                                                                      
      +--> if the curr msg is to read                                        
      |                                                                      
      |------> cmd |= 'rx'                                                   
      |                                                                      
      +--> write slave addr and cmd to hw registers                          
```
  
```
+---------------------------+                                                                  
| aspeed_i2c_ic_irq_handler | : the top i2c isr, for each set bit: call that bus's handler     
+------|--------------------+                                                                  
       |    +---------------------------+                                                      
       |--> | irq_desc_get_handler_data | get aspeed_i2c_ic from irq desc                      
       |    +---------------------------+                                                      
       |    +-------------------+                                                              
       |--> | irq_desc_get_chip | get irq chip, e.g., dummy_irq_chip                           
       |    +-------------------+                                                              
       |    +-------------------+                                                              
       |--> | chained_irq_enter | do nothing bc our chip is 'dummy_irq_chip'                   
       |    +-------------------+                                                              
       |                                                                                       
       |--> read status from register                                                          
       |                                                                                       
       |--> for each set bit (bus# = 14)                                                       
       |                                                                                       
       |        +---------------------------+                                                  
       |------> | generic_handle_domain_irq | given hwirq, get irq desc and call ->handle_irq()
       |        +---------------------------+                                                  
       |    +-----------------+                                                                
       +--> | chained_irq_exit| do nothing bc our chip is 'dummy_irq_chip'                     
            +-----------------+                                                                
```
  
```
drivers/i2c/busses/i2c-aspeed.c
+--------------------+
| aspeed_i2c_bus_irq | : ack, based on master state: write or read, if rx done: ack
+----|---------------+
     |
     |--> read interrupt bits
     |
     |--> ack all interrupts except rx done
     |
     |--> (ignore the case of slave controller)
     |
     |    +----------------------+
     +--> | aspeed_i2c_slave_irq | advance state machine and handle r/w request from host accordingly
     |    +----------------------+ (so called 'ssif')
     |
     +--> if there's remaining events
          |
          |    +-----------------------+
          +--> | aspeed_i2c_master_irq | based on master state,
               +-----------------------+ write to or read from hw reg for a byte, notify waiting task
```
  
```
drivers/i2c/busses/i2c-aspeed.c                                                                  
+----------------------+                                                                          
| aspeed_i2c_slave_irq | : advance state machine and handle r/w request from host accordingly     
+-|--------------------+                                                                          
  |                                                                                               
  |--> read cmd from hw reg                                                                       
  |                                                                                               
  |--> if slave is requested, proceed the state machine                                           
  |                                                                                               
  |--> if slave is inactive, return (maybe the interrupt is for controller as master mode)        
  |                                                                                               
  |--> switch slave_state                                                                         
  |    case read_requested                                                                        
  |    |    +-----------------+                                                                   
  |    |--> | i2c_slave_event | execute slave callback (e.g., handle read/write request from host)
  |    |    +-----------------+                                                                   
  |    +--> write hw reg to do tx                                                                 
  |                                                                                               
  |--> case read_processed                                                                        
  |    |    +-----------------+                                                                   
  |    |--> | i2c_slave_event |                                                                   
  |    |    +-----------------+                                                                   
  |    +--> write hw reg to do tx                                                                 
  |                                                                                               
  |--> case write_requested                                                                       
  |    -    +-----------------+                                                                   
  |    +--> | i2c_slave_event |                                                                   
  |         +-----------------+                                                                   
  |                                                                                               
  +--> case write_received                                                                        
       -    +-----------------+                                                                   
       +--> | i2c_slave_event |                                                                   
            +-----------------+                                                                   
```  
  
```
include/linux/i2c.h                                                                    
+-----------------+                                                                     
| i2c_slave_event | : execute slave callback (e.g., handle read/write request from host)
+-|---------------+                                                                     
  |                                                                                     
  +--> call ->slave_cb, e.g.,                                                           
       +-------------+                                                                  
       | ssif_bmc_cb | handle smbus read or write request (from host perspective)       
       +-------------+                                                                  
```

```
+-----------------------+                                                                                      
| aspeed_i2c_master_irq | : based on master state, write to or read from hw reg for a byte, notify waiting task
+-----|-----------------+                                                                                      
      |                                                                                                        
      |--> switch master state                                                                                 
      |                                                                                                        
      |--> case master_tx                                                                                      
      |                                                                                                        
      |--> case master_tx_first                                                                                
      |                                                                                                        
      |------> if msg still has bytes to send                                                                  
      |                                                                                                        
      |----------> write next byte to hw reg and send out                                                      
      |                                                                                                        
      |------> else                                                                                            
      |                                                                                                        
      |            +-----------------------------+                                                             
      |----------> | aspeed_i2c_next_msg_or_stop | start next msg if there's any, or otherwise stop            
      |            +-----------------------------+                                                             
      |                                                                                                        
      |--> case master_rx_first                                                                                
      |                                                                                                        
      |--> case master_rx                                                                                      
      |                                                                                                        
      +------> read data from hw register                                                                      
      |                                                                                                        
      |------> copy to msg buf                                                                                 
      |                                                                                                        
      |------> if msg expects more data                                                                        
      |                                                                                                        
      |----------> set master state to 'master rx'                                                             
      |                                                                                                        
      |----------> prepare rx cmd and write to hw register                                                     
      |                                                                                                        
      |------> else                                                                                            
      |                                                                                                        
      |            +-----------------------------+                                                             
      |----------> | aspeed_i2c_next_msg_or_stop | start next msg if there's any, or otherwise stop            
      |            +-----------------------------+                                                             
      |                                                                                                        
      |--> case stop: set master state to 'invactive'                                                          
      |                                                                                                        
      |--> case inactive: shouldn't receive interrupt                                                          
      |                                                                                                        
      |--> default: unknow, so set master state to inactive                                                    
      |                                                                                                        
      |    +----------+                                                                                        
      +--> | complete |                                                                                        
           +----------+                                                                                        
```

```
+-----------------------------+                                                              
| aspeed_i2c_next_msg_or_stop | : start next msg if there's any, or otherwise stop           
+-------|---------------------+                                                              
        |                                                                                    
        |--> if bus still has msg                                                            
        |                                                                                    
        |------> move to next msg                                                            
        |                                                                                    
        |        +---------------------+                                                     
        |------> | aspeed_i2c_do_start | prepare cmd, write slave addr & cmd to hw registers 
        |        +---------------------+                                                     
        |                                                                                    
        |--> else                                                                            
        |                                                                                    
        |        +--------------------+                                                      
        +------> | aspeed_i2c_do_stop | set master state to stop, write 'stop' to hw register
                 +--------------------+                                                      
```
  
</details>

### SMBus
  
The SMBus protocol shares similarities with I2C, but differs in ioctl argument structure and data stream.

- read_write: 
  - Similar to I2C flags, indicates transaction direction.
- command: 
  - SMBus interprets first received byte as command, unlike I2C.
- len: 
  - Instead of flexible length, it accepts predefined options like 
  - I2C_SMBUS_BYTE_DATA
  - I2C_SMBUS_WORD_DATA
  - I2C_SMBUS_BLOCK_DATA
- buf: 
  - Depending on len, I2C library may prepend accurate size information to byte array before ioctl.

The data structures for SMBus and I2C are different, and I2C drivers can choose to provide either two handlers or just one. 
To handle this, the kernel provides an emulation function that converts SMBus ioctl data into I2C messages before passing them to the driver. 
Let's consider an example with len=I2C_SMBUS_BLOCK_DATA and read_write=write:

1. The application initially calls a library API with command, length, and data arguments.
2. The library inserts the length at the beginning of the data and prepares the SMBus ioctl structure accordingly.
3. The len field in the structure is then set to I2C_SMBUS_BLOCK_DATA instead of the actual size.
4. During the structure conversion, the kernel adds the command byte to the data.
5. The data stream for I2C and SMBus differs, especially with the presence of the command byte and sometimes the length byte. 
   Therefore, it may be helpful to know the type of devices being used before selecting the transaction type in ioctl. 
   Note that outside this section, I continue to use the term 'I2C device' for convenience.

<p align="center"><img src="images/i2c/i2c-and-smbus.png" /></p>
  
<details><summary> More Details </summary>
  
```
  i2c control         |      smbus control                   i2c control
                      |                                      (emulation)
                      |                                                 
                      |                                                 
                      |                                                 
    i2c_msg           |     i2c_smbus_ioctl_data              i2c_msg   
   +--------+         |       +-----------+                  +--------+ 
   |  addr  |         |       |read_write |                  |  addr  | 
   | flags  |         |       |  command  |                  | flags  | 
   |  len   |         |       |   size    |                  |  len   | 
+---- *buf  |         |     +---- *data   |               +---- buf   | 
|  +--------+         |     | +-----------+               |  +--------+ 
|                     |     |                             |             
|                     |     |                             |             
|   +------+          |     |   +------+                  |   +------+  
+-->| data |          |     +---> len  |                  +-->| cmd  |  
    +------+          |         +------+                      +------+  
    | data |          |         | data |                      | len  |  
    +------+          |         +------+                      +------+  
    | data |          |         | data |                      | data |  
    +------+          |         +------+                      +------+  
        -             |         | data |                      | data |  
        -             |         +------+                      +------+  
    +------+          |             -                         | data |  
    | data |          |             -                         +------+  
    +------+          |         +------+                          -     
                      |         | data |                          -     
                      |         +------+                      +------+  
                      |                                       | data |  
                      |                                       +------+  
                      |                                                 
                      |                                                 
                      |                                                 
                      |                                                 
```
  
```
+--------------------+                                                                             
| i2cdev_ioctl_smbus | : copy smbus data from user, transfer i2c packet(s), copy smbus data to user
+----|---------------+                                                                             
     |                                                                                             
     |--> if it's 'quick' or 'write a byte'                                                        
     |                                                                                             
     |               +----------------+                                                            
     +------> return | i2c_smbus_xfer | lock, transfer i2c packet(s), unlock                       
     |               +----------------+                                                            
     |                                                                                             
     |--> copy smbus data from user space if necessary                                             
     |                                                                                             
     |    +----------------+                                                                       
     +--> | i2c_smbus_xfer | lock, transfer i2c packet(s), unlock                                  
     |    +----------------+                                                                       
     |                                                                                             
     +--> copy smbus data to user space if necessary                                               
```

```
+----------------+                                       
| i2c_smbus_xfer | : lock, transfer i2c packet(s), unlock
+---|------------+                                       
    |                                                    
    |--> lock                                            
    |                                                    
    |    +------------------+                            
    |--> | __i2c_smbus_xfer | transfer i2c packet(s)     
    |    +------------------+                            
    |                                                    
    +--> unlock                                          
```

```
+------------------+                                                      
| __i2c_smbus_xfer | : transfer i2c packet(s)                             
+----|-------------+                                                      
     |                                                                    
     |--> if adapter has ->smbus_xfer (not our case)                      
     |                                                                    
     |------> (ignore) and return                                         
     |                                                                    
     |    +-------------------------+                                     
     +--> | i2c_smbus_xfer_emulated | emulate smbus behavior by i2c method
          +-------------------------+                                     
```
  
</details>
  
## <a name="mux"></a> Mux
  
When dealing with I2C devices that have limited or non-configurable addresses, the number of such devices on a bus is restricted. 
In such cases, a multiplexer (mux) can be used. 
A mux is a specialized I2C device that splits a bus into multiple channels, allowing the I2C controller to access one channel at a time. 
The advantage is that the address space behind the mux is independent, enabling identical components to be placed on separate channels.

The mux mechanism revolves around adapters responsible for data transfer to connected devices. 
Both the controller and each channel of the mux serve as adapters, but the mux itself does not. 
The adapter hierarchy follows the hardware layout, with the controller as the root, followed by first-level channels, second-level channels, and so on. 
Each type of adapter has a different transfer function.

- Transfer logic of a controller-type adapter:
  - Calls the I2C driver, such as `aspeed_i2c_master_xfer()`, to perform the data transfer.
- Transfer logic of a channel-type adapter:
  - Calls the transfer() function of the parent adapter to select the desired channel.
  - Calls the transfer() function of the parent adapter to perform the data transaction.
  - Calls the transfer() function of the parent adapter to deselect the channel if necessary.
  
In the case where a channel-type adapter is used with a mux, the transfer logic allows for recursively switching between mux channels before the actual data transaction. 
Let's take an example from the `aspeed-bmc-ampere-mtjade.dts` file where the mux settings are specified. 
In this example, the root adapter is connected to two I2C devices: mux 0x70 and mux 0x71. 
Mux 0x71 is further divided into eight channels, and two of those channels are connected to mux 0x75.

When we want to communicate with an endpoint on a specific channel, the request for channel selection is sent recursively to the root adapter and fulfilled in reverse order. 
This means that the necessary channel switches are made starting from the top and going down the hierarchy. 
Similarly, the subsequent data transaction follows a similar recursive process. 
Finally, the root adapter forwards the I2C messages to the driver for processing.
  
<p align="center"><img src="images/i2c/mux.png" /></p>
  
<details><summary> More Details </summary>
  
```
                                   ch0 ---- nvme0                           ch0 ---- nvme8
                                   ch1 ---- nvme1                           ch1 ---- nvme9
                     ch3   +-----+ ch2 ---- nvme2            ch4   +-----+  ch2 ---- nvme10
               +---------- | mux | ch3 ---- nvme3      +---------- | mux |  ch3 ---- nvme11
               |           +-----+ ch4 ---- nvme4      |           +-----+  ch4 ---- nvme12
               |            slave  ch5 ---- nvme5      |            slave   ch5 ---- nvme13
               |            0x75   ch6 ---- nvme6      |            0x75    ch6 ---- nvme14
               |                   ch7 ---- nvme7      |                    ch7 ---- nvme15
               |                                       |
               |                                       |
               |                                       |                    ch0 ---- nvme16
               |                                       |                    ch1 ---- nvme17
               |                                       |     ch3   +-----+  ch2 ---- nvme18
               |                                       |   +------ | mux |  ch3 ---- nvme19
               |                                       |   |       +-----+  ch4 ---- nvme20
             +-----+                                  +-----+       slave   ch5 ---- nvme21
             | mux |                                  | mux |       0x75    ch6 ---- nvme22
             +--+--+                                  +--+--+               ch7 ---- nvme23
i2c-5           |slave                                   |slave
                |0x70                                    |0x71
                |                                        |
   +------------+----------------------------------------+---------------+----------------------------+
                                                                         |
                                                                         |slave
                                                                         |0x72
                                                                      +--+--+
                                                                      | mux |  ch0 ---- nvme_m2_0
                                                                      +-----+  ch1 ---- nvme_m2_1
```

```
                                                                                                                                         
                                                                 mux75                                                                   
                  +----------- mux70                            +-----+                                                                  
                  |                                             |  +--|---(adapter) ----- (client)                                       
                  |                                             |  +--|---(adapter) ----- (client)                                       
                  |            mux71                    (client)|  |--|---(adapter) ----- (client)                                       
                  |           +-----+                   +-------|--|--|---(adapter) ----- (client)                                       
                  |           |  +--|---(adapter)       |       |  |--|---(adapter) ----- (client)                                       
                  |           |  +--|---(adapter)       |       |  |--|---(adapter) ----- (client)                                       
        (adapter) |   (client)|  |--|---(adapter)       |       |  +--|---(adapter) ----- (client)                                       
 i2c-5 -----------|-----------|--|--|---(adapter) ---+  |       |  +--|---(adapter) ----- (client)                                       
                  |           |  |--|---(adapter) ---|--+       +-----+                                                                  
                  |           |  |--|---(adapter)    |                                                                                   
                  |           |  +--|---(adapter)    |                                                                                   
                  |           |  +--|---(adapter)    |                                                                                   
                  |           +-----+                |           mux75                                                                   
                  |                                  |          +-----+                                                                  
                  |                                  |          |  +--|---(adapter) ----- (client)                                       
                  +----------- mux72                 |          |  +--|---(adapter) ----- (client)                                       
                                                     |  (client)|  |--|---(adapter) ----- (client)                                       
                                                     +----------|--|--|---(adapter) ----- (client)                                       
                                                                |  |--|---(adapter) ----- (client)                                       
                                                                |  |--|---(adapter) ----- (client)                                       
                                                                |  +--|---(adapter) ----- (client)                                       
                                                                |  +--|---(adapter) ----- (client)                                       
                                                                +-----+                                                                  
                                                                                                                             i2c_mux_priv
                     client                                                                                                   +--------+ 
                   +---------+                                                       client                  +------------------ muxc  | 
      +--------------adapter |                                  i2c_mux_priv       +---------+               |                |chan_id | 
      |            +---------+                                   +--------+  +-------adapter |<-+       muxc v                |  algo  | 
      v                                         +------------------ muxc  |  |     +---------+  |   +----------+              |  adap  | 
   adapter           client                     |                |chan_id |  |           |------------ parent  | <--------+   +--------+ 
  +------+         +---------+                  v                |  algo  |  |           |      |   |  select  |          |              
  | algo | <---------adapter | <---+       muxc                  |  adap <---+ <---------+      |   | deselect |          |       -      
  +------+         +---------+     |   +----------+ <--------+   +--------+                     |   |   priv ------+      |       -      
           <-----------------------|----- parent  |          |                                  |   +----------+   |      |              
      ^              client        |   |  select  |          |       -                          |                  |      |  i2c_mux_priv
      |            +---------+     |   | deselect |          |       -                          |      pca954x  <--+      |   +--------+ 
      +--------------adapter |     |   |   priv ------+      |                                  |    +---------+          +------muxc  | 
                   +---------+     |   +----------+   |      |  i2c_mux_priv                    +--> | client  |              |chan_id | 
                                   |                  |      |   +--------+                          |  chip   |              |  algo  | 
                                   |      pca954x  <--+      +------muxc  |                          +---------+              |  adap  | 
                                   |    +---------+              |chan_id |                                                   +--------+ 
                                   +--> | client  |              |  algo  |                                                              
                                        |  chip   |              |  adap  |                                                              
                                        +---------+              +--------+                                                              
```
  
```
+---------------------+                                              
| pca954x_select_chan | : switch mux to target channel               
+-----|---------------+                                              
      |    +----------------+                                        
      |--> | pca954x_regval | given target channel, prepare reg value
      |    +----------------+                                        
      |                                                              
      |--> if last_chan != reg value                                 
      |                                                              
      |        +-------------------+                                 
      |------> | pca954x_reg_write |                                 
      |        +-------------------+                                 
      |                                                              
      +------> ->last_chan = reg value                               
```

```
+----------------------+                                                 
| pca954x_deselect_mux | : switch mux to predefined 'idle state' or 0    
+-----|----------------+                                                 
      |                                                                  
      |--> if data has defined 'idle state'                              
      |                                                                  
      |               +---------------------+                            
      |------> return | pca954x_select_chan | switch to that 'idle state'
      |               +---------------------+                            
      |                                                                  
      |--> if idle state == 'disconnect'                                 
      |                                                                  
      |------> ->last_state = 0                                          
      |                                                                  
      |        +-------------------+                                     
      +------> | pca954x_reg_write | switch to 0, no matter what it means
               +-------------------+                                     
```

```
+-----------------------+                                                         
| __i2c_mux_master_xfer | : switch mux, transfer i2c packet, switch mux to default
+-----|-----------------+                                                         
      |                                                                           
      |--> call ->select(), e.g.,                                                 
      |    +---------------------+                                                
      |    | pca954x_select_chan | switch mux to target channel                   
      |    +---------------------+                                                
      |                                                                           
      |    +----------------+                                                     
      |--> | __i2c_transfer | transfer i2c packet                                 
      |    +----------------+                                                     
      |                                                                           
      +--> call ->deselect(), e.g.,                                               
           +----------------------+                                               
           | pca954x_deselect_mux | switch mux to predefined 'idle state' or 0    
           +----------------------+                                               
```
  
</details>

## <a name="ssif"></a> SMBus System Interface (SSIF)

The host, functioning as the I2C master, transmits IPMI commands to the slave (BMC), which processes the requests and provides responses. 
The BMC initiates the process in its interrupt handler, dispatching events to the slave handler, reading registers for detailed events, and delivering them to the SSIF driver. 
SSIF events, in regular order, comprise:

- **I2C_SLAVE_WRITE_REQUESTED:** Host sends a write request to the BMC.
- **I2C_SLAVE_WRITE_RECEIVED:** Host writes IPMI request to the BMC.
- **I2C_SLAVE_STOP:** Host completes the write action, awaiting the BMC to handle and return a response.
- **I2C_SLAVE_READ_REQUESTED:** Host sends a read request to the BMC.
- **I2C_SLAVE_READ_PROCESSED:** Host reads IPMI response from the BMC.

In the BMC runtime environment, a user space application, such as `ssifbridge`, reads IPMI requests from the SSIF device and forwards them to the IPMI service for processing. 
The response then follows the path from which the request originated, eventually reaching the host.

<p align="center"><img src="images/i2c/ssif.png" /></p>
  
<details><summary> More Details </summary>

```
drivers/char/ipmi/ssif_bmc_aspeed.c                                                                  
+----------------+                                                                                    
| ssif_bmc_probe | : alloc ssif_bmc, register misc dev, prepare client for bus, enable slave mode     
+-|--------------+                                                                                    
  |    +----------------+                                                                             
  |--> | ssif_bmc_alloc | alloc ssif_bmc, register misc dev, prepare client for bus, enable slave mode
  |    +----------------+                                                                             
  |                                                                                                   
  |--> ssif_bmc.priv = adapter data                                                                   
  |                                                                                                   
  +--> install status function = aspeed_set_ssif_bmc_status                                           
```
  
```
+----------------+                                                                                    
| ssif_bmc_alloc | : alloc ssif_bmc, register misc dev, prepare client for bus, enable slave mode     
+-|--------------+                                                                                    
  |                                                                                                   
  |--> alloc 'ssif_bmc'                                                                               
  |                                                                                                   
  |--> set up misc dev (minor, name, fops=ssif_bmc_fops)                                              
  |                                                                                                   
  |    +---------------+                                                                              
  |--> | misc_register | determine dev#, add arg 'misc' to list (misc_list)                           
  |    +---------------+                                                                              
  |                                                                                                   
  |--> relate arg 'client' and ssif_bmc                                                               
  |                                                                                                   
  |    +--------------------+                                                                         
  +--> | i2c_slave_register | set up client (callback) and register to bus as slave, enable slave mode
       +--------------------+ +-------------+                                                         
                              | ssif_bmc_cb | handle smbus read or write request                      
                              +-------------+                                                         
```
  
```
drivers/char/misc.c                                                  
+---------------+                                                     
| misc_register | : determine dev#, add arg 'misc' to list (misc_list)
+-|-------------+                                                     
  |                                                                   
  |--> determine if minor# is dynamic selection                       
  |                                                                   
  |--> if is_dynamic                                                  
  |    |                                                              
  |    |    +---------------------+                                   
  |    |--> | find_first_zero_bit | find the first available one      
  |    |    +---------------------+                                   
  |    |                                                              
  |    +--> set bitmap to mark it's used                              
  |                                                                   
  |--> else                                                           
  |    -                                                              
  |    +--> check if the specified one is available                   
  |                                                                   
  |--> make dev# = (10, minor)                                        
  |                                                                   
  |    +---------------------------+                                  
  |--> | device_create_with_groups | create files in in sysfs         
  |    +---------------------------+                                  
  |                                                                   
  +--> add arg 'misc' into list (misc_list)                           
```
  
```
drivers/char/ipmi/ssif_bmc.c                                                                  
+-------------+                                                                                
| ssif_bmc_cb | : handle smbus read or write request (from host perspective)
+-|-----------+                                                                                
  |                                                                                            
  |--> switch event                                                                            
  |    case read_requested (master requests 'read')
  |    |                                                                                       
  |    |--> if is_single_part                                                                  
  |    |    |                                                                                  
  |    |    |    +--------------------------------+                                            
  |    |    +--> | set_singlepart_response_buffer | return length of ipmi response
  |    |         +--------------------------------+                                            
  |    +--> else                                                                               
  |         -    +-------------------------------+                                             
  |         +--> | set_multipart_response_buffer | copy data from payload to response_buf, return length of ipmi response
  |              +-------------------------------+                                             
  |                                                                                            
  |--> case write_requested (master requests 'write')                                                                   
  |    -                                                                                       
  |    +--> ssif_bmc.msg_ids = 0                                                               
  |                                                                                            
  |--> case read_processed (master reads a byte)                                                                     
  |    |                                                                                       
  |    |    +-----------------------+                                                          
  |    +--> | handle_read_processed | get a byte from ipmi response for master to read
  |         +-----------------------+                                                          
  |                                                                                            
  |--> case write_received (master writes a byte)                                                                     
  |    |                                                                                       
  |    |--> if msg_idx is 0                                                                    
  |    |    |                                                                                  
  |    |    |    +---------------------+                                                       
  |    |    +--> | initialize_transfer | smbus_cmd = arg value                                 
  |    |         +---------------------+                                                       
  |    +--> else                                                                               
  |         -    +-----------------------+                                                     
  |         +--> | handle_write_received | save a byte from master to request buffer
  |              +-----------------------+                                                     
  |--> case stop (master finishes the 'write' and waits for response from us)                                                                              
  |    -                                                                                       
  |    +--> if last_event == write_received                                                    
  |         |                                                                                  
  |         |     +-------------------------+                                                  
  |         +-->  | complete_write_received | validate pec, wake up task to handle and generate response
  |               +-------------------------+                                                  
  |                                                                                            
  +--> update ssif_bmc.last_event                                                              
```
  
```
drivers/char/ipmi/ssif_bmc.c
+-------------------------------+                             
| set_multipart_response_buffer | : copy data from payload to response_buf, return length of ipmi response
+-|-----------------------------+                             
  |                                                           
  |--> switch smbus_cmd                                       
  |                                                           
  |--> case read_start                                        
  |    -                                                      
  |    +--> set up response buffer (net_fn, cmd, payload, ...)
  |                                                           
  |--> case read_middle                                       
  |    |                                                      
  |    |--> if it's end (remain_len < 31)                     
  |    |    -                                                 
  |    |    +--> label 0xff (end) on response buffer          
  |    |                                                      
  |    |--> else (middle)                                     
  |    |    -                                                 
  |    |    +--> label block_num on response buffer           
  |    |                                                      
  |    +--> copy payload to response buffer                   
  |                                                           
  +--> update ssif_bmc.processed_bytes                        
```
  
```
drivers/char/ipmi/ssif_bmc.c                                                
+-----------------------+                                                    
| handle_read_processed | : get a byte from ipmi response for master to read
+-|---------------------+                                                    
  |                                                                          
  |--> calculate pec on start_read_addr, ssif_cmd, restart_write_addr        
  |                                                                          
  |--> if it's single_part_read                                              
  |    |                                                                     
  |    |--> determine value (buf, pec, or 0)                                 
  |    |                                                                     
  |    |    +-------------------+                                            
  |    +--> | complete_response | reset ssif_bmc, wake up task if there's any
  |         +-------------------+                                            
  |                                                                          
  +--> else                                                                  
       |                                                                     
       |--> get value from pec                                               
       |                                                                     
       |    +-------------------+                                            
       +--> | complete_response | reset ssif_bmc, wake up task if there's any
            +-------------------+                                            
```
  
```
drivers/char/ipmi/ssif_bmc.c                                                     
+---------------------+                                                           
| initialize_transfer | : smbus_cmd = arg value                                   
+-|-------------------+                                                           
  |                                                                               
  |--> smbus_cmd = arg value                                                      
  |                                                                               
  +--> if host sends a new request bc of timeout on bmc side                      
       -                                                                          
       +--> if response is in progress                                            
            |                                                                     
            |    +-------------------+                                            
            +--> | complete_response | reset ssif_bmc, wake up task if there's any
                 +-------------------+                                            
```
  
```
drivers/char/ipmi/ssif_bmc.c                   
+-----------------------+                       
| handle_write_received | : save a byte from master to request buffer
+-|---------------------+                       
  |                                             
  |--> get buffer from ssif_bmc.request         
  |                                             
  |--> switch smbus_cmd                         
  |                                             
  |--> case single_part_write                   
  |    -                                        
  |    +--> save value in buf, idx++            
  |                                             
  |--> case multi_part_write_start              
  |--> case multi_part_write_middle             
  +--> case multi_part_write_end                
       -                                        
       +--> get request len, idx++              
```
  
```
drivers/char/ipmi/ssif_bmc.c                                                   
+-------------------------+                                                     
| complete_write_received | : validate pec, wake up task to handle and generate response
+-|-----------------------+                                                     
  |    +--------------+                                                         
  |--> | validate_pec | validate checksum                                       
  |    +--------------+                                                         
  |                                                                             
  +--> if cmd == single_part_write || cmd == multi_part_write                   
       |                                                                        
       |    +----------------+                                                  
       +--> | handle_request | lable 'request available', wake up task to handle and generate response
            +----------------+                                                  
```
  
```
drivers/char/ipmi/ssif_bmc.c                                            
+----------------+                                                       
| handle_request | : lable 'request available', wake up task to handle and generate response
+-|--------------+                                                       
  |                                                                      
  |--> if ->set_ssif_bmc_status exists                                   
  |    -                                                                 
  |    +--> call it, e.g.,                                               
  |         +----------------------------+                               
  |         | aspeed_set_ssif_bmc_status | write hw reg to set bmc status
  |         +----------------------------+                               
  |                                                                      
  |--> label 'request_available' on ssif_bmc                             
  |                                                                      
  |--> reset response                                                    
  |                                                                      
  |    +-------------+                                                   
  +--> | wake_up_all | wake up waiting task if there's any               
       +-------------+                                                   
```
  
```
drivers/i2c/i2c-core-slave.c                                                                       
+--------------------+                                                                              
| i2c_slave_register | : set up client (callback) and register to bus as slave, enable slave mode   
+-|------------------+                                                                              
  |                                                                                                 
  |--> return error if i2c adapter doesn't support being a slave                                    
  |                                                                                                 
  |--> save callback in client                                                                      
  |                                                                                                 
  +--> call ->reg_slave(), e.g.,                                                                    
       +----------------------+                                                                     
       | aspeed_i2c_reg_slave | bus.slave = client, set it slave_addr into hw reg, enable slave mode
       +----------------------+                                                                     
```
  
```
drivers/i2c/busses/i2c-aspeed.c                                                               
+----------------------+                                                                       
| aspeed_i2c_reg_slave | : bus.slave = client, set it slave_addr into hw reg, enable slave mode
+-|--------------------+                                                                       
  |                                                                                            
  |--> get bus from adapter                                                                    
  |                                                                                            
  |--> bus.slave = arg client                                                                  
  |                                                                                            
  |    +------------------------+                                                              
  +--> | __aspeed_i2c_reg_slave | save its slave_addr into hw reg, enable slave mode           
       +------------------------+                                                              
```
  
```
drivers/char/ipmi/ssif_bmc.c
static const struct file_operations ssif_bmc_fops = {
    .owner      = THIS_MODULE,
    .read       = ssif_bmc_read,
    .write      = ssif_bmc_write,
    .poll       = ssif_bmc_poll,
    .unlocked_ioctl = ssif_bmc_ioctl,
};
```
  
```
drivers/char/ipmi/ssif_bmc.c                                     
+---------------+                                                 
| ssif_bmc_read | : wait for ssif request and copy to user        
+-|-------------+                                                 
  |    +--------------------------+                               
  |--> | receive_ssif_bmc_request | wait till request is available
  |    +--------------------------+                               
  |                                                               
  +--> copy data to user buffer                                   
```
  
```
drivers/char/ipmi/ssif_bmc.c                                           
+--------------------------+                                            
| receive_ssif_bmc_request | : wait till request is available           
+-|------------------------+                                            
  |                                                                     
  |--> if 'blocking' is specified                                       
 retry:|                                                                
  |    |    +--------------------------+                                
  |    +--> | wait_event_interruptible |                                
  |         +--------------------------+                                
  |                                                                     
  +--> if request isn't available at the point (after waking up)        
       |                                                                
       |                                                                
       |--> if 'non blocking' is specified                              
       |    -                                                           
       |    +--> return error 'again' (it's possibly woken up by signal)
       |                                                                
       +--> else, go to 'retry' (what happened?)                        
```
  
```
drivers/char/ipmi/ssif_bmc.c                                                
+----------------+                                                           
| ssif_bmc_write | : copy response from user, send it, set bmc status 'ready'
+-|--------------+                                                           
  |                                                                          
  |--> copy response from user buffer                                        
  |                                                                          
  |    +------------------------+                                            
  |--> | send_ssif_bmc_response | label 'response_in_progress'               
  |    +------------------------+                                            
  |                                                                          
  +--> if ->set_ssif_bmc_status() exists                                     
       |                                                                     
       +--> call it, e.g.,                                                   
            +----------------------------+                                   
            | aspeed_set_ssif_bmc_status | write hw reg to set bmc status    
            +----------------------------+                                   
```
  
```
drivers/char/ipmi/ssif_bmc.c                                           
+------------------------+                                              
| send_ssif_bmc_response | : label 'response_in_progress'               
+-|----------------------+                                              
  |                                                                     
  |--> if 'blocking' is specified                                       
  |    |                                                                
  |    |    +--------------------------+                                
  |    +--> | wait_event_interruptible |                                
  |         +--------------------------+                                
  |                                                                     
  |--> wait till the progress finishes                                  
  |                                                                     
  +--> label 'response_in_progress' (does anyone monitoring this field?)
```
  
### ssifbridge
  
```
ssifbridged.cpp                                                                                              
+------+                                                                                                      
| main |                                                                                                      
+-|----+                                                                                                      
  |                                                                                                           
  |--> request service name "xyz.openbmc_project.Ipmi.Channel.ipmi_ssif"                                      
  |                                                                                                           
  |    +--------------------------+                                                                           
  +--> | SsifChannel::SsifChannel | set up async read for request/response handling, add interface and init it
       +--------------------------+                                                                           
```
  
```
ssifbridged.cpp                                                                                         
+--------------------------+                                                                             
| SsifChannel::SsifChannel | : set up async read for request/response handling, add interface and init it
+-|------------------------+                                                                             
  |                                                                                                      
  |--> open "/dev/ipmi-ssif-host"                                                                        
  |                                                                                                      
  |    +-------------------------+                                                                       
  |--> | SsifChannel::async_read | set up async read (handle request and return response)                
  |    +-------------------------+                                                                       
  |                                                                                                      
  +--> for object: "/xyz/openbmc_project/Ipmi/Channel/ipmi_ssif"                                         
       add iface: "xyz.openbmc_project.Ipmi.Channel.ipmi_ssif"                                           
```
  
```
ssifbridged.cpp                                                                                                          
+-------------------------+                                                                                               
| SsifChannel::async_read | : set up async read (handle request and return response)                                      
+-|-----------------------+                                                                                               
  |                                                                                                                       
  +--> set up async read                                                                                                  
          +--------------------------------------------------------------------------------------------------------------+
          |+-----------------------------+                                                                               |
          || SsifChannel::processMessage | set up request (from host), send to ipmid, set up response, write to ssif dev |
          |+-----------------------------+                                                                               |
          +--------------------------------------------------------------------------------------------------------------+
```
  
```
ssifbridged.cpp
+-----------------------------+                                                                                          
| SsifChannel::processMessage | : set up request (from host), send to ipmid, set up response, write to ssif dev (to host)
+-|---------------------------+                                                                                          
  |    +-------------------------+                                                                                       
  +--> | SsifChannel::async_read | set up async_read for next time                                                       
  |    +-------------------------+                                                                                       
  |                                                                                                                      
  |--> get net_fn/lun/cmd from transfer buffer to set up 'prev_req_cmd'                                                  
  |                                                                                                                      
  |--> copy payload to local 'data'                                                                                      
  |                                                                                                                      
  +--> dbus-call service (ipmid) method                                                                                  
          +-----------------------------------------+                                                                    
          |service: "xyz.openbmc_project.Ipmi.Host" |                                                                    
          |object: "/xyz/openbmc_project/Ipmi"      |                                                                    
          |iface: "xyz.openbmc_project.Ipmi.Server" |                                                                    
          |method: "execute"                        |                                                                    
          +-------------------------------------------------------------------------------+                              
          |if net_fs/lun/cmd of response don't matcho those from the last request, return |                              
          |                                                                               |                              
          |set up response                                                                |                              
          |                                                                               |                              
          |write response to ssif device                                                  |                              
          +-------------------------------------------------------------------------------+                              
```
  
</details>

## <a name="ipmb"></a> Intelligent Platform Management Bus (IPMB)

<p align="center"><img src="images/i2c/ipmb.png" /></p>

<details><summary> More Details </summary>

```
drivers/char/ipmi/ipmb_dev_int.c                                                                     
+------------+                                                                                        
| ipmb_probe | : prepare ipmb_dev, register misc dev, register slave callback                         
+-|----------+                                                                                        
  |                                                                                                   
  |--> alloc ipmb_dev                                                                                 
  |                                                                                                   
  |--> setup misc_dev and install 'ipmb_fops'                                                         
  |    +-----------+                                                                                  
  |    | ipmb_read | get data from first element in req_queue, copy to user                           
  |    +-----------+                                                                                  
  |    +------------+                                                                                 
  |    | ipmb_write | get msg from user, send out i2c or smbus packet                                 
  |    +------------+                                                                                 
  |                                                                                                   
  |    +---------------+                                                                              
  |--> | misc_register | determine dev#, add arg 'misc' to list (misc_list)                           
  |    +---------------+                                                                              
  |                                                                                                   
  |--> check if property "i2c-protocol" is set in dt                                                  
  |                                                                                                   
  |    +--------------------+                                                                         
  +--> | i2c_slave_register | set up client (callback) and register to bus as slave, enable slave mode
       +--------------------+ +---------------+                                                       
                              | ipmb_slave_cb | assemble request, prepend to queue, wake up waiter(s) 
                              +---------------+                                                       
```

```
drivers/char/ipmi/ipmb_dev_int.c                                             
+-----------+                                                                 
| ipmb_read | : get data from first element in req_queue, copy to user        
+-|---------+                                                                 
  |                                                                           
  |--> while request_queue is empty                                           
  |    |                                                                      
  |    |    +--------------------------+                                      
  |    +--> | wait_event_interruptible | wait till request_queue has something
  |         +--------------------------+                                      
  |                                                                           
  |--> get first element from queue                                           
  |                                                                           
  |--> copy data from element to msg                                          
  |                                                                           
  |    +----------+                                                           
  |--> | list_del | remove element from queue                                 
  |    +----------+                                                           
  |    +--------------+                                                       
  +--> | copy_to_user |                                                       
       +--------------+                                                       
```

```
drivers/char/ipmi/ipmb_dev_int.c                                  
+------------+                                                     
| ipmb_write | : get msg from user, send out i2c or smbus packet   
+-|----------+                                                     
  |    +----------------+                                          
  |--> | copy_from_user | copy msg from user                       
  |    +----------------+                                          
  |                                                                
  |--> if ipmb_dev supports i2c protocol (not smbus)               
  |    |                                                           
  |    |    +----------------+                                     
  |    |--> | ipmb_i2c_write | setup i2c msg and trasfer i2c packet
  |    |    +----------------+                                     
  |    +--> return                                                 
  |                                                                
  |--> get slave_addr and net_lun from msg                         
  |                                                                
  |    +----------------------------+                              
  +--> | i2c_smbus_write_block_data |                              
       +----------------------------+                              
```

```
drivers/char/ipmi/ipmb_dev_int.c                                                                                            
+---------------+                                                                                                            
| ipmb_slave_cb | : assemble request, prepend to queue, wake up waiter(s)                                                    
+-|-------------+                                                                                                            
  |                                                                                                                          
  +--> switch event                                                                                                          
                                                                                                                             
       case write_requested                                                                                                  
       -                                                                                                                     
       +--> buf[1] = client addr                                                                                             
                                                                                                                             
       case write_received                                                                                                   
       -                                                                                                                     
       +--> buf[+idx] = val                                                                                                  
                                                                                                                             
       case stop                                                                                                             
       |                                                                                                                     
       |    +---------------------+                                                                                          
       +--> | ipmb_handle_request | alloc queue_elem, copy request from ipmb_dev, prepend to request_queue, wake up waiter(s)
            +---------------------+                                                                                          
```

```
drivers/char/ipmi/ipmb_dev_int.c                                                                                  
+---------------------+                                                                                            
| ipmb_handle_request | : alloc queue_elem, copy request from ipmb_dev, prepend to request_queue, wake up waiter(s)
+-|-------------------+                                                                                            
  |                                                                                                                
  |--> alloc queue_elem                                                                                            
  |                                                                                                                
  |--> copy request from ipmb_dev to queue_elem                                                                    
  |                                                                                                                
  |    +----------+                                                                                                
  |--> | list_add | prepend queue_elem to request_queue                                                            
  |    +----------+                                                                                                
  |    +-------------+                                                                                             
  +--> | wake_up_all | wake up whichever waiting on the queue                                                      
       +-------------+                                                                                             
```

</details>
  
## <a name="system-startup"></a> System Startup
  
During kernel startup, functions build the I2C framework and log relevant information.
  
- `aspeed_i2c_ic_of_init`
  - I2C buses share a single interrupt line, with the parent interrupt service routine (ISR) being prepared.
  - Upon receiving an interrupt, the parent ISR reads from the hardware and switches to the corresponding bus handler.
- `i2c_dev_init`
  - A range of device numbers is reserved for I2C character devices (cdev), with a callback function being attached.
  - When an adapter is detected, the callback function assists in adding its cdev to the object map.
- `aspeed_i2c_probe_bus`
  - I2C bus registrations from the device tree eventually invoke the probing function, which sets up an adapter within the I2C framework.
  - Bus 0 and bus 13 are not visible as they are disabled in the device tree and excluded from the process.
  
```
start_kernel
-
+--> init_IRQ
     -
     +--> irqchip_init
          -
          +--> of_irq_init
               -
               +--> aspeed_i2c_ic_of_init                [    0.000000] i2c controller registered, irq 17


kernel_init
-
+--> kernel_init_freeable
     -
     +--> do_basic_setup
          -
          +--> do_initcalls
               |
               |--> of_platform_default_populate_init    (parse dtb and register devices)
               |
               |--> i2c_dev_init                         [    2.073894] i2c_dev: i2c /dev entries driver
               |
               +--> aspeed_i2c_bus_driver_init
                    |
                    |--> aspeed_i2c_probe_bus            [    2.076731] aspeed-i2c-bus 1e78a080.i2c-bus: i2c bus 1 registered, irq 35
                    |--> aspeed_i2c_probe_bus            [    2.078057] aspeed-i2c-bus 1e78a0c0.i2c-bus: i2c bus 2 registered, irq 36
                    |--> aspeed_i2c_probe_bus            [    2.080005] aspeed-i2c-bus 1e78a100.i2c-bus: i2c bus 3 registered, irq 37
                    |--> aspeed_i2c_probe_bus            [    2.081004] aspeed-i2c-bus 1e78a140.i2c-bus: i2c bus 4 registered, irq 38
                    |--> aspeed_i2c_probe_bus            [    2.082583] aspeed-i2c-bus 1e78a180.i2c-bus: i2c bus 5 registered, irq 39
                    |--> aspeed_i2c_probe_bus            [    2.083489] aspeed-i2c-bus 1e78a1c0.i2c-bus: i2c bus 6 registered, irq 40
                    |--> aspeed_i2c_probe_bus            [    2.084395] aspeed-i2c-bus 1e78a300.i2c-bus: i2c bus 7 registered, irq 41
                    |--> aspeed_i2c_probe_bus            [    2.085306] aspeed-i2c-bus 1e78a340.i2c-bus: i2c bus 8 registered, irq 42
                    |--> aspeed_i2c_probe_bus            [    2.086268] aspeed-i2c-bus 1e78a380.i2c-bus: i2c bus 9 registered, irq 43
                    |--> aspeed_i2c_probe_bus            [    2.087459] aspeed-i2c-bus 1e78a3c0.i2c-bus: i2c bus 10 registered, irq 44
                    |--> aspeed_i2c_probe_bus            [    2.096190] aspeed-i2c-bus 1e78a400.i2c-bus: i2c bus 11 registered, irq 45
                    +--> aspeed_i2c_probe_bus            [    2.097970] aspeed-i2c-bus 1e78a440.i2c-bus: i2c bus 12 registered, irq 46
```
  
<details><summary> More Details </summary>
                                                                                     
```
[init sequence]
aspeed_i2c_ic_of_init      : [X] irqchip related; not part of i2c driver
i2c_init                   : [O]
pca953x_init               : [X] gpio related
tpm_tis_i2c_driver_init    : [X] tpm related; it's the i2c user instead of driver
mctp_i2c_mod_init          : [X] netdev related
i2c_dev_init               : [O]
aspeed_i2c_bus_driver_init : [O]
fsi_i2c_driver_init        : [X] skip, no device registered
i2c_mux_gpio_driver_init   : [X] skip, no device registered
pca9541_driver_init        : [O]
pca954x_driver_init        : [O] 
pca955x_driver_init        : [X] led related
bmp280_i2c_driver_init     : [X] skip, no device registered
```

```
+-----------------------+                                                          
| aspeed_i2c_ic_of_init |                                                          
+-----|-----------------+                                                          
      |                                                                            
      |--> set up i2c interrupt controller (i2c_ic)                                
      |                                                                            
      |    +----------+                                                            
      |--> | of_iomap | read register base from DTS/DTB                            
      |    +----------+                                                            
      |    +----------------------+                                                
      +--> | irq_of_parse_and_map | get parent irq                                 
      |    +----------------------+                                                
      |    +-----------------------+                                               
      |--> | irq_domain_add_linear | register domain                               
      |    +-----------------------+                                               
      |    +----------------------------------+                                    
      |--> | irq_set_chained_handler_and_data | handler = aspeed_i2c_ic_irq_handler
      |    +----------------------------------+                                    
      |                                                                            
      +--> print "i2c controller registered, irq 17"
```
  
```
+----------+                                      
| i2c_init | : register i2c bus and dummy driver  
+--|-------+                                      
   |    +--------------+                          
   |--> | bus_register | register 'i2c_bus_type'  
   |    +--------------+                          
   |    +----------------+                        
   +--> | i2c_add_driver | register a dummy driver
        +----------------+                        
```

```
+--------------+                                                                                 
| i2c_dev_init | : create i2c class and track addition/removal of adapters                       
+---|----------+                                                                                 
    |                                                                                            
    +--> print "i2c /dev entries driver"                                                         
    |                                                                                            
    |    +------------------------+                                                              
    |--> | register_chrdev_region | reserve the specified dev# range                             
    |    +------------------------+                                                              
    |    +--------------+                                                                        
    |--> | class_create | "i2c-dev"                                                              
    |    +--------------+                                                                        
    |    +-----------------------+                                                               
    |--> | bus_register_notifier | track the addition and removal of adapters                    
    |    +-----------------------+ 'i2cdev_notifier_call'                                                              
    |                                                                                            
    |--> for each i2c dev                                                                        
    |                                                                                            
    |        +-----------------------+                                                           
    +------> | i2cdev_attach_adapter | bind to existing adapters, but there's none at the momemnt
             +-----------------------+                                                           
```

```
+----------------------+                                                                   
| i2cdev_notifier_call | : action handler, e.g., if 'add device': prepare and register cdev
+-----|----------------+                                                                   
      |                                                                                    
      |--> switch action                                                                   
      |                                                                                    
      |--> case 'add device'                                                               
      |                                                                                    
      |        +-----------------------+                                                   
      |------> | i2cdev_attach_adapter | alloc i2c_dev, init as cdev and register it       
      |        +-----------------------+                                                   
      |                                                                                    
      |--> case 'del device'                                                               
      |                                                                                    
      |        +-----------------------+                                                   
      +------> | i2cdev_detach_adapter | (skip)                                            
               +-----------------------+                                                   
```

```
+-----------------------+                                                 
| i2cdev_attach_adapter | : alloc i2c_dev, init as cdev and register it   
+-----|-----------------+                                                 
      |                                                                   
      |--> return if dev type != 'i2c_adapter_type'                       
      |                                                                   
      |    +----------------+                                             
      |--> | to_i2c_adapter | get adapter that contains the dev           
      |    +----------------+                                             
      |    +------------------+                                           
      |--> | get_free_i2c_dev | alloc i2c_dev and add to 'i2c_dev_list'   
      |    +------------------+                                           
      |    +-----------+                                                  
      |--> | cdev_init | init dev with 'i2cdev_fops'                      
      |    +-----------+                                                  
      |    +-------------------+                                          
      |--> | device_initialize |                                          
      |    +-------------------+                                          
      |                                                                   
      |--> determine dev_t  = (i2c_major, adapter_nr)                     
      |                                                                   
      |--> set up i2c_dev->dev                                            
      |                                                                   
      |    +--------------+                                               
      |--> | dev_set_name | "i2c-%d"                                      
      |    +--------------+                                               
      |    +-----------------+                                            
      +--> | cdev_device_add | add cdev to kobj_map and register inner dev
           +-----------------+                                            
```

```
parent ---->   bus@1e78a000 {
                   compatible = "simple-bus";
                   #address-cells = <0x01>;
                   #size-cells = <0x01>;
                   ranges = <0x00 0x1e78a000 0x1000>;

    target ---->   i2c-bus@80 {
                       #address-cells = <0x01>;
                       #size-cells = <0x00>;
                       #interrupt-cells = <0x01>;
                       reg = <0x80 0x40>;       <------- reg addr = 0x1e78a000 (parent) + 0x80 (child), size is 0x40
                       compatible = "aspeed,ast2500-i2c-bus";
                       clocks = <0x02 0x1a>;
                       resets = <0x02 0x07>;
                       bus-frequency = <0x186a0>;
                       interrupts = <0x01>;     <------- local hwirq 1 is mapped to virq 35 (non-linear mapping)
                       interrupt-parent = <0x1c>;
                       status = "okay";
                   };
```

```
+----------------------+                                                                    
| aspeed_i2c_probe_bus | : map iomem, init controller, register isr, register adapter       
+-----|----------------+                                                                    
      |                                                                                     
      |--> alloc bus                                                                        
      |                                                                                     
      |    +-----------------------+                                                        
      |--> | platform_get_resource | get iomem info from resource                           
      |    +-----------------------+                                                        
      |    +-----------------------+                                                        
      |--> | devm_ioremap_resource | map iomem                                              
      |    +-----------------------+                                                        
      |    +-------------------------------+                                                
      |--> | devm_reset_control_get_shared | (skip)                                         
      |    +-------------------------------+                                                
      |    +------------------------+                                                       
      |--> | reset_control_deassert | (skip)                                                
      |    +------------------------+                                                       
      |                                                                                     
      |--> set up adapter and install ops 'aspeed_i2c_algo'                                 
      |                                                                                     
      |--> clear interrupt if there's any (hw)                                              
      |                                                                                     
      |    +-----------------+                                                              
      |--> | aspeed_i2c_init | init i2c controller (hw)                                     
      |    +-----------------+                                                              
      |    +----------------------+                                                         
      |--> | irq_of_parse_and_map | get irq from of of_node                                 
      |    +----------------------+                                                         
      |    +------------------+                                                             
      |--> | devm_request_irq | prepare 'action' (handler, thread_fn, ...) and install to irq desc
      |    +------------------+ (ack, based on master state: write or read, if rx done: ack)
      |    +-----------------+                                                              
      |--> | i2c_add_adapter | determine adapter id and register it                         
      |    +-----------------+                                                              
      |                                                                                     
      +--> print "i2c bus %d registered, irq %d\n"                                          
```

```
+-----------------+                           
| aspeed_i2c_init | : init i2c controller (hw)
+----|------------+                           
     |                                        
     |--> disable everything (hw)             
     |                                        
     |    +---------------------+             
     |--> | aspeed_i2c_init_clk |             
     |    +---------------------+             
     |                                        
     |--> enable master mode (hw)             
     |                                        
     +--> enable interrupt (hw)               
```

```
+-----------------+                                                                                                 
| i2c_add_adapter | : determine adapter id and register it                                                          
+----|------------+                                                                                                 
     |                                                                                                              
     |--> if of node has specified the adapter# (not our case)                                                      
     |                                                                                                              
     |------> read and set to adapter                                                                               
     |                                                                                                              
     |        +----------------------------+                                                                        
     |------> | __i2c_add_numbered_adapter | :                                                                      
     |        +------|---------------------+                                                                        
     |               |                                                                                              
     |               |--> ensure # isn't used                                                                       
     |               |                                                                                              
     |               |    +----------------------+                                                                  
     |               +--> | i2c_register_adapter | set name/bus/type, register dev and of_node, install recovery ops
     |                    +----------------------+                                                                  
     |                                                                                                              
     |------> return                                                                                                
     |                                                                                                              
     |    +-----------+                                                                                             
     |--> | idr_alloc | get an unused id                                                                            
     |    +-----------+                                                                                             
     |                                                                                                              
     |--> set to adapter                                                                                            
     |                                                                                                              
     |    +----------------------+                                                                                  
     +--> | i2c_register_adapter | set name/bus/type, register dev and of_node, install recovery ops                
          +----------------------+                                                                                  
```

```
+----------------------+                                                                                               
| i2c_register_adapter | : set name/bus/type, register dev and of_node, install recovery ops                           
+-----|----------------+                                                                                               
      |                                                                                                                
      |--> ensure adapter has installed 'i2c_adapter_lock_ops' and 'i2c_adapter_mux_root_ops'                          
      |                                                                                                                
      |    +----------------------------------+                                                                        
      |--> | i2c_setup_host_notify_irq_domain | (ignore, our adapter probably won't support I2C_FUNC_SMBUS_HOST_NOTIFY)
      |    +----------------------------------+                                                                        
      |    +--------------+                                                                                            
      |--> | dev_set_name | "i2c-%d"                                                                                   
      |    +--------------+                                                                                            
      |                                                                                                                
      |--> dev.bus = i2c_bus_type                                                                                      
      |                                                                                                                
      |--> dev.type = i2c_adapter_type                                                                                 
      |                                                                                                                
      |    +-----------------+                                                                                         
      |--> | device_register |                                                                                         
      |    +-----------------+                                                                                         
      |    +--------------------------+                                                                                
      |--> | of_i2c_setup_smbus_alert | do nothing bc of disabled config                                               
      |    +--------------------------+                                                                                
      |    +-------------------+                                                                                       
      |--> | i2c_init_recovery | install recovery ops                                                                  
      |    +-------------------+                                                                                       
      |    +-------------------------+                                                                                 
      |--> | of_i2c_register_devices |                                                                                 
      |    +-------------------------+                                                                                 
      |    +----------------------------+                                                                              
      |--> | i2c_scan_static_board_info | check if adapter num == bus num, this is an error                            
      |    +----------------------------+                                                                              
      |                                                                                                                
      |--> for each driver registered to 'i2c_bus_type'                                                                
      |                                                                                                                
      |        +-----------------------+                                                                               
      +------> | __process_new_adapter | (skip, seems it's only used by hwmon)                                         
               +-----------------------+                                                                               
```
  
```
struct device_type i2c_adapter_type = {
    .groups     = i2c_adapter_groups,
    .release    = i2c_adapter_dev_release,
};


static const struct attribute_group *i2c_adapter_groups[] = {   \    -+
    &i2c_adapter_group,                     \                         | generated
    NULL,                           \                                 | by macro
}                                                                    -+


static const struct attribute_group i2c_adapter_group = {       \    -+
    .attrs = i2c_adapter_attrs,                 \                     | generated
};                                                                   -+ by macro


static struct attribute *i2c_adapter_attrs[] = {
    &dev_attr_name.attr,
    &dev_attr_new_device.attr,
    &dev_attr_delete_device.attr,
    NULL
};


struct device_attribute dev_attr_new_device = {     \                -+
    .attr = {.name = "new_device",                \                   |
        .mode = 0x200 },     \                                        | generated
    .store  = new_device_store,                       \               | by macro
}                                                                    -+


struct device_attribute dev_attr_delete_device = {     \             -+
    .attr = {.name = "delete_device",                \                |
        .mode = S_IWUSR },     \                                      | generated
    .show   = NULL,                        \                          | by macro
    .store  = delete_device_store,                       \            |
}                                                                    -+
```

```
+------------------+                                                                   
| new_device_store | : get (type, addr) from buf, register new i2c client dev          
+----|-------------+                                                                   
     |    +----------------+                                                           
     |--> | to_i2c_adapter | get adapter from dev                                      
     |    +----------------+                                                           
     |                                                                                 
     |--> parse arg buf to get (type, addr)                                            
     |                                                                                 
     |    +-----------------------+                                                    
     |--> | i2c_new_client_device | prepare 'i2c client' and register device           
     |    +-----------------------+ (which potentially triggers other i2c driver probe)
     |    +---------------+                                                            
     |--> | list_add_tail | add client dev to the list of adapter                      
     |    +---------------+                                                            
     |                                                                                 
     +--> print "%s: Instantiated device %s at 0x%02hx\n", "new_device"                
```
  
```
+---------------------+                                       
| delete_device_store | : remove i2c client dev from adapter  
+-----|---------------+                                       
      |                                                       
      |--> parse addr from arg buf                            
      |                                                       
      |--> for each client of adapter                         
      |                                                       
      |------> if addr matches                                
      |                                                       
      |----------> print "%s: Deleting device %s at 0x%02hx\n"
      |                                                       
      |            +----------+                               
      |----------> | list_del | remove client dev from adapter
      |            +----------+                               
      |            +-----------------------+                  
      +----------> | i2c_unregister_device |                  
                   +-----------------------+                  
```
  
```
+--------------------------+
| of_i2c_register_devices  | : for each child of adapter: register i2c client device (might trigger pca954x_probe)
+------|-------------------+
       |
       |--> for each child of adapter
       |
       |        +------------------------+
       +------> | of_i2c_register_device | get slave addr of child, prepare i2c client and register it
                +------------------------+ (might trigger pca954x_probe)                    
```

```
+------------------------+                                                                                              
| of_i2c_register_device | : get slave addr of child, prepare i2c client and register it (might trigger pca954x_probe)  
+-----|------------------+                                                                                              
      |    +-----------------------+                                                                                    
      |--> | of_i2c_get_board_info | get property 'reg' as slave addr, save to arg info                                 
      |    +-----------------------+                                                                                    
      |    +-----------------------+                                                                                    
      +--> | i2c_new_client_device | prepare 'i2c client' and register device (which potentially triggers pca954x_probe)
           +-----------------------+                                                                                    
```

```
+-----------------------+                                                                                      
| i2c_new_client_device | : prepare 'i2c client' and register device (which potentially triggers pca954x_probe)
+-----|-----------------+                                                                                      
      |                                                                                                        
      |--> alloc 'client' (in our case, it's a mux)                                                            
      |                                                                                                        
      |--> set up 'client'                                                                                     
      |                                                                                                        
      |    +---------------------+                                                                             
      |--> | i2c_check_addr_busy | check if the slave addr is used                                             
      |    +---------------------+                                                                             
      |    +-----------------+                                                                                 
      +--> | device_register | add to bus, send 'add dev' to notifier, probe device                            
           +-----------------+                                         e.g., pca954x_probe                     
```

```
+---------------+                                                                                          
| pca954x_probe |                                                                                          
+---|-----------+                                                                                          
    |    +---------------+                                                                                 
    |--> | i2c_mux_alloc | alloc mux core (muxc) and install ops                                           
    |    +---------------+ +---------------------+                                                         
    |                      | pca954x_select_chan | switch mux to target channel                            
    |                      +----------------------+                                                        
    |                      | pca954x_deselect_mux | switch mux to predefined 'idle state' or 0             
    |                      +----------------------+                                                        
    |                                                                                                      
    |--> relate client dev and mux                                                                         
    |                                                                                                      
    |    +-------------------------+                                                                       
    |--> | devm_gpiod_get_optional | reset mux if client dev asks so                                       
    |    +-------------------------+                                                                       
    |                                                                                                      
    |--> determine mux chip (e.g., 9542? 9543?)                                                            
    |                                                                                                      
    |--> set idle state to 'idle as is' by default                                                         
    |                                                                                                      
    |--> further set to 'disconnect if dev property specifies                                              
    |                                                                                                      
    |    +--------------+                                                                                  
    |--> | pca954x_init | determine initial ->last_chan and switch to it                                   
    |    +--------------+                                                                                  
    |    +-------------------+                                                                             
    |--> | pca954x_irq_setup | ???                                                                         
    |    +-------------------+                                                                             
    |                                                                                                      
    |--> for each mux channel                                                                              
    |                                                                                                      
    |        +---------------------+                                                                       
    |------> | i2c_mux_add_adapter | prepare adaptor, install special ops, and register it                 
    |        +---------------------+                                                                       
    |                                                                                                      
    |--> if ->irq has value                                                                                
    |                                                                                                      
    |        +---------------------------+                                                                 
    |------> | devm_request_threaded_irq | install isr 'pca954x_irq_handler', which read a byte from client
    |        +---------------------------+                                                                 
    |    +----------+                                                                                      
    +--> | dev_info | print "registered %d multiplexed busses for I2C %s %s\n"                             
         +----------+                                                                                      
```

```
+---------------------+                                                                    
| i2c_mux_add_adapter | : prepare adaptor, install special ops, and register it            
+-----|---------------+                                                                    
      |                                                                                    
      |--> alloc priv for adapter                                                          
      |                                                                                    
      |--> relate it to muxc and channel idx                                               
      |                                                                                    
      |--> install ops, e.g.,                                                              
      |    +-----------------------+                                                       
      |    | __i2c_mux_master_xfer | switch mux, transfer i2c packet, switch mux to default
      |    +-----------------------+                                                       
      |                                                                                    
      |--> set up priv->adap                                                               
      |                                                                                    
      |    +-----------------+                                                             
      +--> | i2c_add_adapter | determine adapter id and register it                        
           +-----------------+                                                             
```

```
drivers/i2c/busses/i2c-ast2600.c                                                                                           
+-------------------+                                                                                                       
| ast2600_i2c_probe | prepare i2c-bus, determine mode, ioremap registers, init hw, register irq, add adapter                
+-|-----------------+                                                                                                       
  |                                                                                                                         
  |--> alloc i2c-bus, look for global register map, and read its control register                                           
  |                                                                                                                         
  |--> set bus mode = dma                                                                                                   
  |                                                                                                                         
  |--> if 'byte-mode' is specified in dt                                                                                    
  |    -                                                                                                                    
  |    +--> mode = byte                                                                                                     
  |                                                                                                                         
  |--> if 'buff-mode' is specified in dt                                                                                    
  |    |                                                                 i2c0: i2c-bus@80 {                                 
  |    |    +-----------------------+                                        #address-cells = <1>;                          
  |    |--> | platform_get_resource | get mem resource 1                     #size-cells = <0>;                             
  |    |    +-----------------------+                                        #interrupt-cells = <1>;                        
  |    |    +-----------------------+                                        reg = <0x80 0x80>, <0xC00 0x20>;               
  |    |--> | devm_ioremap_resource | ioremap                                compatible = "aspeed,ast2600-i2c-bus";         
  |    |    +-----------------------+                                        clocks = <&syscon ASPEED_CLK_APB2>;            
  |    |                                                                     resets = <&syscon ASPEED_RESET_I2C>;           
  |    +--> mode = buff                                                      interrupts = <GIC_SPI 110 IRQ_TYPE_LEVEL_HIGH>;
  |                                                                          bus-frequency = <100000>;                      
  |--> if not byte mode, smbus algo = ast2600_i2c_smbus_xfer                 buff-mode;                                     
  |                                                                          pinctrl-names = "default";                     
  |    +--------------------------------+                                    pinctrl-0 = <&pinctrl_i2c1_default>;           
  |--> | devm_platform_ioremap_resource | ioremap mem resource 0             status = "disabled";                           
  |    +--------------------------------+                                };                                                 
  |    +----------------------+                                                                                             
  |--> | irq_of_parse_and_map | get irq value                                                                               
  |    +----------------------+                                                                                             
  |                                                                                                                         
  |--> get property 'bus-frequency'                                                                                         
  |                                                                                                                         
  |    +------------------+                                                                                                 
  |--> | ast2600_i2c_init | init i2c hw                                                                                     
  |    +------------------+                                                                                                 
  |    +------------------+                                                                                                 
  |--> | devm_request_irq | prepare 'action' (handler, thread_fn, ...) and install to irq desc                              
  |    +------------------+ +---------------------+                                                                         
  |                         | ast2600_i2c_bus_irq |                                                                         
  |                         +---------------------+                                                                         
  |                                                                                                                         
  |--> set master interrupt generation                                                                                      
  |                                                                                                                         
  |    +-----------------+                                                                                                  
  |--> | i2c_add_adapter | determine adapter id and register it                                                             
  |    +-----------------+                                                                                                  
  |                                                                                                                         
  +--> print "%s [%d]: adapter [%d khz] mode [%d]\n"                                                                        
```

```
drivers/i2c/busses/i2c-ast2600.c    
+------------------+                 
| ast2600_i2c_init | : init i2c hw   
+-|----------------+                 
  |                                  
  |--> reset i2c                     
  |                                  
  |--> enable master mode            
  |                                  
  |--> disable slave address         
  |                                  
  |--> set ac timing (?)             
  |                                  
  |--> clear master interrupt        
  |                                  
  |--> if bus mode == dma (default)  
  |    |                             
  |    |    +---------------------+  
  |    +--> | dmam_alloc_coherent |  
  |         +---------------------+  
  |                                  
  |--> clear slave interrupt         
  |                                  
  +--> set slave interrupt generation
```

```
drivers/i2c/busses/i2c-ast2600.c                                                                    
+---------------------+                                                                              
| ast2600_i2c_bus_irq | : handle interrupt (from slave perspective first, then master)               
+-|-------------------+                                                                              
  |                                                                                                  
  |--> if slave is enabled                                                                           
  |    |                                                                                             
  |    |    +-----------------------+                                                                
  |    |--> | ast2600_i2c_slave_irq | given isr, handle read or write request from master accordingly
  |    |    +-----------------------+                                                                
  |    |                                                                                             
  |    +--> if something is done, return handled                                                     
  |                                                                                                  
  |    +------------------------+                                                                    
  +--> | ast2600_i2c_master_irq | given isr, handle tx/rx accordingly                                
       +------------------------+                                                                    
```

```
drivers/i2c/busses/i2c-ast2600.c                                                                                     
+-----------------------+                                                                                             
| ast2600_i2c_slave_irq | : given isr, handle read or write request from master accordingly                           
+-|---------------------+                                                                                             
  |                                                                                                                   
  |--> read slave ier and isr                                                                                         
  |                                                                                                                   
  |--> if nothing, return                                                                                             
  |                                                                                                                   
  |--> if bit 'packet done' is set in isr                                                                             
  |    |                                                                                                              
  |    |--> if bus mode == dma                                                                                        
  |    |    |                                                                                                         
  |    |    |    +----------------------------------+                                                                 
  |    |    +--> | ast2600_i2c_slave_packet_dma_irq | given isr, handle read or write request from master accordingly 
  |    |         +----------------------------------+                                                                 
  |    |                                                                                                              
  |    +--> else （mode == buff)                                                                                       
  |         |                                                                                                         
  |         |    +-----------------------------------+                                                                
  |         +--> | ast2600_i2c_slave_packet_buff_irq | given isr, handle read or write request from master accordingly
  |              +-----------------------------------+                                                                
  |                                                                                                                   
  +--> else (mode == byte)                                                                                            
       |                                                                                                              
       |    +----------------------------+                                                                            
       +--> | ast2600_i2c_slave_byte_irq | given isr, handle read or write request from master accordingly            
            +----------------------------+                                                                            
```

```
drivers/i2c/busses/i2c-ast2600.c                                                                     
+----------------------------------+                                                                  
| ast2600_i2c_slave_packet_dma_irq | : given isr, handle read or write request from master accordingly
+-|--------------------------------+                                                                  
  |                                                                                                   
  |--> switch isr                                                                                     
  |    case blabla                                                                                    
  |    |    +-----------------+                                                                       
  |    |--> | i2c_slave_event | event = master-requests-to-write                                      
  |    |    +-----------------+                                                                       
  |    +--> for each byte in rx len                                                                   
  |         -    +-----------------+                                                                  
  |         +--> | i2c_slave_event | event = master-writes-a-byte                                     
  |              +-----------------+                                                                  
  |    case blabla                                                                                    
  |    -     +-----------------+                                                                      
  |    +---> | i2c_slave_event | event = master-finishes-the-write-and-wait-for-response              
  |          +-----------------+                                                                      
  |    case blabla                                                                                    
  |    |    +-----------------+                                                                       
  |    |--> | i2c_slave_event | event = master-requests-to-write                                      
  |    |    +-----------------+                                                                       
  |    |--> for each byte in rx len                                                                   
  |    |    -    +-----------------+                                                                  
  |    |    +--> | i2c_slave_event | event = master-writes-a-byte                                     
  |    |         +-----------------+                                                                  
  |    |    +-----------------+                                                                       
  |    +--> | i2c_slave_event | event = master-finishes-the-write-and-wait-for-response               
  |         +-----------------+                                                                       
  |    case blabla                                                                                    
  |    |    +-----------------+                                                                       
  |    |--> | i2c_slave_event | event = master-requests-to-write                                      
  |    |    +-----------------+                                                                       
  |    |--> for each byte in rx len                                                                   
  |    |    -    +-----------------+                                                                  
  |    |    +--> | i2c_slave_event | event = master-writes-a-byte                                     
  |    |         +-----------------+                                                                  
  |    |    +-----------------+                                                                       
  |    +--> | i2c_slave_event | event = master-requests-to-read                                       
  |         +-----------------+                                                                       
  |    case blabla                                                                                    
  |    -    +-----------------+                                                                       
  |    +--> | i2c_slave_event | event = master-requests-to-read                                       
  |         +-----------------+                                                                       
  |    case blabla                                                                                    
  |    -    +-----------------+                                                                       
  |    +--> | i2c_slave_event | event = master-reads-a-byte                                           
  |         +-----------------+                                                                       
  |                                                                                                   
  +--> write cmd, set packet_done, clear isr                                                          
```

```
drivers/i2c/busses/i2c-ast2600.c                                                      
+------------------------+                                                             
| ast2600_i2c_master_irq | : given isr, handle tx/rx accordingly                       
+-|----------------------+                                                             
  |                                                                                    
  |--> read isr and ier                                                                
  |                                                                                    
  +--> if bit 'packet done' is set in isr                                              
       |                                                                               
       |--> if protocol == smbus                                                       
       |    |                                                                          
       |    |    +-------------------------------+                                     
       |    +--> | ast2600_i2c_smbus_package_irq | given isr, handle tx/rx accordingly 
       |         +-------------------------------+                                     
       |                                                                               
       +--> else (protocl == i2c)                                                      
            |                                                                          
            |    +--------------------------------+                                    
            +--> | ast2600_i2c_master_package_irq | given isr, handle tx/rx accordingly
                 +--------------------------------+                                    
```
  
</details>

## <a name="cheat-sheet"></a> Cheat Sheet

- Detect I2C devices on a bus.

```
E.g.,
  
i2cdetect -y -a 11
  
# -y: yes, to avoid the interactive confirmation
# -a: all devices or the tool skip the first and last few addresses
# 11: bus 11
```
  
- List all I2C adapters (controllers + mux channels).
  
```
i2cdetect -l
```
  
- Transfer data with the target device on a bus.
 
```
E.g.,
  
i2ctransfer -y -a 11 w1@0x71 0x40 r1
  
# -y: yes, to avoid the interactive confirmation
# -a: all devices or the tool skip the first and last few addresses
# 11: bus 11
# w1@0x71: write 1 byte to slave 0x71
# 0x40: the data to write
# r1: read 1 byte from the previously specified slave address
# i2ctransfer uses ICC
```
  
```
Equivalent:
  
ipmitool i2c bus=11 0xe2 0x01 0x40
  
# 0xe2 is the 8-bit address representation of 0x71
# [note] openbmc accepts bus numbers 0 to 7 only due to the uint3_t type in ipmiMasterWriteRead()
# [note] tools use different i2c protocols
#     i2c: i2ctransfer, ipmitool
#     smbus: i2cget, i2cset
```
  
- Dump transaction data from kernel space.
  
```
E.g.,
  
# enable i2c tracing
echo 1 > /sys/kernel/debug/tracing/events/i2c/enable
  
# add a filter, or it traces on all buses by default
echo adapter_nr==11 >/sys/kernel/debug/tracing/events/i2c/filter
  
# read trace
cat /sys/kernel/debug/tracing/trace
```

- Enable dynamic debug.

```
echo "file i2c-ast2600.c +p" >> /sys/kernel/debug/dynamic_debug/control
```
  
## <a name="reference"></a> Reference

- [S. Crump, SMBus Compatibility With an I2C Device](https://www.ti.com/lit/an/sloa132/sloa132.pdf)
- [Ftrace:snoop i2c bus transactions](https://technolinchpin.wordpress.com/2017/08/07/ftracesnoop-i2c-bus-transactions/)
