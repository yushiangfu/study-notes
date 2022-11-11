```
+------+                                                          
| main | : request service 'Settings' and generate lots of objects
+-|----+                                                          
  |                                                               
  |--> ->request_name("xyz.openbmc_project.Settings")             
  |                                                               
  |    +--------------+                                           
  +--> | loadSettings | generate lots of objects                  
       +--------------+                                           
```

```
+--------------+                                                             
| loadSettings | : generate lots of objects                                  
+-|------------+                                                             
  |                                                                          
  +--> ste up obj: "/xyz/openbmc_project/control/minimum_ship_level_required"
                   "/xyz/openbmc_project/control/host0/auto_reboot"          
                   "/xyz/openbmc_project/control/host0/boot"                 
                   "/xyz/openbmc_project/control/host0/boot/one_time"        
                   "/xyz/openbmc_project/control/host0/power_cap"            
                   "/xyz/openbmc_project/control/host0/power_restore_policy" 
                   "/xyz/openbmc_project/control/power_restore_delay"        
                   "/xyz/openbmc_project/control/host0/acpi_power_state"     
                   "/xyz/openbmc_project/time/owner"                         
                   "/xyz/openbmc_project/time/sync_method"                   
                   "/xyz/openbmc_project/network/host0/intf"                 
                   "/xyz/openbmc_project/network/host0/intf/addr"            
                   "/xyz/openbmc_project/control/host0/TPMEnable"            
                   "/xyz/openbmc_project/control/power_supply_redundancy"    
                   "/xyz/openbmc_project/control/host0/turbo_allowed"        
                   "/xyz/openbmc_project/control/host0/systemGUID"           
                   "/xyz/openbmc_project/software/bios_active"               
                   "/xyz/openbmc_project/software/cpld_active"               
                   "/xyz/openbmc_project/software"                           
                   "/xyz/openbmc_project/control/processor_error_config"     
                   "/xyz/openbmc_project/control/bmc_reset_disables"         
                   "/com/intel/control/ocotshutdown_policy_config"           
                   "/xyz/openbmc_project/Chassis/Control/NMISource"          
                   "/xyz/openbmc_project/state/chassis0"                     
                   "/xyz/openbmc_project/control/chassis_capabilities_config"
                   "/xyz/openbmc_project/control/thermal_mode"               
                   "/xyz/openbmc_project/control/cfm_limit"                  
                   "/xyz/openbmc_project/ipmi/sol/eth0"                      
                   "/xyz/openbmc_project/ipmi/sol/eth1"                      
                   "/xyz/openbmc_project/ipmi/sol/eth2"                      
                   "/xyz/openbmc_project/control/host0/ac_boot"              
                   "/xyz/openbmc_project/Inventory/Item/Dimm"                
                   "/xyz/openbmc_project/logging/rest_api_logs"              
                   "/xyz/openbmc_project/software/apply_time"                
                   "/xyz/openbmc_project/logging/settings"                   
                   "/xyz/openbmc_project/pfr/last_events"                    
```
