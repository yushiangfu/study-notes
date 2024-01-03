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
