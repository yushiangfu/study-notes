## Index

- [Introduction](#introduction)
- [Driver](#driver)
- [System Startup](#system-startup)
- [Tools](#tools)
- [Cheat Sheet](#cheat-sheet)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

(TBD)

## <a name="driver"></a> Driver

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

## <a name="system-startup"></a> System Startup

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
      |--> | devm_request_irq | register isr 'aspeed_i2c_bus_irq'                           
      |    +------------------+ (ack, based on master state: write or read, if rx done: ack)
      |    +-----------------+                                                              
      |--> | i2c_add_adapter | determine adapter id and register it                         
      |    +-----------------+                                                              
      |                                                                                     
      +--> print "i2c bus %d registered, irq %d\n"                                          
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
     |    +-----------------------+                                                                                    
     +--> | aspeed_i2c_master_irq | based on master state, write to or read from hw reg for a byte, notify waiting task
     |    +-----------------------+                                                                                    
     |                                                                                                                 
     |--> if bits has 'rx done'                                                                                        
     |                                                                                                                 
     +------> write hw register to ack                                                                                 
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

## <a name="tools"></a> Tools

## <a name="cheat-sheet"></a> Cheat Sheet

## <a name="reference"></a> Reference
