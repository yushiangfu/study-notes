```
kcs_bmc_ipmi_init
ast_kcs_bmc_driver_init:  register platform driver 'ast_kcs_bmc_driver'
  aspeed_kcs_probe:       alloc and set up priv/kcs_bmc, register isr for h2b irq, enable channel, register kcs_bmc
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
