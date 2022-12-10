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
