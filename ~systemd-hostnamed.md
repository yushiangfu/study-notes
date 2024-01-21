```
src/hostname/hostnamed.c                                                  
+-----+                                                                    
| run | : register vtable, request "org.freedesktop.hostname1"             
+-|---+                                                                    
  |    +--------------------+                                              
  |--> | service_parse_argv | handle options                               
  |    +--------------------+                                              
  |    +------------------+                                                
  |--> | sd_event_default | get default event                              
  |    +------------------+                                                
  |    +-----------------------+                                           
  |--> | sd_event_set_watchdog | arm watchdog of event                     
  |    +-----------------------+                                           
  |    +---------------------+                                             
  |--> | sd_event_add_signal | add signal source (SIGINT) for event        
  |    +---------------------+                                             
  |    +---------------------+                                             
  |--> | sd_event_add_signal | add signal source (SIGTERM) for event       
  |    +---------------------+                                             
  |    +-------------+                                                     
  |--> | connect_bus | register vtable, request "org.freedesktop.hostname1"
  |    +-------------+                                                     
  |    +--------------------------+                                        
  +--> | bus_event_loop_with_idle |                                        
       +--------------------------+                                        
```

```
src/hostname/hostnamed.c                                             
+-------------+                                                       
| connect_bus | : register vtable, request "org.freedesktop.hostname1"
+-|-----------+                                                       
  |    +-----------------------+                                      
  |--> | sd_bus_default_system | get default bus                      
  |    +-----------------------+                                      
  |    +------------------------+                                     
  |--> | bus_add_implementation | register vtable                     
  |    +------------------------+                                     
  |    +---------------------------+                                  
  |--> | sd_bus_request_name_async | "org.freedesktop.hostname1"      
  |    +---------------------------+                                  
  |    +---------------------+                                        
  +--> | sd_bus_attach_event | prepare all kinds of sources for event 
       +---------------------+                                        
```

```
src/hostname/hostnamed.c                                                          
+---------------------+                                                            
| method_set_hostname | : determine and update hostname                            
+-|-------------------+                                                            
  |    +---------------------+                                                     
  |--> | sd_bus_message_read | get hostname from msg                               
  |    +---------------------+                                                     
  |                                                                                
  |--> read '/etc/hostname' and verify it                                          
  |                                                                                
  |    +--------------------------------+                                          
  +--> | context_update_kernel_hostname | update hostname from /etc/hostname or msg
       +--------------------------------+                                          
```
