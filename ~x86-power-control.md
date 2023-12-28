> Study case: Aspeed OpenBMC (commit 742fec782ef6c34c9fcd866116631e1d7aeedf8c)

## Index

- [Introduction](#introduction)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

<p align="center"><img src="images/x86-power-control/gpio.png" /></p>

```
service: "xyz.openbmc_project.State.Host"                 <-- obsolete
service: "xyz.openbmc_project.State.Chassis"              <-- obsolete
service: "xyz.openbmc_project.State.OperatingSystem"      <-- obsolete
service: "xyz.openbmc_project.Chassis.Buttons"            <-- obsolete
service: "xyz.openbmc_project.Control.Host.RestartCause"  <-- obsolete

service: "xyz.openbmc_project.State.Host0"
service: "xyz.openbmc_project.State.Chassis0"
service: "xyz.openbmc_project.State.OperatingSystem0"
service: "xyz.openbmc_project.Chassis.Buttons0"
service: "xyz.openbmc_project.Control.Host.RestartCause0"

obj: "/xyz/openbmc_project/state/host0"
    iface: "xyz.openbmc_project.State.Host"
        prop: "RequestedHostTransition"  <-- Manage host power status.
        prop: "CurrentHostState"         <-- Retrieve current host power status.

obj: "/xyz/openbmc_project/state/chassis0"
    iface: "xyz.openbmc_project.State.Chassis"
        prop: "RequestedPowerTransition"  <-- Manage chassis power status.
        prop: "CurrentPowerState"         <-- Retrieve current host power status.
        prop: "LastStateChangeTime"       <-- timestamp

obj: "/xyz/openbmc_project/chassis/buttons/power"
    iface: "xyz.openbmc_project.Chassis.Buttons"
        prop: "ButtonMasked"   <-- Deassert the 'power out' signal.
        prop: "ButtonPressed"  <-- Verify the current status of the power button.

obj: "/xyz/openbmc_project/chassis/buttons/reset"
    iface: "xyz.openbmc_project.Chassis.Buttons"
        prop: "ButtonMasked"   <-- Enable or disable the power button.
        prop: "ButtonPressed"  <-- Verify the current status of the reset button.

obj: "/xyz/openbmc_project/chassis/buttons/id"
    iface: "xyz.openbmc_project.Chassis.Buttons"
        prop: "ButtonPressed"  <-- Verify the current status of the ID button.

obj: "/xyz/openbmc_project/state/os"
    iface: "xyz.openbmc_project.State.OperatingSystem.Status"
        prop: "OperatingSystemState"  <-- Verify the current status of 'post complete.'

obj: "/xyz/openbmc_project/control/host0/restart_cause"
    iface: "xyz.openbmc_project.Control.Host.RestartCause"
        prop: "RestartCause"           <-- Retrieve the host's restart cause.
        prop: "RequestedRestartCause"  <-- Add a restart cause related to the watchdog.
```

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

### power state handler

```
src/power_control.cpp                                                                                                                                   
+----------------------+                                                                                                                                 
| getPowerStateHandler | : given power state, return state handler                                                                                       
+-|--------------------+                                                                                                                                 
  |                                                                                                                                                      
  |--> switch power_state                                                                                                                                
  |    case on                                                                                                                                           
  |    -           +--------------+                                                                                                                      
  |    +--> return | powerStateOn | given event (e.g., graceful_power_off, power_cycle), perform the actions                                             
  |                +--------------+                                                                                                                      
  |--> case wait_for_ps_power_ok                                                                                                                         
  |    -           +----------------------------+                                                                                                        
  |    +--> return | powerStateWaitForPSPowerOK | handle events (ps_power_assert/ps_power_ok_watchdog_timer_expired/sio_power_good_assert)               
  |                +----------------------------+                                                                                                        
  |--> case wait_for_sio_power_good                                                                                                                      
  |    -           +-------------------------------+                                                                                                     
  |    +--> return | powerStateWaitForSIOPowerGood | handle events (sio_power_good_assert/sio_power_good_watchdog_timer_expired)                         
  |                +-------------------------------+                                                                                                     
  |--> case off                                                                                                                                          
  |    -           +---------------+                                                                                                                     
  |    +--> return | powerStateOff | handle events (ps_power_ok, sio_s5, sio_power_good, power_button_pressed, power_on_request)                         
  |                +---------------+                                                                                                                     
  |--> case transition_to_off                                                                                                                            
  |    -           +---------------------------+                                                                                                         
  |    +--> return | powerStateTransitionToOff | handle event (ps_power_ok_deassert)                                                                     
  |                +---------------------------+                                                                                                         
  |--> case graceful_transition_to_off                                                                                                                   
  |    -           +-----------------------------------+                                                                                                 
  |    +--> return | powerStateGracefulTransitionToOff | handle events (ps_poewr_ok, graceful_timer_expired, power_off_req, power_cycle_req, reset_req)  
  |                +-----------------------------------+                                                                                                 
  |--> case cycle_off                                                                                                                                    
  |    -           +--------------------+                                                                                                                
  |    +--> return | powerStateCycleOff | handle events (ps_power_ok, sio_s5_deassert, power_button_pressed, power_cycle_timer_expired)                  
  |                +--------------------+                                                                                                                
  |--> case transition_to_cycle_off                                                                                                                      
  |    -           +--------------------------------+                                                                                                    
  |    +--> return | powerStateTransitionToCycleOff | handle event (ps_power_ok_deassert)                                                                
  |                +--------------------------------+                                                                                                    
  |--> case graceful_transition_to_cycle_off                                                                                                             
  |    -           +----------------------------------------+                                                                                            
  |    +--> return | powerStateGracefulTransitionToCycleOff | handle event (ps_power_ok, graceful_power_off_timer_expired, power_off, power_cycle, reset)
  |                +----------------------------------------+                                                                                            
  +--> case check_for_warm_reset                                                                                                                         
       -           +-----------------------------+                                                                                                       
       +--> return | powerStateCheckForWarmReset | handle events (sio_s5, warm_reset_detected, ps_power_ok)                                              
                   +-----------------------------+                                                                                                       
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

```
src/power_control.cpp                                                                                                  
+----------------------------+                                                                                          
| powerStateWaitForPSPowerOK | : handle event (ps_power_assert/ps_power_ok_watchdog_timer_expired/sio_power_good_assert)
+-|--------------------------+                                                                                          
  |                                                                                                                     
  |--> switch event                                                                                                     
  |--> case ps_power_ok_assert                                                                                          
  |    |                                                                                                                
  |    |--> cancel gpio and watchdog timers                                                                             
  |    |                                                                                                                
  |    |--> if sio is enabled                                                                                           
  |    |    |                                                                                                           
  |    |    |    +--------------------------------+                                                                     
  |    |    |--> | sioPowerGoodWatchdogTimerStart | set a timer to handle event of sio_power_good_watchdog_timr_expired 
  |    |    |    +--------------------------------+                                                                     
  |    |    |    +---------------+                                                                                      
  |    |    +--> | setPowerState | state = wait_for_sio_power_good                                                      
  |    |         +---------------+                                                                                      
  |    |                                                                                                                
  |    +--> else                                                                                                        
  |         -    +---------------+                                                                                      
  |         +--> | setPowerState | state = on                                                                           
  |              +---------------+                                                                                      
  |                                                                                                                     
  |--> case ps_power_ok_watchdog_timer_expired                                                                          
  |    |                                                                                                                
  |    |    +---------------+                                                                                           
  |    |--> | setPowerState | state = off                                                                               
  |    |    +---------------+                                                                                           
  |    |    +--------------------+                                                                                      
  |    +--> | psPowerOKFailedLog | log                                                                                  
  |         +--------------------+                                                                                      
  |                                                                                                                     
  +--> case sio_power_good_assert                                                                                       
       |                                                                                                                
       |--> cancel watchdog timer                                                                                       
       |                                                                                                                
       |    +---------------+                                                                                           
       +--> | setPowerState | state = on                                                                                
            +---------------+                                                                                           
```

```
src/power_control.cpp                                                                                         
+-------------------------------+                                                                              
| powerStateWaitForSIOPowerGood | : handle events (sio_power_good_assert/sio_power_good_watchdog_timer_expired)
+-|-----------------------------+                                                                              
  |                                                                                                            
  |--> switch event                                                                                            
  |--> case sio_power_good_assert                                                                              
  |    |                                                                                                       
  |    |--> cancel watchdog timer                                                                              
  |    |                                                                                                       
  |    |    +---------------+                                                                                  
  |    +--> | setPowerState | state = on                                                                       
  |         +---------------+                                                                                  
  |                                                                                                            
  +--> case sio_power_good_watchdog_timer_expired                                                              
       |                                                                                                       
       |    +---------------+                                                                                  
       |--> | setPowerState | state = off                                                                      
       |    +---------------+                                                                                  
       |    +--------------------------+                                                                       
       +--> | systemPowerGoodFailedLog | log                                                                   
            +--------------------------+                                                                       
```

```
src/power_control.cpp                                                                                                 
+---------------+                                                                                                      
| powerStateOff | : handle events (ps_power_ok, sio_s5, sio_power_good, power_button_pressed, power_on_request)        
+-|-------------+                                                                                                      
  |                                                                                                                    
  |--> switch event                                                                                                    
  |    case ps_power_ok_assert                                                                                         
  |    |                                                                                                               
  |    |--> if sio is enabled                                                                                          
  |    |    -    +--------------------------------+                                                                    
  |    |    |--> | sioPowerGoodWatchdogTimerStart | set a timer to handle event of sio_power_good_watchdog_timr_expired
  |    |    |    +--------------------------------+                                                                    
  |    |    |    +---------------+                                                                                     
  |    |    +--> | setPowerState | state = wait_for_sio_power_good                                                     
  |    |         +---------------+                                                                                     
  |    +--> else                                                                                                       
  |         -    +---------------+                                                                                     
  |         +--->| setPowerState | state = off                                                                         
  |              +---------------+                                                                                     
  |--> case sio_s5_deassert                                                                                            
  |    |    +-----------------------------+                                                                            
  |    |--> | psPowerOKWatchdogTimerStart |  set a timer to handle event (ps_power_ok_watchdog_timer_expired)          
  |    |    +-----------------------------+                                                                            
  |    |    +---------------+                                                                                          
  |    +--->| setPowerState | state = wait_for_ps_power_ok                                                             
  |         +---------------+                                                                                          
  |--> case sio_power_good_assert                                                                                      
  |    -    +---------------+                                                                                          
  |    +--->| setPowerState | state = on                                                                               
  |         +---------------+                                                                                          
  |--> case power_button_pressed                                                                                       
  |    -                                                                                                               
  |    +--> (same as case sio_s5_deassert)                                                                             
  |                                                                                                                    
  +--> case power_on_request                                                                                           
       |                                                                                                               
       |--> (same as case sio_s5_deassert)                                                                             
       |    +---------+                                                                                                
       +--> | powerOn | assert gpio to power on                                                                        
            +---------+
```

```
src/power_control.cpp                                                                                                                
+-----------------------------------+                                                                                                 
| powerStateGracefulTransitionToOff | : handle events (ps_poewr_ok, graceful_timer_expired, power_off_req, power_cycle_req, reset_req)
+-|---------------------------------+                                                                                                 
  |                                                                                                                                   
  |--> case ps_power_ok_deassert                                                                                                      
  |    |                                                                                                                              
  |    |--> cancel timer                                                                                                              
  |    |    +---------------+                                                                                                         
  |    +--> | setPowerState | state = off                                                                                             
  |         +---------------+                                                                                                         
  |--> case graceful_power_off_timer_expired                                                                                          
  |    -    +---------------+                                                                                                         
  |    +--> | setPowerState | state = on                                                                                              
  |         +---------------+                                                                                                         
  |--> case power_off_request                                                                                                         
  |    |                                                                                                                              
  |    |--> cancel timer                                                                                                              
  |    |    +---------------+                                                                                                         
  |    |--> | setPowerState | state = transition_to_off                                                                               
  |    |    +---------------+                                                                                                         
  |    |    +---------------+                                                                                                         
  |    +--> | forcePowerOff | force power off                                                                                         
  |         +---------------+                                                                                                         
  |--> case power_cycle_request                                                                                                       
  |    |                                                                                                                              
  |    |--> cancel timer                                                                                                              
  |    |    +---------------+                                                                                                         
  |    |--> | setPowerState | state = transition_to_cycle_off                                                                         
  |    |    +---------------+                                                                                                         
  |    |    +---------------+                                                                                                         
  |    +--> | forcePowerOff | force power off                                                                                         
  |         +---------------+                                                                                                         
  +--> case reset_request                                                                                                             
       |                                                                                                                              
       |--> cancel timer                                                                                                              
       |    +---------------+                                                                                                         
       |--> | setPowerState | state = on                                                                                              
       |    +---------------+                                                                                                         
       |    +-------+                                                                                                                 
       +--> | reset | assert gpio to reset                                                                                            
            +-------+                                                                                                                 
```

```
src/power_control.cpp                                                                                                   
+--------------------+                                                                                                   
| powerStateCycleOff | : handle events (ps_power_ok, sio_s5_deassert, power_button_pressed, power_cycle_timer_expired)   
+-|------------------+                                                                                                   
  |                                                                                                                      
  |--> switch event                                                                                                      
  |--> case ps_power_ok_assert                                                                                           
  |    |                                                                                                                 
  |    |--> cancel timer                                                                                                 
  |    +--> if sio is enabled                                                                                            
  |    |    -    +--------------------------------+                                                                      
  |    |    |--> | sioPowerGoodWatchdogTimerStart | prepare timer to handle event (sio_power_good_watchdog_timer_expired)
  |    |    |    +--------------------------------+                                                                      
  |    |    |    +---------------+                                                                                       
  |    |    +--> | setPowerState | state = wait_for_sio_power_good                                                       
  |    |         +---------------+                                                                                       
  |    +--> else                                                                                                         
  |         -    +---------------+                                                                                       
  |         +--> | setPowerState | state = on                                                                            
  |              +---------------+                                                                                       
  |--> case sio_s5_deassert                                                                                              
  |    |                                                                                                                 
  |    |--> cancel timer                                                                                                 
  |    |    +-----------------------------+                                                                              
  |    |--> | psPowerOKWatchdogTimerStart | set a timer to handle event (ps_power_ok_watchdog_timer_expired)             
  |    |    +-----------------------------+                                                                              
  |    |    +---------------+                                                                                            
  |    +--> | setPowerState | state = wait_for_ps_power_ok                                                               
  |         +---------------+                                                                                            
  |--> case power_button_pressed                                                                                         
  |    |                                                                                                                 
  |    |--> cancel timer                                                                                                 
  |    |    +-----------------------------+                                                                              
  |    |--> | psPowerOKWatchdogTimerStart | set a timer to handle event (ps_power_ok_watchdog_timer_expired)             
  |    |    +-----------------------------+                                                                              
  |    |    +---------------+                                                                                            
  |    +--> | setPowerState | state = wait_for_ps_power_ok                                                               
  |         +---------------+                                                                                            
  +--> case power_cycle_timer_expired                                                                                    
       |    +-----------------------------+                                                                              
       |--> | psPowerOKWatchdogTimerStart | set a timer to handle event (ps_power_ok_watchdog_timer_expired)             
       |    +-----------------------------+                                                                              
       |    +---------------+                                                                                            
       |--> | setPowerState | state = wait_for_ps_power_ok                                                               
       |    +---------------+                                                                                            
       |    +---------+                                                                                                  
       +--> | powerOn | assert gpio to power on                                                                          
            +---------+                                                                                                  
```

```
src/power_control.cpp                                                                                                                  
+----------------------------------------+                                                                                              
| powerStateGracefulTransitionToCycleOff | : handle event (ps_power_ok, graceful_power_off_timer_expired, power_off, power_cycle, reset)
+-|--------------------------------------+                                                                                              
  |                                                                                                                                     
  |--> switch event                                                                                                                     
  |--> case ps_power_ok_deassert                                                                                                        
  |    |                                                                                                                                
  |    |--> cancel timer                                                                                                                
  |    |    +---------------+                                                                                                           
  |    |--> | setPowerState | state = cycle_off                                                                                         
  |    |    +---------------+                                                                                                           
  |    |    +----------------------+                                                                                                    
  |    +--> | powerCycleTimerStart | prepare timer to handle event (power_cycle_timer_expired)                                          
  |         +----------------------+                                                                                                    
  |--> case graceful_power_off_timer_expired                                                                                            
  |    -    +---------------+                                                                                                           
  |    +--> | setPowerState | state = on                                                                                                
  |         +---------------+                                                                                                           
  |--> case power_off_request                                                                                                           
  |    |                                                                                                                                
  |    |--> cancel timer                                                                                                                
  |    |    +---------------+                                                                                                           
  |    |--> | setPowerState | state = transition_to_off                                                                                 
  |    |    +---------------+                                                                                                           
  |    |    +---------------+                                                                                                           
  |    +--> | forcePowerOff | force power off                                                                                           
  |         +---------------+                                                                                                           
  |--> case power_cycle_request                                                                                                         
  |    |                                                                                                                                
  |    |--> cancel timer                                                                                                                
  |    |    +---------------+                                                                                                           
  |    |--> | setPowerState | state = transition_to_cycle_off                                                                           
  |    |    +---------------+                                                                                                           
  |    |    +---------------+                                                                                                           
  |    +--> | forcePowerOff | force power off                                                                                           
  |         +---------------+                                                                                                           
  +--> case reset_request                                                                                                               
       |                                                                                                                                
       |--> cancel timer                                                                                                                
       |    +---------------+                                                                                                           
       |--> | setPowerState | state = on                                                                                                
       |    +---------------+                                                                                                           
       |    +---------------+                                                                                                           
       |--> | forcePowerOff | force power off                                                                                           
       |    +---------------+                                                                                                           
       |    +-------+                                                                                                                   
       +--> | reset | assert gpio to reset                                                                                              
            +-------+                                                                                                                   
```

```
src/power_control.cpp                                                                    
+-----------------------------+                                                           
| powerStateCheckForWarmReset | : handle events (sio_s5, warm_reset_detected, ps_power_ok)
+-|---------------------------+                                                           
  |                                                                                       
  |--> swtich event                                                                       
  |--> case sio_s5_assert                                                                 
  |    |                                                                                  
  |    |--> cancel timer                                                                  
  |    |    +---------------+                                                             
  |    +--> | setPowerState | state = transition_to_off                                   
  |         +---------------+                                                             
  |--> case warm_reset_detected                                                           
  |    -    +---------------+                                                             
  |    +--> | setPowerState | state = on                                                  
  |         +---------------+                                                             
  +--> case ps_power_ok_deassert                                                          
       |                                                                                  
       |--> cancel timer                                                                  
       |    +---------------+                                                             
       |--> | setPowerState | state = off                                                 
       |    +---------------+                                                             
       |    +------+                                                                      
       +--> | beep | send method call to service "xyz.openbmc_project.BeepCode"           
            +------+                                                                      
```
