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
