```
drivers/pwm/pwm-aspeed-ast2600.c                                                       
+------------------+                                                                    
| aspeed_pwm_probe | : alloc and setup pwm-chip (install ops), register to 'pwms_chips' 
+-|----------------+                                                                    
  |                                                                                     
  |--> alloc priv                                                                       
  |                                                                                     
  |    +-----------------------+                                                        
  |--> | syscon_node_to_regmap | read hw_reg info and do iomap (for later scu operation)
  |    +-----------------------+                                                        
  |    +--------------------------+                                                     
  |--> | devm_add_action_or_reset | add action as dev resource                          
  |    +--------------------------+ +-------------------------+                         
  |                                 | aspeed_pwm_reset_assert | assert reset            
  |                                 +-------------------------+                         
  |                                                                                     
  |--> for each child node (not our case)                                               
  |    |                                                                                
  |    |    +---------------------------+                                               
  |    +--> | aspeed_pwm_extend_feature |                                               
  |         +---------------------------+                                               
  |                                                                                     
  |--> install 'aspeed_pwm_ops'                                                         
  |    +------------------+                                                             
  |    | aspeed_pwm_apply |                                                             
  |    +----------------------+                                                         
  |    | aspeed_pwm_get_state |                                                         
  |    +----------------------+                                                         
  |    +-------------+                                                                  
  |--> | pwmchip_add | alloc pwm devs for chip, add chip to 'pwms_chips' list           
  |    +-------------+                                                                  
  |    +--------------------------+                                                     
  +--> | devm_add_action_or_reset | add action as dev resource                          
       +--------------------------+ +------------------------+                          
                                    | aspeed_pwm_chip_remove | remove pwmchip           
                                    +------------------------+                          
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
drivers/pwm/pwm-aspeed-ast2600.c                           
+------------------+                                        
| aspeed_pwm_apply | : calculate duty cycle and write to reg
+-|----------------+                                        
  |                                                         
  |--> calculate and determine div                          
  |                                                         
  |--> calculate duty with div                              
  |                                                         
  |    +--------------------+                               
  +--> | regmap_update_bits | write duty to reg             
       +--------------------+                               
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
drivers/hwmon/tach-aspeed-ast2600.c                                                                         
+-------------------+                                                                                        
| aspeed_tach_probe |                                                                                        
+-|-----------------+                                                                                        
  |                                                                                                          
  |--> alloc priv                                                                                            
  |                                                                                                          
  |    +-----------------------+                                                                             
  |--> | syscon_node_to_regmap | read hw_reg info and do iomap (for later scu operation)                     
  |    +-----------------------+                                                                             
  |                                                                                                          
  +--> for each child node (fan 0~15)                                                                        
       |                                                                                                     
       |    +------------------------+                                                                       
       |--> | aspeed_tach_create_fan | read dt property to get channel, config its hw registers and enable it
       |    +------------------------+                                                                       
       |    +--------------------------------------+                                                         
       +--> | devm_hwmon_device_register_with_info | setup hwmon_dev, register it, add to arg dev as resource
            +--------------------------------------+                                                         
```

```
drivers/hwmon/tach-aspeed-ast2600.c                                                               
+------------------------+                                                                         
| aspeed_tach_create_fan | : read dt property to get channel, config its hw registers and enable it
+-|----------------------+                                                                         
  |    +----------------------+                                                                    
  |--> | of_property_read_u32 | read 'reg' property                                                
  |    +----------------------+                                                                    
  |    +--------------------------------+                                                          
  +--> | aspeed_create_fan_tach_channel | config hw registers and enable this channel              
       +--------------------------------+                                                          
```

```
drivers/hwmon/tach-aspeed-ast2600.c                                            
+--------------------------------+                                              
| aspeed_create_fan_tach_channel | : config hw registers and enable this channel
+-|------------------------------+                                              
  |    +-------------------+                                                    
  |--> | regmap_write_bits | write limited_inverse                              
  |    +-------------------+                                                    
  |    +-------------------+                                                    
  |--> | regmap_write_bits | write tach_debounce                                
  |    +-------------------+                                                    
  |    +-------------------+                                                    
  |--> | regmap_write_bits | write tach_edge                                    
  |    +-------------------+                                                    
  |    +-------------------+                                                    
  |--> | regmap_write_bits | write tach_divisor                                 
  |    +-------------------+                                                    
  |    +-------------------+                                                    
  |--> | regmap_write_bits | write threshold                                    
  |    +-------------------+                                                    
  |    +-----------------------+                                                
  +--> | aspeed_tach_ch_enable | write reg to enable the tach channel           
       +-----------------------+                                                
```
