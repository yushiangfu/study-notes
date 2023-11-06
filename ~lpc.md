```
kcs_bmc_ipmi_init:            register 'kcs_bmc_ipmi_driver', probe devices on 'kcs_bmc_devices
ast_kcs_bmc_driver_init:      register platform driver 'ast_kcs_bmc_driver'
  aspeed_kcs_probe:           alloc and set up priv/kcs_bmc, register isr for h2b irq, enable channel, register kcs_bmc
aspeed_lpc_ctrl_driver_init:  register platform driver aspeed_lpc_ctrl_driver
  aspeed_lpc_ctrl_probe:      
reset_simple_driver_init:     register platform driver 'reset_simple_driver'
  reset_simple_probe:         set up 'data' and install ops, prepare reset_controller_dev
```

```
drivers/char/ipmi/kcs_bmc_cdev_ipmi.c                                                    
+-------------------+                                                                     
| kcs_bmc_ipmi_init | : register 'kcs_bmc_ipmi_driver', probe devices on 'kcs_bmc_devices'
+-|-----------------+                                                                     
  |    +-------------------------+                                                        
  +--> | kcs_bmc_register_driver | register kcs driver, probe devices on 'kcs_bmc_devices'
       +-------------------------+                                                        
```

```
drivers/char/ipmi/kcs_bmc.c                                                                                         
+-------------------------+                                                                                          
| kcs_bmc_register_driver | : register kcs driver, probe devices on 'kcs_bmc_devices'                                
+-|-----------------------+                                                                                          
  |                                                                                                                  
  |--> add driver to list 'kcs_bmc_drivers'                                                                          
  |                                                                                                                  
  +--> for each dev on 'kcs_bmc_devices'                                                                             
       -                                                                                                             
       +--> call ->add_device(), e.g.,                                                                               
            +-------------------------+                                                                              
            | kcs_bmc_ipmi_add_device | prepare 'priv' and ste up its misc_dev, register misc_dev, add 'priv' to list
            +-------------------------+                                                                              
```

```
drivers/char/ipmi/kcs_bmc_cdev_ipmi.c                                                                     
+-------------------------+                                                                                
| kcs_bmc_ipmi_add_device | : prepare 'priv' and ste up its misc_dev, register misc_dev, add 'priv' to list
+-|-----------------------+                                                                                
  |                                                                                                        
  |--> alloc 'priv'                                                                                        
  |                                                                                                        
  |--> install ops 'kcs_bmc_ipmi_client_ops'                                                               
  |                                                                                                        
  |--> alloc spaces for data_in/data_out/kbuffer                                                           
  |                                                                                                        
  |--> set up misc_dev and install fops 'kcs_bmc_ipmi_fops'                                                
  |                                                                                                        
  |    +---------------+                                                                                   
  |--> | misc_register | determine dev#, add arg 'misc' to list (misc_list)                                
  |    +---------------+                                                                                   
  |                                                                                                        
  |--> add 'priv' to list 'kcs_bmc_ipmi_instances'                                                         
  |                                                                                                        
  +--> print "Initialised IPMI client for channel %d"                                                      
```

```
 drivers/char/ipmi/kcs_bmc_aspeed.c                                                                             
+------------------+                                                                                            
| aspeed_kcs_probe | : alloc and set up priv/kcs_bmc, register isr for h2b irq, enable channel, register kcs_bmc
+-|----------------+                                                                                            
  |                                                                                                             
  |--> get matched data (ops)                                                                                   
  |                                                                                                             
  |--> call ->get_channel(), e.g.,                                                                              
  |    +------------------------------+                                                                         
  |    | aspeed_kcs_of_v1_get_channel | read node property 'kcs_chan'                                           
  |    +------------------------------+                                                                         
  |                                                                                                             
  |--> call ->get_io_address(), e.g.,                                                                           
  |    +---------------------------------+                                                                      
  |    | aspeed_kcs_of_v1_get_io_address | read node property 'kcs_addr'                                        
  |    +---------------------------------+                                                                      
  |                                                                                                             
  |--> read property "aspeed,lpc-interrupts" to see if we have upstream irq                                     
  |                                                                                                             
  |--> alloc 'priv'                                                                                             
  |                                                                                                             
  |--> set up 'kcs_bmc' and install ops 'aspeed_kcs_ops'                                                        
  |                                                                                                             
  |    +-------------+ +----------------------+                                                                 
  |--> | timer_setup | | aspeed_kcs_check_obe | read a byte and handle that event                               
  |    +-------------+ +----------------------+                                                                 
  |                                                                                                             
  |--> mask irq                                                                                                 
  |                                                                                                             
  |    +------------------------+                                                                               
  |--> | aspeed_kcs_set_address | ???                                                                           
  |    +------------------------+                                                                               
  |    +----------------------------------+                                                                     
  |--> | aspeed_kcs_config_downstream_irq | get irq# and register isr (for host-to-bmc irq)                     
  |    +----------------------------------+                                                                     
  |                                                                                                             
  |--> if we have upstream irq (from bmc to host) (not our case), skip                                          
  |                                                                                                             
  |    +---------------------------+                                                                            
  |--> | aspeed_kcs_enable_channel |                                                                            
  |    +---------------------------+                                                                            
  |    +--------------------+                                                                                   
  |--> | kcs_bmc_add_device | add 'kcs_bmc' to list, call each kcs drvier to add it as well                     
  |    +--------------------+                                                                                   
  |                                                                                                             
  +--> print "Initialised channel %d at 0x%x\n"                                                                 
```

```
drivers/char/ipmi/kcs_bmc_cdev_ipmi.c                                             
+----------------------+                                                           
| aspeed_kcs_check_obe | : get client of kcs_bmc to handle event (cmd or data)     
+-|--------------------+                                                           
  |    +----------------+                                                          
  |--> | aspeed_kcs_inb | read a byte from hw reg                                  
  |    +----------------+                                                          
  |                                                                                
  |--> if bit 'obf' is set                                                         
  |    |                                                                           
  |    |    +-----------+                                                          
  |    |--> | mod_timer | respawn timer later                                      
  |    |    +-----------+                                                          
  |    |                                                                           
  |    +--> return                                                                 
  |                                                                                
  |    +----------------------+                                                    
  +--> | kcs_bmc_handle_event | get client of kcs_bmc to handle event (cmd or data)
       +----------------------+                                                    
```

```
drivers/char/ipmi/kcs_bmc.c                                                  
+----------------------+                                                      
| kcs_bmc_handle_event | : get client of kcs_bmc to handle event (cmd or data)
+-|--------------------+                                                      
  |                                                                           
  |--> get client from kcs_bmc                                                
  |                                                                           
  +--> call ->event(), e.g.,                                                  
       +--------------------+                                                 
       | kcs_bmc_ipmi_event | read status, handle cmd or data accordingly     
       +--------------------+                                                 
```

```
drivers/char/ipmi/kcs_bmc_cdev_ipmi.c                              
+--------------------+                                              
| kcs_bmc_ipmi_event | : read status, handle cmd or data accordingly
+-|------------------+                                              
  |    +---------------------+                                      
  |--> | kcs_bmc_read_status |                                      
  |    +---------------------+                                      
  |                                                                 
  |--> read status                                                  
  |                                                                 
  |--> if it's about cmd                                            
  |                                                                 
  |        +-------------------------+                              
  |------> | kcs_bmc_ipmi_handle_cmd | handle cmd                   
  |        +-------------------------+                              
  |                                                                 
  |--> else (data)                                                  
  |                                                                 
  |        +--------------------------+                             
  +------> | kcs_bmc_ipmi_handle_data | handle data                 
           +--------------------------+                             
```

```
drivers/char/ipmi/kcs_bmc_aspeed.c                                                   
+----------------------------------+                                                  
| aspeed_kcs_config_downstream_irq | : get irq# and register isr (for host-to-bmc irq)
+-|--------------------------------+                                                  
  |    +------------------+                                                           
  |--> | platform_get_irq | get irq from dt node                                      
  |    +------------------+                                                           
  |    +------------------+ +----------------+                                        
  +--> | devm_request_irq | | aspeed_kcs_irq | handle event                           
       +------------------+ +----------------+                                        
```

```
drivers/soc/aspeed/aspeed-lpc-ctrl.c                                                                              
+-----------------------+                                                                                          
| aspeed_lpc_ctrl_probe | : prepare 'lpc_ctrl', set up misc dev 'aspeed-lpc-ctrl, install fops and register the dev
+-|---------------------+                                                                                          
  |                                                                                                                
  |--> alloc 'lpc_ctrl'                                                                                            
  |                                                                                                                
  +--> handle property 'flash' and 'memory-region'                                                                 
  |                                                                                                                
  |    +-----------------------+                                                                                   
  |--> | syscon_node_to_regmap | ensure system controller exists and return its register map                       
  |    +-----------------------+                                                                                   
  |                                                                                                                
  |--> set up misc dev "aspeed-lpc-ctrl" and install fops 'aspeed_lpc_ctrl_fops'                                   
  |                                                                                                                
  |    +---------------+                                                                                           
  +--> | misc_register | determine dev#, add arg 'misc' to list (misc_list)                                        
       +---------------+                                                                                           
```

```
drivers/reset/reset-simple.c                                                                               
+--------------------+                                                                                      
| reset_simple_probe | : set up 'data' and install ops, prepare reset_controller_dev                        
+-|------------------+                                                                                      
  |                                                                                                         
  |--> alloc 'data'                                                                                         
  |                                                                                                         
  |--> get resource and iomap                                                                               
  |                                                                                                         
  |--> set up 'data' and install ops                                                                        
  |                                                                                                         
  |    +--------------------------------+                                                                   
  +--> | devm_reset_controller_register | alloc reset_controller_dev, register it and add to dev as resource
       +--------------------------------+                                                                   
```

```
drivers/reset/core.c                                                                                  
+--------------------------------+                                                                     
| devm_reset_controller_register | : alloc reset_controller_dev, register it and add to dev as resource
+-|------------------------------+                                                                     
  |                                                                                                    
  |--> alloc reset_controller_dev as resource                                                          
  |                                                                                                    
  |    +---------------------------+                                                                   
  |--> | reset_controller_register | add reset_controller_dev to 'reset_controller_list'               
  |    +---------------------------+                                                                   
  |                                                                                                    
  +--> add resource to dev                                                                             
```

```
drivers/clk/clk-aspeed.c                                                                                   
+------------------+                                                                                        
| aspeed_clk_probe | : setup reset controler, register clocks and save in aspeed_clk_data[]                 
+-|----------------+                                                                                        
  |    +-----------------------+                                                                            
  |--> | syscon_node_to_regmap | ensure system controller exists and return its register map                
  |    +-----------------------+                                                                            
  |    +--------------+                                                                                     
  |--> | devm_kzalloc | alloc aspeed_reset                                                                  
  |    +--------------+                                                                                     
  |                                                                                                         
  |--> setup aspeed_reset and install 'aspeed_reset_ops'                                                    
  |                                                                                                         
  |    +--------------------------------+                                                                   
  |--> | devm_reset_controller_register | alloc reset_controller_dev, register it and add to dev as resource
  |    +--------------------------------+                                                                   
  |    +----------------------------+                                                                       
  |--> | clk_hw_register_fixed_rate | alloc and setup 'fixed', register to proper list and enable           
  |    +----------------------------+ ("uart")                                                              
  |    +----------------------+                                                                             
  |--> | clk_hw_register_gate | alloc and setup 'gate', register to proper list and enable                  
  |    +----------------------+ ("sd_extclk_gate")                                                          
  |    +-------------------------------+                                                                    
  |--> | clk_hw_register_divider_table | alloc and setup 'divider', register to proper list and enable      
  |    +-------------------------------+ ("sd_extclk")                                                      
  |    +-------------------------------+                                                                    
  |--> | clk_hw_register_divider_table | alloc and setup 'divider', register to proper list and enable      
  |    +-------------------------------+ ("mac")                                                            
  |    +-------------------------------+                                                                    
  |--> | clk_hw_register_divider_table | alloc and setup 'divider', register to proper list and enable      
  |    +-------------------------------+ ("lhclk")                                                          
  |    +-------------------------------+                                                                    
  |--> | clk_hw_register_divider_table | alloc and setup 'divider', register to proper list and enable      
  |    +-------------------------------+ ("bclk")                                                           
  |    +----------------------------+                                                                       
  |--> | clk_hw_register_fixed_rate | alloc and setup 'fixed', register to proper list and enable           
  |    +----------------------------+ ("fixed-24m")                                                         
  |    +---------------------+                                                                              
  |--> | clk_hw_register_mux | alloc and setup 'fixed', register to proper list and enable                  
  |    +---------------------+ ("eclk-mux")                                                                 
  |    +-------------------------------+                                                                    
  |--> | clk_hw_register_divider_table | alloc and setup 'divider', register to proper list and enable      
  |    +-------------------------------+ ("eclk")                                                           
  |                                                                                                         
  +--> for each aspeed gate                                                                                 
       |                                                                                                    
       |    +-----------------------------+                                                                 
       +--> | aspeed_clk_hw_register_gate | alloc and setup 'gate', register to proper list and enable      
            +-----------------------------+                                                                 
```

```
include/linux/clk-provider.h                                                                                            
+----------------------------+                                                                                           
| clk_hw_register_fixed_rate | : alloc and setup 'fixed', register to proper list and enable                             
+------------------------------+                                                                                         
| __clk_hw_register_fixed_rate | : alloc and setup 'fixed', register to proper list and enable                           
+-|----------------------------+                                                                                         
  |                                                                                                                      
  |--> alloc 'fixed'                                                                                                     
  |                                                                                                                      
  |--> set up local 'init' and save in 'fixed'                                                                           
  |                                                                                                                      
  |--> if dev is provided || node isn't provided                                                                         
  |    |                                                                                                                 
  |    |    +-----------------+                                                                                          
  |    +--> | clk_hw_register | alloc and set up core, prepare clk and add to core, add core to proper list and enable   
  |         +-----------------+                                                                                          
  |                                                                                                                      
  +--> elif node is provided                                                                                             
       |                                                                                                                 
       |    +--------------------+                                                                                       
       +--> | of_clk_hw_register | alloc and set up core, prepare clk and add to core, add core to proper list and enable
            +--------------------+                                                                                       
```

```
drivers/clk/clk.c                                                                                          
+-----------------+                                                                                         
| clk_hw_register | : alloc and set up core, prepare clk and add to core, add core to proper list and enable
+----------------++                                                                                         
| __clk_register | : alloc and set up core, prepare clk and add to core, add core to proper list and enable 
+-|--------------+                                                                                          
  |                                                                                                         
  |--> alloc 'core'                                                                                         
  |                                                                                                         
  |--> core->ops = init->ops                                                                                
  |                                                                                                         
  |--> setup other fields of 'core'                                                                         
  |                                                                                                         
  |    +------------------------------+                                                                     
  |--> | clk_core_populate_parent_map | alloc and setup 'parents'                                           
  |    +------------------------------+                                                                     
  |    +-----------+                                                                                        
  |--> | alloc_clk | alloc and setup 'clk'                                                                  
  |    +-----------+                                                                                        
  |    +------------------------+                                                                           
  |--> | clk_core_link_consumer | add clk to core                                                           
  |    +------------------------+                                                                           
  |    +-----------------+                                                                                  
  +--> | __clk_core_init | add 'core' to proper list, set duty_cycle/rate, prepare() and enable()           
       +-----------------+                                                                                  
```

```
drivers/clk/clk.c                                          
+------------------------------+                            
| clk_core_populate_parent_map | : alloc and setup 'parents'
+-|----------------------------+                            
  |                                                         
  |--> alloc 'parents' and save in 'core'                   
  |                                                         
  +--> for each parent                                      
       -                                                    
       +--> set up parent                                   
```

```
drivers/clk/clk.c                                                                          
+-----------------+                                                                         
| __clk_core_init | : add 'core' to proper list, set duty_cycle/rate, prepare() and enable()
+-|---------------+                                                                         
  |    +--------------------+                                                               
  |--> | clk_pm_runtime_get | (do nothing in our case)                                      
  |    +--------------------+                                                               
  |                                                                                         
  |--> check if 'core' is already registered                                                
  |                                                                                         
  |--> if ->init exists, call it                                                            
  |                                                                                         
  |    +-------------------+                                                                
  |--> | __clk_init_parent | ensure parent has 'core'                                       
  |    +-------------------+                                                                
  |                                                                                         
  |--> add 'core' to proper list                                                            
  |                                                                                         
  |--> set core's accuracy                                                                  
  |                                                                                         
  |    +--------------------+                                                               
  |--> | clk_core_get_phase | call ->get_phase()                                            
  |    +--------------------+                                                               
  |    +-----------------------------------+                                                
  |--> | clk_core_update_duty_cycle_nolock | set clock's duty cycle                         
  |    +-----------------------------------+                                                
  |                                                                                         
  |--> set core's rate                                                                      
  |                                                                                         
  |--> if flag has is_critical                                                              
  |    |                                                                                    
  |    |    +------------------+                                                            
  |    +--> | clk_core_prepare | recursively call parent's ->prepare()                      
  |         +------------------+                                                            
  |    +----------------------+                                                             
  +--> | clk_core_enable_lock | call ->enable()                                             
  |    +----------------------+                                                             
  |    +----------------------------------+                                                 
  +--> | clk_core_reparent_orphans_nolock | re-parent orphan clocks in list                 
       +----------------------------------+                                                 
```

```
include/linux/clk-provider.h                                                                                            
+----------------------+                                                                                                 
| clk_hw_register_gate | : alloc and setup 'gate', register to proper list and enable                                    
+------------------------+                                                                                               
| __clk_hw_register_gate | : alloc and setup 'gate', register to proper list and enable                                  
+-|----------------------+                                                                                               
  |                                                                                                                      
  |--> alloc 'gate'                                                                                                      
  |                                                                                                                      
  |--> set up local 'init', install 'clk_gate_ops', save in 'gate'                                                       
  |                                                                                                                      
  |--> if dev is provided || node isn't provided                                                                         
  |    |                                                                                                                 
  |    |    +-----------------+                                                                                          
  |    +--> | clk_hw_register | alloc and set up core, prepare clk and add to core, add core to proper list and enable   
  |         +-----------------+                                                                                          
  |                                                                                                                      
  +--> elif node is provided                                                                                             
       |                                                                                                                 
       |    +--------------------+                                                                                       
       +--> | of_clk_hw_register | alloc and set up core, prepare clk and add to core, add core to proper list and enable
            +--------------------+                                                                                       
```

```
drivers/reset/core.c                                                                   
+--------------------------+                                                            
| __devm_reset_control_get | : get reset_control and add to dev as resource             
+-|------------------------+                                                            
  |                                                                                     
  |--> alloc ptr to reset_control_release                                               
  |                                                                                     
  |    +---------------------+                                                          
  |--> | __reset_control_get | get target rc_dev from list, given id, get rc from rc_dev
  |    +---------------------+                                                          
  |                                                                                     
  |--> *ptr = reset_control                                                             
  |                                                                                     
  |    +------------+                                                                   
  +--> | devres_add | add ptr to dev as resource                                        
       +------------+                                                                   
```

```
drivers/reset/core.c                                                                           
+---------------------+                                                                         
| __reset_control_get | : get target rc_dev from list, given id, get rc from rc_dev             
+-|-------------------+                                                                         
  |                                                                                             
  |--> if dev->of_node                                                                          
  |    |                                                                                        
  |    |    +------------------------+                                                          
  |    +--> | __of_reset_control_get | get target rc_dev from list, given id, get rc from rc_dev
  |         +------------------------+                                                          
  |                                                                                             
  +--> else                                                                                     
       |                                                                                        
       |    +---------------------------------+                                                 
       +--> | __reset_control_get_from_lookup | (skip)                                          
            +---------------------------------+                                                 
```

```
drivers/reset/core.c                                                                             
+------------------------+                                                                        
| __of_reset_control_get | : get target rc_dev from list, given id, get rc from rc_dev            
+-|----------------------+                                                                        
  |                                                                                               
  |--> if arg id is provided                                                                      
  |    |                                                                                          
  |    |    +--------------------------+                                                          
  |    +--> | of_property_match_string | get property 'reset-names'                               
  |         +--------------------------+                                                          
  |    +----------------------------+                                                             
  |--> | of_parse_phandle_with_args | get property 'resets'                                       
  |    +----------------------------+                                                             
  |                                                                                               
  |--> for each entry in 'reset_controller_list'                                                  
  |    -                                                                                          
  |    +--> if target entry found (e.g., syscon)                                                  
  |         -                                                                                     
  |         +--> break                                                                            
  |                                                                                               
  |--> call ->of_xlate() to translate args into id                                                
  |                                                                                               
  |    +------------------------------+                                                           
  +--> | __reset_control_get_internal | ensure reset_control of matched id is in reset_control_dev
       +------------------------------+                                                           
```

```
drivers/reset/core.c                                                                                                   
+------------------------------+                                                                                        
| __reset_control_get_internal | : ensure reset_control of matched id is in reset_control_dev                           
+-|----------------------------+                                                                                        
  |                                                                                                                     
  |--> for each rest_control in reset_control_dev                                                                       
  |    -                                                                                                                
  |    +--> if matched id found                                                                                         
  |         |                                                                                                           
  |         |--> if reset_control isn't for shared && caller isn't asking for shared && caller isn't asking for acquired
  |         |    -                                                                                                      
  |         |    +--> break                                                                                             
  |         |                                                                                                           
  |         |--> if reset_control isn't for shared || call isn't asking for shared                                      
  |         |    -                                                                                                      
  |         |    +--> warn and return error                                                                             
  |         |                                                                                                           
  |         +--> return reset_control                                                                                   
  |                                                                                                                     
  |--> alloc reset_control                                                                                              
  |                                                                                                                     
  |--> add reset_control to reset_control_dev                                                                           
  |                                                                                                                     
  |--> ->acquired = arg acquired                                                                                        
  |                                                                                                                     
  +--> ->shared = arg shared                                                                                            
```
