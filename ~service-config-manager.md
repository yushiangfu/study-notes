```
+------+                                                                                                    
| main | check and init                                                                                     
+-|----+                                                                                                    
  |                                                                                                         
  |--> request service: "xyz.openbmc_project.Control.Service.Manager"                                       
  |                                                                                                         
  |--> set up obj (?): "/xyz/openbmc_project/control/service"                                               
  |                                                                                                         
  |--> prepare match rule and callback                                                                      
  |       +------------------------------------------------------------------------------------------------+
  |       |if query hasn't started yet                                                                     |
  |       ||                                                                                               |
  |       ||--> set query_started = true                                                                   |
  |       ||                                                                                               |
  |       ||    +------+                                                                                   |
  |       |+--> | init | get units from service 'systemd1', write to file and create dbus objects for them |
  |       |     +------+                                                                                   |
  |       +------------------------------------------------------------------------------------------------+
  |    +--------------+                                                                                     
  +--> | checkAndInit | check and init                                                                      
       +--------------+                                                                                     
```

```
+------+                                                                                                                     
| init | get units from service 'systemd1', write to file and create dbus objects for them                                   
+-|----+                                                                                                                     
  |                                                                                                                          
  +--> ->async_method_call                                                                                                   
          +-----------------------------------------------------------------------------------------------------------------+
          |+-------------------------+                                                                                      |
          || handleListUnitsResponse | add units to units_to_monitor, writeback to file, create objects for needed services |
          |+-------------------------+                                                                                      |
          +-----------------------------------------------------------------------------------------------------------------+
          service: "org.freedesktop.systemd1"                                                                                
          object: "/org/freedesktop/systemd1"                                                                                
          iface: "org.freedesktop.systemd1.Manager"                                                                          
          method: "ListUnits"                                                                                                
```

```
+-------------------------+                                                                                       
| handleListUnitsResponse | : add units to units_to_monitor, writeback to file, create objects for needed services
+-|-----------------------+                                                                                       
  |                                                                                                               
  |--> for each unit                                                                                              
  |    |                                                                                                          
  |    |    +----------------------------+                                                                        
  |    |--> | getUnitNameTypeAndInstance | given string, determine type and parse instance & unit name            
  |    |    +----------------------------+                                                                        
  |    |                                                                                                          
  |    +--> if unit_name is in service_names                                                                      
  |         |                                                                                                     
  |         |--> determine instantiated_unit_name (_40 means '@', _2d means '-')                                  
  |         |                                                                                                     
  |         |--> if the name is in units_to_monitor                                                               
  |         |    |                                                                                                
  |         |    |--> if the type is 'service'                                                                    
  |         |    |    -                                                                                           
  |         |    |    +--> save unit_name, service_name, and object_path in value                                 
  |         |    |                                                                                                
  |         |    +--> elif the type is 'socket'                                                                   
  |         |         -                                                                                           
  |         |         +--> save object_path in value                                                              
  |         |                                                                                                     
  |         +--> ensure instantiated_unit_name is in units_to_monitor                                             
  |                                                                                                               
  |--> if "/etc/srvcfg-mgr.json" exists                                                                           
  |    -                                                                                                          
  |    +--> update saved_monitor_list by units_to_monitor                                                         
  |                                                                                                               
  |--> writeback to file                                                                                          
  |                                                                                                               
  +--> create objects for needed service                                                                          
```

```
+----------------------------+                                                              
| getUnitNameTypeAndInstance | : given string, determine type and parse instance & unit name
+-|--------------------------+                                                              
  |                                                                                         
  +--> if there is '.' in string                                                            
       |                                                                                    
       |--> determine type: 'service' or 'socket'                                           
       |                                                                                    
       |--> if there is '@' in string                                                       
       |    |                                                                               
       |    |--> determine intance name (string after '@')                                  
       |    |                                                                               
       |    +--> determine unit name                                                        
       |                                                                                    
       +--> else                                                                            
            -                                                                               
            +--> determine unit name                                                        
```

```
+--------------+                                                                                            
| checkAndInit | : check and init                                                                           
+-|------------+                                                                                            
  |                                                                                                         
  +--> ->async_method_call                                                                                  
          +------------------------------------------------------------------------------------------------+
          |if query hasn't started yet                                                                     |
          ||                                                                                               |
          ||--> set query_started = true                                                                   |
          ||                                                                                               |
          ||    +------+                                                                                   |
          ||--> | init | get units from service 'systemd1', write to file and create dbus objects for them |
          ||    +------+                                                                                   |
          ||                                                                                               |
          |+--> ->async_wait                                                                               |
          |        +-----------------------------+                                                         |
          |        |+--------------+             |                                                         |
          |        || checkAndInit | (recursive) |                                                         |
          |        |+--------------+             |                                                         |
          |        +-----------------------------+                                                         |
          +------------------------------------------------------------------------------------------------+
          service: "org.freedesktop.systemd1"                                                               
          object: "/org/freedesktop/systemd1"                                                               
          iface: "org.freedesktop.DBus.Properties"                                                          
          method: "Get"                                                                                     
```
