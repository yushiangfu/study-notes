- server

```
 console-server.c                                                                           
 [main] : setup console, parse config, init routing, open tty, register dbus, endlessly poll
 |                                                                                          
 |--> parse arguments                                                                       
 |                                                                                          
 |--> alloc 'console'                                                                       
 |                                                                                          
 |--> [config_init] parse config file, e.g., /etc/obmc-console/server.ttyS2.conf            
 |                                                                                          
 |--> [set_socket_info] determine socket name                                               
 |                                                                                          
 |--> [uart_routing_init] handle 'uart_routing_init' in config file                         
 |                                                                                          
 |--> [tty_init] given tty name, open and setup                                             
 |                                                                                          
 |--> [dbus_init] setup dbus obj                                                            
 |                                                                                          
 |--> [handlers_init] call console handlers (probably none)                                 
 |                                                                                          
 |--> [run_console] endlessly poll (for events from dev, dbus, and pollers)                 
 |                                                                                          
 +--> (skip the remaining tear-down functions)                                              
```

```
 console-server.c                                                        
 [tty_init] : given tty name, open and setup                             
 |                                                                       
 |--> [tty_find_device] given tty name, further setup dev path in console
 |                                                                       
 |--> switch tty_type                                                    
 |    case vuart                                                         
 |    ---> (skip, probably not our case)                                 
 |    case uart                                                          
 |    ---> [config_get_value] get baud rate from config                  
 |                                                                       
 +--> [tty_init_io] open tty and init termios                            
```

```
 console-server.c                                    
 [dbus_init] : setup dbus obj                        
 |                                                   
 |--> [sd_bus_default_system] get default bus        
 |                                                   
 |--> register obj with uart interface               
 |                                                   
 |--> register obj with access interface             
 |                                                   
 |--> [sd_bus_request_name]                          
 |                                                   
 +--> save dbus fd in console (for later poll)       
                                                     
                                                     
 [service] xyz.openbmc_project.Console.default       
                                                     
      [obj] /xyz/openbmc_project/console/default     
                                                     
           [iface] xyz.openbmc_project.Console.UART  
                                                     
           [iface] xyz.openbmc_project.Console.Access
```

```
 console-server.c                                                                            
 [run_console] : endlessly poll (for events from dev, dbus, and pollers)                     
 -                                                                                           
 +--> endless loop                                                                           
      |                                                                                      
      |--> [get_poll_timeout] determine timeout                                              
      |                                                                                      
      |--> [poll]                                                                            
      |                                                                                      
      |--> if there's revent of internal fd                                                  
      |    |                                                                                 
      |    |--> [read]                                                                       
      |    |                                                                                 
      |    +--> [ringbuffer_queue] copy data to ring buffer and call each consumer to take it
      |                                                                                      
      |--> if there's revent of other fd                                                     
      |    -                                                                                 
      |    +--> [sd_bus_process] given bus state, handle it accordingly                      
      |                                                                                      
      +--> [call_pollers] call pollers' callbacks                                            
```

```
 ringbuffer.c                                                                   
 [ringbuffer_queue] : copy data to ring buffer and call each consumer to take it
 |                                                                              
 |--> ensure each consumer has enough space                                     
 |                                                                              
 |--> copy data to ring buffer                                                  
 |                                                                              
 +--> for each consumer                                                         
      -                                                                         
      +--> call ->poll_fn()                                                     
```

```
 console-server.c                                                              
 [call_pollers] : call pollers' callbacks                                      
 |                                                                             
 |--> for each console poller                                                  
 |                                                                             
 |--> if there's event from poller                                             
 |    |                                                                        
 |    |--> call ->event_fn()                                                   
 |    |                                                                        
 |    +--> if poller is removed, label 'removed'                               
 |                                                                             
 +--> endless loop                                                             
      |                                                                        
      |--> if it's labled 'removed'                                            
      |    -                                                                   
      |    +--> [console_poller_unregister] remove poller and its fd from array
      |                                                                        
      +--> exit loop if appropriate
```

```
 console-server.c                                                 
 [console_poller_unregister] : remove poller and its fd from array
 |                                                                
 |--> poller number--                                             
 |                                                                
 |--> remove poller from array (by memmove)                       
 |                                                                
 +--> remove poller fd from array (by memmove)                    
```

- client
