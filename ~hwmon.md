```
drivers/hwmon/aspeed-g6-pwm-tach.c                                                                                 
+-----------------------+                                                                                           
| aspeed_pwm_tach_probe | : ioremap register base, install chip ops, setup hwmon_dev and register it
+-|---------------------+                                                                                           
  |                                                                                                                 
  |--> alloc priv                                                                                                   
  |                                                                                                                 
  |    +--------------------------------+                                                                           
  |--> | devm_platform_ioremap_resource | ioremap register base                                                     
  |    +--------------------------------+                                                                           
  |                                                                                                                 
  |--> set chip ops = aspeed_pwm_ops                                                                                
  |                                                                                                                 
  |    +------------------+                                                                                         
  |--> | devm_pwmchip_add | alloc pwm devs for chip, add chip to 'pwms_chips' list                                  
  |    +------------------+                                                                                         
  |                                                                                                                 
  |--> for each chil  node (fan-0 ~ fan-15)                                                                         
  |    |                                                                                                            
  |    |    +------------------------+                                                                              
  |    +--> | aspeed_tach_create_fan | read property 'tach-ch' to know channel, config registers and enable hardware
  |         +------------------------+                                                                              
  |    +--------------------------------------+                                                                     
  +--> | devm_hwmon_device_register_with_info | setup hwmon_dev, register it, add to arg dev as resource            
       +--------------------------------------+                                                                     
```

```
drivers/hwmon/aspeed-g6-pwm-tach.c                                                                       
+------------------------+                                                                                
| aspeed_tach_create_fan | : read property 'tach-ch' to know channel, config registers and enable hardware
+-|----------------------+                                                                                
  |    +----------------------------+                                                                     
  |--> | of_property_count_u8_elems | count how many elements are in property "tach-ch"                   
  |    +----------------------------+                                                                     
  |                                                                                                       
  |--> alloc space for them                                                                               
  |                                                                                                       
  |    +---------------------------+                                                                      
  |--> | of_property_read_u8_array | read property 'tach-ch' into allocated space                         
  |    +---------------------------+                                                                      
  |    +-------------------------+                                                                        
  +--> | aspeed_present_fan_tach | config registers and enable hardware                                   
       +-------------------------+                                                                        
```

```
drivers/hwmon/aspeed-chassis.c                                                                           
+----------------------+                                                                                  
| aspeed_chassis_probe | : ioremap register base, disable hw interrupts, setup hwmon_dev and register it  
+-|--------------------+                                                                                  
  |                                                                                                       
  |--> alloc priv                                                                                         
  |                                                                                                       
  |    +--------------------------------+                                                                 
  |--> | devm_platform_ioremap_resource | ioremap register base                                           
  |    +--------------------------------+                                                                 
  |    +-------------------------+                                                                        
  |--> | aspeed_chassis_int_ctrl | write reg to disalbe hw interrupt                                      
  |    +-------------------------+                                                                        
  |    +----------------------------------------+                                                         
  +--> | devm_hwmon_device_register_with_groups | setup hwmon_dev, register it, add to arg dev as resource
       +----------------------------------------+                                                         
```

```
drivers/hwmon/lm75.c                                                                                   
+------------+                                                                                          
| lm75_probe | : get 'vs' regulator and enable, write config byte to lm75                               
+-|----------+                                                                                          
  |                                                                                                     
  |--> get type                                                                                         
  |                                                                                                     
  +--> alloc data for lm75                                                                              
  |                                                                                                     
  |    +--------------------+                                                                           
  |--> | devm_regulator_get | get regulator with id 'vs'                                                
  |    +--------------------+                                                                           
  |    +----------------------+                                                                         
  |--> | devm_regmap_init_i2c | (skip)                                                                  
  |    +----------------------+                                                                         
  |                                                                                                     
  |--> ->params = device_params[lm75]                                                                   
  |                                                                                                     
  |    +------------------+                                                                             
  |--> | regulator_enable | (skip)                                                                      
  |    +------------------+                                                                             
  |    +--------------------------+                                                                     
  |--> | i2c_smbus_read_byte_data | read config byte from lm75                                          
  |    +--------------------------+                                                                     
  |    +-------------------+                                                                            
  |--> | lm75_write_config | write adjusted config byte back to lm75                                    
  |    +-------------------+                                                                            
  |    +--------------------------------------+                                                         
  |--> | devm_hwmon_device_register_with_info | setup hwmon_dev, register it, add to arg dev as resource
  |    +--------------------------------------+                                                         
  |                                                                                                     
  +--> print lm75 8-004d: hwmon2: sensor 'lm75'                                                         
```

```
drivers/pwm/core.c                                                     
+-------------+                                                         
| pwmchip_add | : alloc pwm devs for chip, add chip to 'pwms_chips' list
+-|-----------+                                                         
  |    +------------+                                                   
  |--> | alloc_pwms | alloc available ids for pwm                       
  |    +------------+                                                   
  |                                                                     
  |--> alloc that many pwm devs                                         
  |                                                                     
  |--> init each pwm dev                                                
  |                                                                     
  |--> add arg chip to 'pwm_chips' list                                 
  |                                                                     
  |    +----------------------+                                         
  +--> | pwmchip_sysfs_export | create /sys/class/pwm/pwmchip0          
       +----------------------+                                         
```

```
drivers/hwmon/pwm-fan.c                                                                                          
+---------------+                                                                                                 
| pwm_fan_probe | : get pwm-dev from chip, apply state to hw, add hwmon-dev and colling-dev to arg dev as resource
+-|-------------+                                                                                                 
  |                                                                                                               
  |--> alloc ctx                                                                                                  
  |                                                                                                               
  |    +-----------------+                                                                                        
  |--> | devm_of_pwm_get | get pwm dev from chip                                                                  
  |    +-----------------+                                                                                        
  |    +-----------------------------+                                                                            
  |--> | devm_regulator_get_optional | (skip)                                                                     
  |    +-----------------------------+                                                                            
  |    +----------------+                                                                                         
  |--> | pwm_init_state | get init state                                                                          
  |    +----------------+                                                                                         
  |    +-----------+                                                                                              
  |--> | __set_pwm | given pwm (what unit?), apply to hardware                                                    
  |    +-----------+                                                                                              
  |    +-------------+                                                                                            
  |--> | timer_setup | setup timer for sampling                                                                   
  |    +-------------+                                                                                            
  |    +--------------------+                                                                                     
  |--> | platform_irq_count | get irq count of tach                                                               
  |    +--------------------+                                                                                     
  |                                                                                                               
  |--> if any (probably not our case), alloc that many pwm_fan_tach                                               
  |                                                                                                               
  |--> alloc channels for pwm, and what?                                                                          
  |                                                                                                               
  |--> for each tack irq (probably not our case)                                                                  
  |    |                                                                                                          
  |    |    +------------------+                                                                                  
  |    +--> | devm_request_irq |                                                                                  
  |         +------------------+                                                                                  
  |                                +--------------+                                                               
  |--> install 'pwm_fan_hwmon_ops' | pwm_fan_read |                                                               
  |                                +---------------+                                                              
  |                                | pwm_fan_write |                                                              
  |                                +---------------+                                                              
  |    +--------------------------------------+                                                                   
  |--> | devm_hwmon_device_register_with_info | setup hwmon_dev, register it, add to arg dev as resource          
  |    +--------------------------------------+                                                                   
  |    +-----------------------------+                                                                            
  |--> | pwm_fan_of_get_cooling_data | (cooling related, skip)                                                    
  |    +-----------------------------+                                                                            
  |    +-----------------------------------------+                                                                
  +--> | devm_thermal_of_cooling_device_register | setup cooling-dev, add to arg dev as resource                  
       +-----------------------------------------+                                                                
```

```
drivers/hwmon/pwm-fan.c                                    
+-----------+                                               
| __set_pwm | : given pwm (what unit?), apply to hardware   
+-|---------+                                               
  |                                                         
  |--> given arg pwm, calculate updated duty cycle          
  |                                                         
  |    +-----------------+                                  
  +--> | pwm_apply_state | given state, apply it to hardware
       +-----------------+                                  
```

```
drivers/pwm/core.c                                                   
+-----------------+                                                   
| pwm_apply_state | : given state, apply it to hardware               
+-|---------------+                                                   
  |                                                                   
  |--> if state attrs == pwm state attrs, return                      
  |                                                                   
  |--> if ->apply() exists                                            
  |    |                                                              
  |    |--> call it, e.g.,                                            
  |    |    +------------------+                                      
  |    |    | aspeed_pwm_apply | calculate duty cycle and write to reg
  |    |    +------------------+                                      
  |    |                                                              
  |    +--> save state in pwm                                         
  |                                                                   
  +--> else                                                           
       -                                                              
       +--> (not our case, skip)                                      
```

```
drivers/hwmon/hwmon.c                                                                                         
+--------------------------------------+                                                                       
| devm_hwmon_device_register_with_info | : setup hwmon_dev, register it, add to arg dev as resource            
+-|------------------------------------+                                                                       
  |    +--------------+                                                                                        
  |--> | devres_alloc | alloc 'device'                                                                         
  |    +--------------+                                                                                        
  |    +---------------------------------+                                                                     
  |--> | hwmon_device_register_with_info | setup hwmon (groups, attrs, ...), register hwmon dev (class = hwmon)
  |    +---------------------------------+                                                                     
  |    +------------+                                                                                          
  +--> | devres_add | add hwmon_dev to arg dev as resource                                                     
       +------------+                                                                                          
```

```
drivers/hwmon/hwmon.c                                                                                    
+---------------------------------+                                                                       
| hwmon_device_register_with_info | : setup hwmon (groups, attrs, ...), register hwmon dev (class = hwmon)
+-------------------------+-------+                                                                       
| __hwmon_device_register | : setup hwmon (groups, attrs, ...), register hwmon dev (class = hwmon)        
+-|-----------------------+                                                                               
  |    +----------------+                                                                                 
  |--> | ida_simple_get | get an available id                                                             
  |    +----------------+                                                                                 
  |                                                                                                       
  |--> alloc hwmon_dev                                                                                    
  |                                                                                                       
  |--> if arg chip is provided                                                                            
  |    |                                                                                                  
  |    |--> alloc groups for hwmon_dev                                                                    
  |    |                                                                                                  
  |    |    +----------------------+                                                                      
  |    |--> | __hwmon_create_attrs |                                                                      
  |    |    +----------------------+                                                                      
  |    |                                                                                                  
  |    +--> save arg groups in hwdev                                                                      
  |                                                                                                       
  |    +-----------------+                                                                                
  |--> | device_register | register device (class = hwmon)                                                
  |    +-----------------+                                                                                
  |                                                                                                       
  +--> if ->read() exists                                                                                 
       |                                                                                                  
       |    +--------------------------------+                                                            
       +--> | hwmon_thermal_register_sensors | (fan has no thermal, skip)                                 
            +--------------------------------+                                                            
```

```
drivers/thermal/thermal_core.c                                                                            
+-----------------------------------------+                                                                
| devm_thermal_of_cooling_device_register | : setup cooling-dev, add to arg dev as resource                
+-|---------------------------------------+                                                                
  |    +--------------+                                                                                    
  |--> | devres_alloc |                                                                                    
  |    +--------------+                                                                                    
  |    +-----------------------------------+                                                               
  |--> | __thermal_cooling_device_register | setup cooling-dev, register dev and add to 'thermal_cdev_list'
  |    +-----------------------------------+                                                               
  |    +------------+                                                                                      
  +--> | devres_add |                                                                                      
       +------------+                                                                                      
```

```
drivers/thermal/thermal_core.c                                                                              
+-----------------------------------+                                                                        
| __thermal_cooling_device_register | : setup cooling-dev, register dev and add to 'thermal_cdev_list'       
+-|---------------------------------+                                                                        
  |                                                                                                          
  |--> alloc cdev                                                                                            
  |                                                                                                          
  |    +----------------+                                                                                    
  |--> | ida_simple_get | get an available id                                                                
  |    +----------------+                                                                                    
  |    +--------------+                                                                                      
  |--> | dev_set_name | "cooling_device%d"                                                                   
  |    +--------------+                                                                                      
  |                                                                                                          
  |--> set cdev type, e.g., pwm-fan                                                                          
  |                                                                                                          
  |--> install cdev ops, e.g., pwm_fan_cooling_ops                                                           
  |                                                                                                          
  |--> set cdev class = thermal_class                                                                        
  |                                                                                                          
  |    +------------------------------------+                                                                
  |--> | thermal_cooling_device_setup_sysfs | (basically do nothing bc of disabled CONFIG_THERMAL_STATISTICS)
  |    +------------------------------------+                                                                
  |    +-----------------+                                                                                   
  |--> | device_register | register cdev->device                                                             
  |    +-----------------+                                                                                   
  |                                                                                                          
  |--> add cdev to 'thermal_cdev_list'                                                                       
  |                                                                                                          
  |    +-----------+                                                                                         
  |--> | bind_cdev | traverse thermal zone, let each entry bind this cdev                                    
  |    +-----------+                                                                                         
  |                                                                                                          
  +--> for each need-update entry in thermal zone list                                                       
       |                                                                                                     
       |    +----------------------------+                                                                   
       +--> | thermal_zone_device_update | (thermal zone related, skip)                                      
            +----------------------------+                                                                   
```

```
drivers/hwmon/iio_hwmon.c                                                                                
+-----------------+                                                                                       
| iio_hwmon_probe | : alloc and setup channels, prepare hwmon_dev and register it                         
+-|---------------+                                                                                       
  |    +--------------------------+                                                                       
  |--> | devm_iio_channel_get_all | alloc channels and save iio_dev/map info in each channel              
  |    +--------------------------+                                                                       
  |                                                                                                       
  |--> alloc state                                                                                        
  |                                                                                                       
  |--> save channels in state                                                                             
  |                                                                                                       
  |--> alloc attrs and save in state                                                                      
  |                                                                                                       
  |--> for each channel                                                                                   
  |    |                                                                                                  
  |    |--> alloc sensor_attr                                                                             
  |    |                                                                                                  
  |    |    +----------------------+                                                                      
  |    |--> | iio_get_channel_type | get channel type, e.g., voltage, temp, current, ...                  
  |    |    +----------------------+                                                                      
  |    |                                                                                                  
  |    |--> given type, determine prefix and update counting                                              
  |    |                                                                                                  
  |    |--> determine attr name, e.g., in0_input                                                          
  |    |                                                                                                  
  |    +--> install show()                                                                                
  |         +--------------------+                                                                        
  |         | iio_hwmon_read_val | read sensor value                                                      
  |         +--------------------+                                                                        
  |                                                                                                       
  |--> determine name for dev, either "%pfwP" or "iio_hwmon"                                              
  |                                                                                                       
  |    +----------------------------------------+                                                         
  +--> | devm_hwmon_device_register_with_groups | setup hwmon_dev, register it, add to arg dev as resource
       +----------------------------------------+                                                         
```

```
drivers/iio/inkern.c                                                                  
+--------------------------+                                                           
| devm_iio_channel_get_all | : alloc channels and save iio_dev/map info in each channel
+-|------------------------+                                                           
  |    +---------------------+                                                         
  |--> | iio_channel_get_all | alloc channels and save iio_dev/map info in each channel
  |    +---------------------+                                                         
  |    +--------------------------+                                                    
  +--> | devm_add_action_or_reset | register callback for device teardown              
       +--------------------------+ +---------------------------+                      
                                    | devm_iio_channel_free_all |                      
                                    +---------------------------+                      
```

```
drivers/iio/inkern.c                                                                                            
+---------------------+                                                                                          
| iio_channel_get_all | : alloc channels and save iio_dev/map info in each channel                               
+-|-------------------+                                                                                          
  |    +----------------------------+                                                                            
  |--> | fwnode_iio_channel_get_all | count map#, alloc that many channels, find iio_dev and save in each channel
  |    +----------------------------+                                                                            
  |                                                                                                              
  |--> traverse iio_map_list to count map# with name == arg name                                                 
  |                                                                                                              
  +--> for each map with name == arg name                                                                        
       |                                                                                                         
       |--> channel[i] = map info                                                                                
       |                                                                                                         
       +--> i++                                                                                                  
```

```
drivers/iio/inkern.c                                                                                       
+----------------------------+                                                                              
| fwnode_iio_channel_get_all | : count map#, alloc that many channels, find iio_dev and save in each channel
+-|--------------------------+                                                                              
  |                                                                                                         
  |--> count the number of maps                                                                             
  |                                                                                                         
  |--> alloc that many channels                                                                             
  |                                                                                                         
  +--> for each map                                                                                         
       |                                                                                                    
       |    +--------------------------+                                                                    
       +--> | __fwnode_iio_channel_get | find target iio_dev from iio_bus_type, save it in channel          
            +--------------------------+                                                                    
```

```
drivers/iio/inkern.c                                                                   
+--------------------------+                                                            
| __fwnode_iio_channel_get | : find target iio_dev from iio_bus_type, save it in channel
+-|------------------------+                                                            
  |    +------------------------------------+                                           
  |--> | fwnode_property_get_reference_args | given args, get target fw-node            
  |    +------------------------------------+                                           
  |    +---------------------------+                                                    
  |--> | bus_find_device_by_fwnode | find target dev on iio_bus_type                    
  |    +---------------------------+                                                    
  |    +----------------+                                                               
  |--> | dev_to_iio_dev | given dev, get outer iio-dev                                  
  |    +----------------+                                                               
  |                                                                                     
  |--> save iio_dev in channel                                                          
  |                                                                                     
  +--> save iio_dev->channel in channel                                                 
```

```
drivers/hwmon/iio_hwmon.c                      
+--------------------+                          
| iio_hwmon_read_val | : read sensor value      
+-|------------------+                          
  |    +----------------------------+           
  |--> | iio_read_channel_processed | read value
  |    +----------------------------+           
  |                                             
  +--> if type is power, do extra adjustment    
```

```
drivers/iio/inkern.c                                                                               
+----------------------------+                                                                      
| iio_read_channel_processed | : read value                                                         
+----------------------------------+                                                                
| iio_read_channel_processed_scale | : read value                                                   
+-|--------------------------------+                                                                
  |                                                                                                 
  |--> if channel info is processed                                                                 
  |    |                                                                                            
  |    |    +------------------+                                                                    
  |    +--> | iio_channel_read | call installed ops to read value                                   
  |         +------------------+ (flag = processed)                                                 
  |                                                                                                 
  +--> else                                                                                         
       |                                                                                            
       |    +------------------+                                                                    
       |--> | iio_channel_read | call installed ops to read value                                   
       |    +------------------+ (flag = raw)                                                       
       |    +---------------------------------------+                                               
       +--> | iio_convert_raw_to_processed_unlocked | read offset and scale to convert the raw value
            +---------------------------------------+                                               
```

```
drivers/iio/inkern.c                                  
+------------------+                                   
| iio_channel_read | : call installed ops to read value
+-|----------------+                                   
  |                                                    
  |--> if ->read_raw_multi() exists                    
  |    -                                               
  |    +--> call ->read_raw_multi()                    
  |                                                    
  +--> else                                            
       -                                               
       +--> call ->read_raw(), e.g.,                   
            +---------------------+                    
            | aspeed_adc_read_raw |                    
            +---------------------+                    
```

```
drivers/iio/adc/aspeed_adc.c                                                                                            
+------------------+                                                                                                     
| aspeed_adc_probe | : setup iio-dev, start channels, install ops, publish to debugfs/sysfs, register cdev               
+-|----------------+                                                                                                     
  |    +-----------------------+                                                                                         
  |--> | devm_iio_device_alloc | alloc iio_dev and setup (name, ...)                                                     
  |    +-----------------------+                                                                                         
  |    +--------------------------------+                                                                                
  |--> | devm_platform_ioremap_resource | ioremap register base                                                          
  |    +--------------------------------+                                                                                
  |                                                                                                                      
  |--> if prescaler is needed (probably not ast2600's case, skip)                                                        
  |                                                                                                                      
  |    +------------------------+                                                                                        
  |--> | aspeed_adc_vref_config | get 'vref' regulator and save in iio_dev priv                                          
  |    +------------------------+                                                                                        
  |    +--------------------------+                                                                                      
  |--> | aspeed_adc_set_trim_data | write register to set trim data                                                      
  |    +--------------------------+                                                                                      
  |                                                                                                                      
  |--> if property "aspeed,battery-sensing" exists (not our case, skip)                                                  
  |                                                                                                                      
  |    +------------------------------+                                                                                  
  |--> | aspeed_adc_set_sampling_rate |                                                                                  
  |    +------------------------------+                                                                                  
  |                                                                                                                      
  |--> wait for init sequence to complete                            static const struct iio_info aspeed_adc_iio_info = {
  |                                                                      .read_raw = aspeed_adc_read_raw,                
  |--> write registers to start all channels in normal mode              .write_raw = aspeed_adc_write_raw,              
  |                                                                      .debugfs_reg_access = aspeed_adc_reg_access,    
  |--> install ops 'aspeed_adc_iio_info'  -------------------------  };                                                  
  |                                                                                                                      
  |    +--------------------------+                                                                                      
  +--> | devm_iio_device_register | publish to debugfs/sysfs, init cdev and register it                                  
       +--------------------------+                                                                                      
```

```
drivers/iio/adc/aspeed_adc.c                                  
+-----------------------+                                      
| devm_iio_device_alloc | : alloc iio_dev and setup (name, ...)
+------------------+----+                                      
| iio_device_alloc |                                           
+-|----------------+                                           
  |                                                            
  |--> alloc iio_dev + extra buffer (opaque?)                  
  |                                                            
  |    +-----------+                                           
  |--> | ida_alloc | get an available id                       
  |    +-----------+                                           
  |    +--------------+                                        
  +--> | dev_set_name | "iio:device%d"                         
       +--------------+                                        
```

```
include/linux/iio/iio.h                                                                                 
+--------------------------+                                                                             
| devm_iio_device_register | : publish to debugfs/sysfs, init cdev and register it                       
+----------------------------+                                                                           
| __devm_iio_device_register | : publish to debugfs/sysfs, init cdev and register it                     
+-----------------------+----+                                                                           
| __iio_device_register | : publish to debugfs/sysfs, init cdev and register it                          
+-|---------------------+                                                                                
  |    +-----------------------------+                                                                   
  |--> | iio_device_register_debugfs | create file in debugfs                                            
  |    +-----------------------------+ e.g.,                                                             
  |                                    /sys/kernel/debug/iio/iio:device0/direct_reg_access               
  |                                    /sys/kernel/debug/iio/iio:device1/direct_reg_access               
  |                                                                                                      
  |    +----------------------------------+                                                              
  |--> | iio_buffers_alloc_sysfs_and_mask | alloc attr & publish to sysfs, register handler to iio-dev   
  |    +----------------------------------+                                                              
  |    +---------------------------+                                                                     
  |--> | iio_device_register_sysfs | add each channel to sysfs, register attr group to sysfs             
  |    +---------------------------+                                                                     
  |    +------------------------------+                                                                  
  |--> | iio_device_register_eventset | alloc event iface & publish to sysfs, register handler to iio-dev
  |    +------------------------------+                                                                  
  |                                                                                                      
  |--> if iio-dev has attached buffer                                                                    
  |    |                                                                                                 
  |    |    +-----------+                                                                                
  |    +--> | cdev_init | install 'iio_buffer_fileops'                                                   
  |         +-----------+                                                                                
  |                                                                                                      
  |--> elif iio-dev has event iface                                                                      
  |    |                                                                                                 
  |    |    +-----------+                                                                                
  |    +--> | cdev_init | install 'iio_event_fileops'                                                    
  |         +-----------+                                                                                
  |    +-----------------+                                                                               
  +--> | cdev_device_add | add cdev to kobj_map and register inner dev                                   
       +-----------------+                                                                               
```

```
drivers/iio/industrialio-buffer.c                                                                    
+----------------------------------+                                                                  
| iio_buffers_alloc_sysfs_and_mask | : alloc attr & publish to sysfs, register handler to iio-dev     
+-|--------------------------------+                                                                  
  |                                                                                                   
  |--> traverse channels to determine mask length                                                     
  |                                                                                                   
  |--> for each attached buffer of iio-dev                                                            
  |    |                                                                                              
  |    |    +-----------------------------------+                                                     
  |    +--> | __iio_buffer_alloc_sysfs_and_mask | alloc attr and register to sysfs                    
  |         +-----------------------------------+                                                     
  |                                                                                                   
  |--> install handler                                                                                
  |    +-------------------------+                                                                    
  |    | iio_device_buffer_ioctl | given idx, get target attached buffer, prepare anon file, return fd
  |    +-------------------------+                                                                    
  |                                                                                                   
  |    +-----------------------------------+                                                          
  +--> | iio_device_ioctl_handler_register | register handler to iio-dev                              
       +-----------------------------------+                                                          
```

```
drivers/iio/industrialio-buffer.c                                                                       
+-----------------------------------+                                                                    
| __iio_buffer_alloc_sysfs_and_mask | : alloc attr and register to sysfs                                 
+-|---------------------------------+                                                                    
  |                                                                                                      
  |--> if channel exists                                                                                 
  |    -                                                                                                 
  |    +--> for each channel                                                                             
  |         |                                                                                            
  |         |    +------------------------------+                                                        
  |         +--> | iio_buffer_add_channel_sysfs | alloc channel dev attributes: 'index', 'type', and 'en'
  |              +------------------------------+                                                        
  |                                                                                                      
  |--> alloc that many attr                                                                              
  |                                                                                                      
  |--> save attr in buffer                                                                               
  |                                                                                                      
  |--> for each attr                                                                                     
  |    |                                                                                                 
  |    |    +----------------------+                                                                     
  |    +--> | iio_buffer_wrap_attr | ???                                                                 
  |         +----------------------+                                                                     
  |                                                                                                      
  |--> set buffer name = "buffer%d"                                                                      
  |                                                                                                      
  |    +---------------------------------+                                                               
  |--> | iio_device_register_sysfs_group | register attr group to iio-dev                                
  |    +---------------------------------+                                                               
  |                                                                                                      
  |--> if index > 0, return (index 0 is for legacy group)                                                
  |                                                                                                      
  |    +-----------------------------------------+                                                       
  +--> | iio_buffer_register_legacy_sysfs_groups | allo two attr, save in legacy group, register to sysfs
       +-----------------------------------------+                                                       
```

```
drivers/iio/industrialio-buffer.c                                                        
+------------------------------+                                                          
| iio_buffer_add_channel_sysfs | : alloc channel dev attributes: 'index', 'type', and 'en'
+-|----------------------------+                                                          
  |    +------------------------+                                                         
  |--> | __iio_add_chan_devattr | alloc channel dev attr "index"                          
  |    +------------------------+                                                         
  |    +------------------------+                                                         
  |--> | __iio_add_chan_devattr | alloc channel dev attr "type"                           
  |    +------------------------+                                                         
  |    +------------------------+                                                         
  +--> | __iio_add_chan_devattr | alloc channel dev attr "en"                             
       +------------------------+                                                         
```

```
drivers/iio/industrialio-buffer.c                                                                  
+-----------------------------------------+                                                         
| iio_buffer_register_legacy_sysfs_groups | : allo two attr, save in legacy group, register to sysfs
+-|---------------------------------------+                                                         
  |                                                                                                 
  |--> alloc attr, save in legacy buffer group                                                      
  |                                                                                                 
  |    +---------------------------------+                                                          
  +--> | iio_device_register_sysfs_group | register attr group to iio-dev                           
  |    +---------------------------------+                                                          
  |                                                                                                 
  |--> alloc attr, save in legacy scan-el group                                                     
  |                                                                                                 
  |    +---------------------------------+                                                          
  +--> | iio_device_register_sysfs_group | register attr group to iio-dev                           
       +---------------------------------+                                                          
```

```
drivers/iio/industrialio-buffer.c                                                               
+-------------------------+                                                                      
| iio_device_buffer_ioctl | : given idx, get target attached buffer, prepare anon file, return fd
+-------------------------+                                                                      
| iio_device_buffer_getfd | : given idx, get target attached buffer, prepare anon file, return fd
+-|-----------------------+                                                                      
  |    +----------------+                                                                        
  |--> | copy_from_user | get idx from user space                                                
  |    +----------------+                                                                        
  |                                                                                              
  |--> given idx, get target attached buffer                                                     
  |                                                                                              
  |--> set 'busy' flag of buffer                                                                 
  |                                                                                              
  |--> alloc ib (iio_dev_buffer_pair)                                                            
  |                                                                                              
  |--> save iio-dev and attached-buffer in ib                                                    
  |                                                                                              
  |    +------------------+                                                                      
  |--> | anon_inode_getfd | prepare an anon file and install to fd table                         
  |    +------------------+ name = "iio:buffer"                                                  
  |                         fops = iio_buffer_chrdev_fileops                                     
  |    +--------------+                                                                          
  +--> | copy_to_user | copy fd to user space                                                    
       +--------------+                                                                          
```

```
drivers/iio/industrialio-core.c                                                       
+---------------------------+                                                          
| iio_device_register_sysfs | : add each channel to sysfs, register attr group to sysfs
+-|-------------------------+                                                          
  |                                                                                    
  |--> count attr#                                                                     
  |                                                                                    
  |--> if channel exists                                                               
  |    -                                                                               
  |    +--> for each channel                                                           
  |         |                                                                          
  |         |    +------------------------------+                                      
  |         +--> | iio_device_add_channel_sysfs | add channel to sysfs                 
  |              +------------------------------+                                      
  |    +---------------------------------+                                             
  +--> | iio_device_register_sysfs_group | register attr group to iio-dev              
       +---------------------------------+                                             
```

```
drivers/iio/industrialio-event.c                                                                   
+------------------------------+                                                                    
| iio_device_register_eventset | : alloc event iface & publish to sysfs, register handler to iio-dev
+-|----------------------------+                                                                    
  |                                                                                                 
  |--> alloc ev_int (event interface)                                                               
  |                                                                                                 
  |    +------------------------------+                                                             
  |--> | __iio_add_event_config_attrs | for each channel, add event to sysfs                        
  |    +------------------------------+                                                             
  |    +---------------------------------+                                                          
  |--> | iio_device_register_sysfs_group | register attr group to iio-dev                           
  |    +---------------------------------+                                                          
  |                                                                                                 
  |--> install ops                                                                                  
  |    +-----------------+                                                                          
  |    | iio_event_ioctl | get event interface, prepare anon file, return fd                        
  |    +-----------------+                                                                          
  |                                                                                                 
  |    +-----------------------------------+                                                        
  +--> | iio_device_ioctl_handler_register | register handler to iio-dev                            
       +-----------------------------------+                                                        
```
