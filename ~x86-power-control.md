```
src/power_control.cpp                                                                             
+------+                                                                                           
| main |                                                                                           
+-|----+                                                                                           
  |    +-------+                                                                                   
  |--> | part0 | request services, monitor event: power_ok, sio_xxx                                
  |    +-------+                                                                                   
  |    +-------+                                                                                   
  |--> | part1 | handle gpios (register handlers, set direction), prepare match rules and callbacks
  |    +-------+                                                                                   
  |    +-------+                                                                                   
  |--> | part2 | handle host state                                                                 
  |    +-------+                                                                                   
  |    +-------+                                                                                   
  |--> | part3 | handle chassis state                                                              
  |    +-------+                                                                                   
  |    +-------+                                                                                   
  |--> | part4 | handle chassis_system state                                                       
  |    +-------+                                                                                   
  |    +-------+                                                                                   
  |--> | part5 | handle power_button and reset_button                                              
  |    +-------+                                                                                   
  |    +-------+                                                                                   
  |--> | part6 | handle nmi_button, nmi_out, id_button, and post_complete                          
  |    +-------+                                                                                   
  |    +-------+                                                                                   
  +--> | part7 | handle restart_cause, monitor host state                                          
       +-------+                                                                                   
```

```
src/power_control.cpp                                                                                            
+-------+                                                                                                         
| part0 | : request services, monitor event: power_ok, sio_xxx                                                    
+-|-----+                                                                                                         
  |    +------------------+                                                                                       
  |--> | loadConfigValues | load power/gpio config                                                                
  |    +------------------+                                                                                       
  |                                                                                                               
  |--> if arg node is 0                                                                                           
  |    -                                                                                                          
  |    +--> request services: xyz.openbmc_project.State.Host                                                      
  |                           xyz.openbmc_project.State.Chassis                                                   
  |                           xyz.openbmc_project.State.OperatingSystem                                           
  |                           xyz.openbmc_project.Chassis.Buttons                                                 
  |                           xyz.openbmc_project.Control.Host.NMI                                                
  |                           xyz.openbmc_project.Control.Host.RestartCause                                       
  |                                                                                                               
  |--> append node to each service name                                                                           
  |                                                                                                               
  |--> request services, e.g.: xyz.openbmc_project.State.Host0                                                    
  |                            xyz.openbmc_project.State.Chassis0                                                 
  |                            xyz.openbmc_project.State.OperatingSystem0                                         
  |                            xyz.openbmc_project.Chassis.Buttons0                                               
  |                            xyz.openbmc_project.Control.Host.NMI0                                              
  |                            xyz.openbmc_project.Control.Host.RestartCause0                                     
  |                                                                                                               
  |--> if power_ok type is 'gpio'                                                                                 
  |    |                                                                                                          
  |    |    +-------------------+                                                                                 
  |    +--> | requestGPIOEvents | find name-matched line, get its fd, wait event and call handler                 
  |         +-------------------+ +------------------+                                                            
  |                               | psPowerOKHandler | given current power state,                                 
  |                               +------------------+ select target handler (e.g., on) to handle event (power ok)
  |                                                                                                               
  |--> elif 'dbus', (ignore)                                                                                      
  |                                                                                                               
  +--> if sio is enabled                                                                                          
       |                                                                                                          
       |    +-------------------+                +---------------------+                                          
       |--> | requestGPIOEvents | sio_power_good | sioPowerGoodHandler |                                          
       |    +-------------------+                +---------------------+                                          
       |    +-------------------+                +---------------------+                                          
       |--> | requestGPIOEvents | sio_on_control | sioOnControlHandler |                                          
       |    +-------------------+                +---------------------+                                          
       |    +-------------------+        +--------------+                                                         
       +--> | requestGPIOEvents | sio_s5 | sioS5Handler |                                                         
            +-------------------+        +--------------+                                                         
```

```
src/power_control.cpp                                                     
+------------------+                                                       
| loadConfigValues | : load power/gpio config                              
+-|----------------+                                                       
  |                                                                        
  |--> read json file and parse                                            
  |                                                                        
  |--> for each node under 'gpio_configs'                                  
  |    |                                                                   
  |    |--> if type is 'gpio'                                              
  |    |    -                                                              
  |    |    +--> handle line_name and polarity                             
  |    |                                                                   
  |    +--> else (type is 'dbus'                                           
  |         -                                                              
  |         +--> handle bus_name, obj_path, interface, property (line_name)
  |                                                                        
  +--> overwrite default TimerMap from config                              
```

```
src/power_control.cpp                                                                                  
+----------------------------------+                                                                    
| power_control::requestGPIOEvents | : find name-matched line, get its fd, wait event and call handler  
+-|--------------------------------+                                                                    
  |    +------------------+                                                                             
  |--> | gpiod::find_line | find name-matched line                                                      
  |    +------------------+                                                                             
  |    +---------------+                                                                                
  |--> | line::request | prepare bulk, add lines into it                                                
  |    +---------------+                                                                                
  |    +--------------------+                                                                           
  |--> | line::event_get_fd | given line, get fd                                                        
  |    +--------------------+                                                                           
  |                                                                                                     
  |--> assign fd to arg gpio_event_descriptor                                                           
  |                                                                                                     
  |    +---------------------------------+                                                              
  +--> | power_control::waitForGPIOEvent | wait, read events from fd, call ->handler(), get back to wait
       +---------------------------------+                                                              
```

```
src/power_control.cpp                                                                             
+---------------------------------+                                                                
| power_control::waitForGPIOEvent | : wait, read events from fd, call ->handler(), get back to wait
+-|-------------------------------+                                                                
  |                                                                                                
  +--> .async_wait                                                                                 
          +------------------------------------------------+                                       
          |+------------------+                            |                                       
          || line::event_read | read events from fd        |                                       
          |+------------------+                            |                                       
          |                                                |                                       
          |call arg event_handler, e.g.,                   |                                       
          |+------------------+                            |                                       
          || psPowerOKHandler |                            |                                       
          |+------------------+                            |                                       
          |                                                |                                       
          |+---------------------------------+             |                                       
          || power_control::waitForGPIOEvent | (recursive) |                                       
          |+---------------------------------+             |                                       
          +------------------------------------------------+                                       
```

```
src/power_control.cpp                                                                                                           
+------------------+                                                                                                             
| psPowerOKHandler | : given current power state, select target handler (e.g., on) to handle event (power ok)                    
+-|----------------+                                                                                                             
  |                                                                                                                              
  |--> given state, determin event (power_ok_assert ot power_ok_deassert)                                                        
  |                                                                                                                              
  |    +-----------------------+                                                                                                 
  +--> | sendPowerControlEvent | given current power state, select target handler (e.g., on) to handle event (e.g., graceful off)
       +-----------------------+                                                                                                 
```

```
src/power_control.cpp                                                                                                      
+-----------------------+                                                                                                   
| sendPowerControlEvent | : given current power state, select target handler (e.g., on) to handle event (e.g., graceful off)
+-|---------------------+                                                                                                   
  |    +----------------------+                                                                                             
  |--> | getPowerStateHandler | given power state, return target handler                                                    
  |    +----------------------+                                                                                             
  |                                                                                                                         
  +--> call handler, e.g.,                                                                                                  
       +--------------+                                                                                                     
       | powerStateOn | handle a few events: e.g., power_button_pressed, power_off, graceful_power_off                      
       +--------------+                                                                                                     
```

```
src/power_control.cpp                                                                                              
+-------+                                                                                                           
| part1 | : handle gpios (register handlers, set direction), prepare match rules and callbacks                      
+-|-----+                                                                                                           
  |    +-------------------+              +--------------------+                                                    
  |--> | requestGPIOEvents | power_button | powerButtonHandler |                                                    
  |    +-------------------+              +--------------------+                                                    
  |    +-------------------+              +--------------------+                                                    
  |--> | requestGPIOEvents | reset_button | resetButtonHandler |                                                    
  |    +-------------------+              +--------------------+                                                    
  |    +-------------------+            +------------------+                                                        
  |--> | requestGPIOEvents | nmi_button | nmiButtonHandler |                                                        
  |    +-------------------+            +------------------+                                                        
  |    +-------------------+           +-----------------+                                                          
  |--> | requestGPIOEvents | id_button | idButtonHandler |                                                          
  |    +-------------------+           +-----------------+                                                          
  |    +-------------------+               +---------------------+                                                  
  +--> | requestGPIOEvents | post_complete | postCompleteHandler |                                                  
  |    +-------------------+               +---------------------+                                                  
  |    +---------------+                                                                                            
  |--> | setGPIOOutput | set gpio nmi_out's direction to output (and value)                                         
  |    +---------------+                                                                                            
  |    +---------------+                                                                                            
  |--> | setGPIOOutput | set gpio power_out's direction to output (and value)                                       
  |    +---------------+                                                                                            
  |    +---------------+                                                                                            
  |--> | setGPIOOutput | set gpio reset_out's direction to output (and value)                                       
  |    +---------------+                                                                                            
  |                                                                                                                 
  |--> determine power_state (on?)                                                                                  
  |                                                                                                                 
  |--> if it's not on                                                                                               
  |    |                                                                                                            
  |    |    +-----------------------------+                                                                         
  |    +--> | PowerRestoreController::run | prepare match rules and callback, query other services to set properties
  |         +-----------------------------+                                                                         
  |                                                                                                                 
  |--> if nmi_out_line is configured                                                                                
  |    |                                                                                                            
  |    |    +-----------------------------------------+                                                             
  |    +--> | power_control::nmiSourcePropertyMonitor | prepare match rule and callback for nmi event               
  |         +-----------------------------------------+                                                             
  |    +--------------------+                                                                                       
  +--> | logStateTransition | log                                                                                   
       +--------------------+                                                                                       
```

```
src/power_control.cpp                                                                  
+------------------------------+                                                        
| power_control::setGPIOOutput | : find target line, set direction to output (and value)
+-|----------------------------+                                                        
  |    +-------------------+                                                            
  |--> | gpiod::find_line  | find name-matched line                                     
  |    +-------------------+                                                            
  |    +---------------+                                                                
  +--> | line::request | given config, set target lines                                 
       +---------------+                                                                
```

```
src/power_control.cpp                                                                                    
+-----------------------------+                                                                           
| PowerRestoreController::run | : prepare match rules and callback, query other services to set properties
+-|---------------------------+                                                                           
  |                                                                                                       
  |--> obj = "/xyz/openbmc_project/control/host" + node + "/power_restore_policy"                         
  |                                                                                                       
  |    +-----------------------+                                                                          
  |--> | powerRestorePolicyLog | log                                                                      
  |    +-----------------------+                                                                          
  |                                                                                                       
  |--> if match is empty (the list isn't created yet)                                                     
  |    -                                                                                                  
  |    +--> prepare match rules and callback                                                              
  |         +---------------------------+                                                                 
  |         | powerRestoreConfigHandler | (skip)                                                          
  |         +---------------------------+                                                                 
  |                                                                                                       
  |--> ->async_method_call                                                                                
  |       +-------------------------+                                                                     
  |       |+---------------+        |                                                                     
  |       || setProperties | (skip) |                                                                     
  |       |+---------------+        |                                                                     
  |       +-------------------------+                                                                     
  |       service: xyz.openbmc_project.Settings                                                           
  |       object: "/xyz/openbmc_project/control/host" + node + "/power_restore_policy"                    
  |       iface: org.freedesktop.DBus.Properties                                                          
  |       method: GetAll                                                                                  
  |       arg: xyz.openbmc_project.Control.Power.RestorePolicy                                            
  |                                                                                                       
  +--> ->async_method_call                                                                                
          +-------------------------+                                                                     
          |+---------------+        |                                                                     
          || setProperties | (skip) |                                                                     
          |+---------------+        |                                                                     
          +-------------------------+                                                                     
          service: xyz.openbmc_project.Settings                                                           
          object: /xyz/openbmc_project/control/host0/ac_boot                                              
          iface: org.freedesktop.DBus.Properties                                                          
          method: GetAll                                                                                  
          arg: xyz.openbmc_project.Common.ACBoot                                                          
```

```
src/power_control.cpp                                                                     
+-----------------------------------------+                                                
| power_control::nmiSourcePropertyMonitor | : prepare match rule and callback for nmi event
+-|---------------------------------------+                                                
  |                                                                                        
  +--> prepare match rule and callback                                                     
          +----------------------------------------+                                       
          |read msg                                |                                       
          |                                        |                                       
          |if the first field shows it's 'enabled' |                                       
          ||                                       |                                       
          ||--> get value from the 2nd field       |                                       
          ||                                       |                                       
          |+--> if the value is set                |                                       
          |     |                                  |                                       
          |     |    +----------+                  |                                       
          |     +--> | nmiReset | (skip)           |                                       
          |          +----------+                  |                                       
          +----------------------------------------+                                       
```

```
src/power_control.cpp                                                                                                           
+-------+                                                                                                                        
| part2 | : handle host state                                                                                                    
+-|-----+                                                                                                                        
  |                                                                                                                              
  |--> in object: "/xyz/openbmc_project/state/host" + node                                                                       
  |    add iface: xyz.openbmc_project.State.Host                                                                                 
  |                                                                                                                              
  |--> set prop RequestedHostTransition = xyz.openbmc_project.State.Host.Transition.Off                                          
  |       +---------------------------------------------------------------------------------------------------------------------+
  |       |if requested == xyz.openbmc_project.State.Host.Transition.Off                                                        |
  |       |-                                                                                                                    |
  |       |+--> if power_button isn't mask                                                                                      |
  |       |     |                                                                                                               |
  |       |     |    +-----------------------+                                                                                  |
  |       |     +--> | sendPowerControlEvent | (skip)                                                                           |
  |       |          +-----------------------+                                                                                  |
  |       |                                                                                                                     |
  |       |elif reuqested == xyz.openbmc_project.State.Host.Transition.On                                                       |
  |       |-                                                                                                                    |
  |       |+--> if power_button isn't mask                                                                                      |
  |       |     |                                                                                                               |
  |       |     |    +-----------------------+                                                                                  |
  |       |     +--> | sendPowerControlEvent | given state, call target handler to process event (power_on_request)             |
  |       |          +-----------------------+                                                                                  |
  |       |                                                                                                                     |
  |       |elif reuqested == xyz.openbmc_project.State.Host.Transition.Reboot                                                   |
  |       |-                                                                                                                    |
  |       |+--> if power_button isn't mask                                                                                      |
  |       |     |                                                                                                               |
  |       |     |    +-----------------------+                                                                                  |
  |       |     +--> | sendPowerControlEvent | given state, call target handler to process event (power_cycle_request)          |
  |       |          +-----------------------+                                                                                  |
  |       |                                                                                                                     |
  |       |elif reuqested == xyz.openbmc_project.State.Host.Transition.GracefulWarmReboot                                       |
  |       |-                                                                                                                    |
  |       |+--> if reset_button isn't mask                                                                                      |
  |       |     |                                                                                                               |
  |       |     |    +-----------------------+                                                                                  |
  |       |     +--> | sendPowerControlEvent | given state, call target handler to process event (graceful_power_cycle_request) |
  |       |          +-----------------------+                                                                                  |
  |       |                                                                                                                     |
  |       |elif reuqested == xyz.openbmc_project.State.Host.Transition.ForceWarmReboot                                          |
  |       |-                                                                                                                    |
  |       |+--> if reset_button isn't mask                                                                                      |
  |       |     |                                                                                                               |
  |       |     |    +-----------------------+                                                                                  |
  |       |     +--> | sendPowerControlEvent | given state, call target handler to process event (reset_request)                |
  |       |          +-----------------------+                                                                                  |
  |       +---------------------------------------------------------------------------------------------------------------------+
  |                                                                                                                              
  +--> set property 'CurrentHostState'                                                                                           
```

```
src/power_control.cpp                                                                                                  
+-------+                                                                                                               
| part3 | : handle chassis state                                                                                        
+-|-----+                                                                                                               
  |                                                                                                                     
  |--> in object: "/xyz/openbmc_project/state/chassis" + node                                                           
  |    add iface: xyz.openbmc_project.State.Chassis                                                                     
  |                                                                                                                     
  |--> set property RequestedPowerTransition = xyz.openbmc_project.State.Chassis.Transition.Off                         
  |       +------------------------------------------------------------------------------------------------------------+
  |       |if request == xyz.openbmc_project.State.Chassis.Transition.Off                                              |
  |       |-                                                                                                           |
  |       |+--> if power_button isn't masked                                                                           |
  |       |     |                                                                                                      |
  |       |     |    +-----------------------+                                                                         |
  |       |     +--> | sendPowerControlEvent | given state, call target handler to process event (power_off_request)   |
  |       |          +-----------------------+                                                                         |
  |       |                                                                                                            |
  |       |elif request == xyz.openbmc_project.State.Chassis.Transition.On                                             |
  |       |-                                                                                                           |
  |       |+--> if power_button isn't masked                                                                           |
  |       |     |                                                                                                      |
  |       |     |    +-----------------------+                                                                         |
  |       |     +--> | sendPowerControlEvent | given state, call target handler to process event (power_on_request)    |
  |       |          +-----------------------+                                                                         |
  |       |                                                                                                            |
  |       |elif request == xyz.openbmc_project.State.Chassis.Transition.PowerCycle                                     |
  |       |-                                                                                                           |
  |       |+--> if power_button isn't masked                                                                           |
  |       |     |                                                                                                      |
  |       |     |    +-----------------------+                                                                         |
  |       |     +--> | sendPowerControlEvent | given state, call target handler to process event (power_cycle_request) |
  |       +------------------------------------------------------------------------------------------------------------+
  |                                                                                                                     
  |--> set property 'CurrentPowerState'                                                                                 
  |                                                                                                                     
  +--> set property 'LastStateChangeTime'                                                                               
```

```
src/power_control.cpp                                                                               
+-------+                                                                                            
| part4 | : handle chassis_system state                                                              
+-|-----+                                                                                            
  |                                                                                                  
  |--> in object: /xyz/openbmc_project/state/chassis_system0                                         
  |    add iface: xyz.openbmc_project.State.Chassis                                                  
  |                                                                                                  
  |--> set property 'RequestedPowerTransition' = xyz.openbmc_project.State.Chassis.Transition.On     
  |                                                                                                  
  |        if request == xyz.openbmc_project.State.Chassis.Transition.PowerCycle                     
  |        |                                                                                         
  |        |    +-------------+                                                                      
  |        +--> | systemReset | call systemd to run chassis-system-reset.target                      
  |             +-------------+                                                                      
  |                                                                                                  
  |--> set proerty 'CurrentPowerState' & 'LastStateChangeTime'                                       
  |                                                                                                  
  |--> if slot_power_config line is set                                                              
  |    |                                                                                             
  |    |    +---------------+                                                                        
  |    |--> | setGPIOOutput | set gpio slot_power_line's direction to output (and value)             
  |    |    +---------------+                                                                        
  |    |                                                                                             
  |    +--> determine power_slot_state                                                               
  |        +----------------------------------------------------------------------------------------+
  |        |in object: "/xyz/openbmc_project/state/chassis_system" + node                           |
  |        |add iface: xyz.openbmc_project.State.Chassis                                            |
  |        |                                                                                        |
  |        |set property RequestedPowerTransition = xyz.openbmc_project.State.Chassis.Transition.On |
  |        |                                                                                        |
  |        |    if request == xyz.openbmc_project.State.Chassis.Transition.On                       |
  |        |    |                                                                                   |
  |        |    |    +-------------+                                                                |
  |        |    +--> | slotPowerOn | slot_power on                                                  |
  |        |         +-------------+                                                                |
  |        |                                                                                        |
  |        |    elif request == xyz.openbmc_project.State.Chassis.Transition.Off                    |
  |        |    |                                                                                   |
  |        |    |    +--------------+                                                               |
  |        |    +--> | slotPowerOff | : slot_power off                                              |
  |        |         +--------------+                                                               |
  |        |                                                                                        |
  |        |    elif request == xyz.openbmc_project.State.Chassis.Transition.PowerCycle             |
  |        |    |                                                                                   |
  |        |    |    +----------------+                                                             |
  |        |    +--> | slotPowerCycle | slot_power off and on                                       |
  |        |         +----------------+                                                             |
  |        +----------------------------------------------------------------------------------------+
  |                                                                                                  
  +--> set proerty 'CurrentPowerState' & 'LastStateChangeTime'                                       
```

```
src/power_control.cpp                                                                       
+-------------+                                                                              
| slotPowerOn | : slot_power on                                                              
+-|-----------+                                                                              
  |                                                                                          
  +--> if slot_power_state != on                                                             
       |                                                                                     
       |--> set slot_power_line value = 1                                                    
       |                                                                                     
       +--> if it's set successfully                                                         
            |                                                                                
            |    +-------------------+                                                       
            +--> | setSlotPowerState | set prop 'CurrentPowerState' and 'LastStateChangeTime'
                 +-------------------+                                                       
```

```
src/power_control.cpp                                                                       
+--------------+                                                                             
| slotPowerOff | : slot_power off                                                            
+-|------------+                                                                             
  |                                                                                          
  +--> if slot_power_state != off                                                            
       |                                                                                     
       |--> set slot_power_line value = 0                                                    
       |                                                                                     
       +--> if value is set to 0 successfully                                                
            |                                                                                
            |    +-------------------+                                                       
            |--> | setSlotPowerState | set prop 'CurrentPowerState' and 'LastStateChangeTime'
            |    +-------------------+                                                       
            |    +---------------+                                                           
            +--> | setPowerState |                                                           
                 +---------------+                                                           
```

```
src/power_control.cpp                    
+----------------+                        
| slotPowerCycle | : slot_power off and on
+-|--------------+                        
  |    +--------------+                   
  |--> | slotPowerOff | slot_power off    
  |    +--------------+                   
  |    +-------------+                    
  +--> | slotPowerOn | slot_power on      
       +-------------+                    
```

```
src/power_control.cpp                                                                       
+-------+                                                                                    
| part5 | : handle power_button and reset_button                                             
+-|-----+                                                                                    
  |                                                                                          
  |--> if power_button_config is set                                                         
  |    |                                                                                     
  |    |--> in object: /xyz/openbmc_project/chassis/buttons/power                            
  |    |    add iface: xyz.openbmc_project.Chassis.Buttons                                   
  |    |                                                                                     
  |    |--> set property ButtonMasked = false                                                
  |    |       +----------------------------------------------------------------------------+
  |    |       |if requested                                                                |
  |    |       ||                                                                           |
  |    |       ||    +---------------+                                                      |
  |    |       |+--> | setGPIOOutput | set gpio power_out's direction to output (and value) |
  |    |       |     +---------------+                                                      |
  |    |       |                                                                            |
  |    |       |else                                                                        |
  |    |       ||                                                                           |
  |    |       ||    +-------------+                                                        |
  |    |       |+--> | line::reset |                                                        |
  |    |       |     +-------------+                                                        |
  |    |       +----------------------------------------------------------------------------+
  |    |                                                                                     
  |    |--> determien if power button is pressed                                             
  |    |                                                                                     
  |    +--> set prop 'ButtonPressed'                                                         
  |                                                                                          
  +--> if reset_button_config is set                                                         
       |                                                                                     
       |--> in object: /xyz/openbmc_project/chassis/buttons/reset                            
       |    add iface: xyz.openbmc_project.Chassis.Buttons                                   
       |                                                                                     
       |--> set property ButtonMasked = false                                                
       |       +----------------------------------------------------------------------------+
       |       |if requested                                                                |
       |       ||                                                                           |
       |       ||    +---------------+                                                      |
       |       |+--> | setGPIOOutput | set gpio reset_out's direction to output (and value) |
       |       |     +---------------+                                                      |
       |       |                                                                            |
       |       |else                                                                        |
       |       ||                                                                           |
       |       ||    +-------------+                                                        |
       |       |+--> | line::reset |                                                        |
       |       |     +-------------+                                                        |
       |       +----------------------------------------------------------------------------+
       |                                                                                     
       |--> determien if reset button is pressed                                             
       |                                                                                     
       +--> set prop 'ButtonPressed'                                                         
```

```
src/power_control.cpp                                                    
+-------+                                                                 
| part6 | : handle nmi_button, nmi_out, id_button, and post_complete      
+-|-----+                                                                 
  |                                                                       
  |--> if nmi_button_line is set                                          
  |    |                                                                  
  |    |--> in object: /xyz/openbmc_project/chassis/buttons/nmi           
  |    |    add iface: xyz.openbmc_project.Chassis.Buttons                
  |    |                                                                  
  |    |--> set property ButtonMasked = false                             
  |    |       +---------------------------------------------+            
  |    |       |given 'requested', determine nmiButtonMasked |            
  |    |       +---------------------------------------------+            
  |    |                                                                  
  |    |--> determine if nmi_button is pressed                            
  |    |                                                                  
  |    +--> set property 'ButtonPressed'                                  
  |                                                                       
  |--> if nmi_out_line is set                                             
  |    |                                                                  
  |    |--> in object: "/xyz/openbmc_project/control/host" + node + "/nmi"
  |    |    add iface: xyz.openbmc_project.Control.Host.NMI               
  |    |                          +----------+                            
  |    +--> register method NMI = | nmiReset | nmi reset                  
  |                               +----------+                            
  |                                                                       
  |--> if id_button_line i set                                            
  |    |                                                                  
  |    |--> in object: /xyz/openbmc_project/chassis/buttons/id            
  |    |    add iface: xyz.openbmc_project.Chassis.Buttons                
  |    |                                                                  
  |    |--> check if id_button is pressed                                 
  |    |                                                                  
  |    +--> set prop 'ButtonPressed'                                      
  |                                                                       
  |--> in object: /xyz/openbmc_project/state/os                           
  |    add iface: xyz.openbmc_project.State.OperatingSystem.Status        
  |                                                                       
  |--> get gpio value to know os state (is post_complete happened?)       
  |                                                                       
  +--> set prop 'OperatingSystemState'                                    
```

```
src/power_control.cpp                                                                  
+----------+                                                                            
| nmiReset | : nmi reset                                                                
+-|--------+                                                                            
  |                                                                                     
  |--> nmi_out_line.set_value()                                                         
  |                                                                                     
  |--> prepare timer                                                                    
  |       +-------------------------+                                                   
  |       |nmi_out_line.set_value() |                                                   
  |       +-------------------------+                                                   
  |                                                                                     
  |    +---------------+                                                                
  |--> | nmiDiagIntLog | log to redfish                                                 
  |    +---------------+                                                                
  |    +----------------------+                                                         
  +--> | nmiSetEnableProperty | send method call to service xyz.openbmc_project.Settings
       +----------------------+                                                         
```

```
src/power_control.cpp                                                                       
+-------+                                                                                    
| part7 | : handle restart_cause, monitor host state                                         
+-|-----+                                                                                    
  |                                                                                          
  |--> in object: "/xyz/openbmc_project/control/host" + node + "/restart_cause"              
  |    add iface: xyz.openbmc_project.Control.Host.RestartCause                              
  |                                                                                          
  |--> set prop 'RestartCause' = xyz.openbmc_project.State.Host.RestartCause.Unknown         
  |                                                                                          
  |--> set prop 'RequestedRestartCause' = xyz.openbmc_project.State.Host.RestartCause.Unknown
  |                                                                                          
  |--> if request == xyz.openbmc_project.State.Host.RestartCause.WatchdogTimer               
  |    |                                                                                     
  |    |    +-----------------+                                                              
  |    +--> | addRestartCause |                                                              
  |         +-----------------+                                                              
  |    +-------------------------+                                                           
  +--> | currentHostStateMonitor | prepare match rule and callback to monitor host state     
       +-------------------------+                                                           
```

```
src/power_control.cpp                                                             
+-------------------------+                                                        
| currentHostStateMonitor | : prepare match rule and callback to monitor host state
+-|-----------------------+                                                        
  |                                                                                
  |--> give host state, handle poh_counter_timer and causes                        
  |                                                                                
  +--> prepare match rule and callback                                             
          +-----------------------------------------------------+                  
          |read msg                                             |                  
          |                                                     |                  
          |give host state, handle poh_counter_timer and causes |                  
          +-----------------------------------------------------+                  
```





```
src/power_control.cpp                                                                                                                       
+--------------+                                                                                                                             
| powerStateOn | : given event (e.g., graceful_power_off, power_cycle), perform the actions                                                  
+-|------------+                                                                                                                             
  |                                                                                                                                          
  |--> switch event                                                                                                                          
  |    case power_ok_deassert                                                                                                                
  |    -    +------+                                                                                                                         
  |    +--> | beep | send method call to service "xyz.openbmc_project.BeepCode"                                                              
  |         +------+                                                                                                                         
  +--> case sio_s5_assert                                                                                                                    
  |    -    +-----------------+                                                                                                              
  |    +--> | addRestartCause | add 'soft_reset' to causes          almost all cases call this                                               
  |         +-----------------+                                     +---------------+                                                        
  |--> case platform_reset_assert or post_complete_deassert         | setPowerState | given state, update global var & dbus properties & file
  |    |    +-----------------+                                     +---------------+                                                        
  |    |--> | addRestartCause | add 'soft_reset' to causes                                                                                   
  |    |    +-----------------+                                                                                                              
  |    |    +--------------------------+                                                                                                     
  |    +--> | warmResetCheckTimerStart | given state, call the corresponding handler to process event 'warm_reset_detected'                  
  |         +--------------------------+                                                                                                     
  |--> case power_button_pressed                                                                                                             
  |    -    +----------------------------+                                                                                                   
  |    +--> | gracefulPowerOffTimerStart | given state, call the corresponding handler to process event 'graceful_power_off_timer_expired'   
  |         +----------------------------+                                                                                                   
  |--> case power_off_request                                                                                                                
  |    -    +---------------+                                                                                                                
  |    +--> | forcePowerOff | force power off                                                                                                
  |         +---------------+                                                                                                                
  |--> case graceful_power_off_request                                                                                                       
  |    |    +----------------------------+                                                                                                   
  |    |--> | gracefulPowerOffTimerStart | given state, call the corresponding handler to process event 'graceful_power_off_timer_expired'   
  |    |    +----------------------------+                                                                                                   
  |    |    +------------------+                                                                                                             
  |    +--> | gracefulPowerOff | graceful power off                                                                                          
  |         +------------------+                                                                                                             
  |--> case power_cycle_request                                                                                                              
  |    -    +---------------+                                                                                                                
  |    +--> | forcePowerOff | force power off                                                                                                
  |         +---------------+                                                                                                                
  |--> case graceful_power_cycle_request                                                                                                     
  |    -    +----------------------------+                                                                                                   
  |    +--> | gracefulPowerOffTimerStart | given state, call the corresponding handler to process event 'graceful_power_off_timer_expired'   
  |         +----------------------------+                                                                                                   
  +--> case reset_request                                                                                                                    
       -    +-------+                                                                                                                        
       +--> | reset | assert gpio to reset                                                                                                   
            +-------+                                                                                                                        
```

```
src/power_control.cpp                                                                    
+---------------+                                                                         
| setPowerState | : given state, update global var & dbus properties & file               
+-|-------------+                                                                         
  |                                                                                       
  |--> update state in global var                                                         
  |                                                                                       
  |    +--------------------+                                                             
  |--> | logStateTransition | log                                                         
  |    +--------------------+                                                             
  |                                                                                       
  |--> set dbus property "CurrentHostState" = running or off                              
  |                                                                                       
  |--> set dbus property "CurrentPowerState" = on or off                                  
  |                                                                                       
  |--> set dbus property "LastStateChangeTime"                                            
  |                                                                                       
  |    +----------------+                                                                 
  +--> | savePowerState | update stateData and save to "/var/lib/power-control/state.json"
       +----------------+                                                                 
```

```
src/power_control.cpp                                                                                
+----------------+                                                                                    
| savePowerState | : update stateData and save to "/var/lib/power-control/state.json"                 
+-|--------------+                                                                                    
  |                                                                                                   
  |--> set timer expiration                                                                           
  |                                                                                                   
  +--> .async_wait                                                                                    
          +------------------------------------------------------------------------------------------+
          |+----------------------+                                                                  |
          || PersistentState::set | update stateData and save to "/var/lib/power-control/state.json" |
          |+----------------------+                                                                  |
          +------------------------------------------------------------------------------------------+
```

```
src/power_control.cpp                                                                                   
+----------------------+                                                                                 
| PersistentState::set | : update stateData and save to "/var/lib/power-control/state.json"              
+-|--------------------+                                                                                 
  |                                                                                                      
  |--> stateData[key] = value                                                                            
  |                                                                                                      
  |    +----------------------------+                                                                    
  +--> | PersistentState::saveState | save stateData (json struct) to "/var/lib/power-control/state.json"
       +----------------------------+                                                                    
```

```
src/power_control.cpp                                                                              
+----------------------------+                                                                      
| PersistentState::saveState | : save stateData (json struct) to "/var/lib/power-control/state.json"
+-|--------------------------+                                                                      
  |                                                                                                 
  |--> file = "/var/lib/power-control/state.json"                                                   
  |                                                                                                 
  +--> dump stateData to file                                                                       
```

```
src/power_control.cpp                                                                                           
+--------------------------+                                                                                     
| warmResetCheckTimerStart | : given state, call the corresponding handler to process event 'warm_reset_detected'
+-|------------------------+                                                                                     
  |                                                                                                              
  |--> set timer expiration                                                                                      
  |                                                                                                              
  +--> .async_wait                                                                                               
          +---------------------------------------------------------------------------------------+              
          |+-----------------------+                                                              |              
          || sendPowerControlEvent | given state, call the corresponding handler to process event |              
          |+-----------------------+                                                              |              
          +---------------------------------------------------------------------------------------+              
```

```
src/power_control.cpp                                                                                                          
+----------------------------+                                                                                                  
| gracefulPowerOffTimerStart | : given state, call the corresponding handler to process event 'graceful_power_off_timer_expired'
+-|--------------------------+                                                                                                  
  |                                                                                                                             
  |--> set timer expiration                                                                                                     
  |                                                                                                                             
  +--> .async_wait                                                                                                              
          +---------------------------------------------------------------------------------------+                             
          |+-----------------------+                                                              |                             
          || sendPowerControlEvent | given state, call the corresponding handler to process event |                             
          |+-----------------------+                                                              |                             
          +---------------------------------------------------------------------------------------+                             
```

```
src/power_control.cpp                    
+---------------+                         
| forcePowerOff | : force power off       
+-|-------------+                         
  |    +-----------------+                
  +--> | assertGPIOForMs | set gpio output
       +-----------------+                
```

```
src/power_control.cpp                                               
+-----------------+                                                  
| assertGPIOForMs | : set gpio output                                
+-|---------------+                                                  
  |    +--------------------+                                        
  +--> | setGPIOOutputForMs | : set gpio output                      
       +-|------------------+                                        
         |                                                           
         |--> if power_button is masked                              
         |    |                                                      
         |    |    +--------------------------+                      
         |    +--> | setMaskedGPIOOutputForMs | gpio_line.set_value()
         |         +--------------------------+                      
         |                                                           
         |--> if reset_button is masked                              
         |    |                                                      
         |    |    +--------------------------+                      
         |    +--> | setMaskedGPIOOutputForMs | gpio_line.set_value()
         |         +--------------------------+                      
         |    +---------------+                                      
         +--> | setGPIOOutput | set gpio output                      
              +---------------+                                      
```