```
src/timedate/timedated.c                                                                     
+-----+                                                                                       
| run |                                                                                       
+|----+                                                                                       
 |    +--------------------+                                                                  
 |--> | service_parse_argv | parse options                                                    
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
 |--> | connect_bus | register vtable, request "org.freedesktop.timedate1"                    
 |    +-------------+                                                                         
 |    +-------------------+                                                                   
 |--> | context_read_data | get timezone (UTC) and save in context                            
 |    +-------------------+                                                                   
 |    +----------------------------+                                                          
 |--> | context_parse_ntp_services | find /usr/lib/systemd/ntp-units.d/*.list, save in context
 |    +----------------------------+                                                          
 |    +--------------------------+                                                            
 +--> | bus_event_loop_with_idle |                                                            
      +--------------------------+                                                            
```

```
src/timedate/timedated.c                                                                   
+-------------------+                                                                       
| context_read_data | : get timezone (UTC) and save in context                              
+-|-----------------+                                                                       
  |    +--------------+                                                                     
  |--> | get_timezone | in our case, "/etc/localtime" doesn't exist, return 'UTC' by default
  |    +--------------+                                                                     
  |                                                                                         
  +--> save timezone in context                                                             
```

```
src/timedate/timedated.c                                                                                      
+----------------------------+                                                                                 
| context_parse_ntp_services | : find /usr/lib/systemd/ntp-units.d/*.list, save in context                     
+-|--------------------------+                                                                                 
  |    +---------------------------------------------+                                                         
  |--> | context_parse_ntp_services_from_environment | do nothing bc "SYSTEMD_TIMEDATED_NTP_SERVICES" isn't set
  |    +---------------------------------------------+                                                         
  |    +--------------------------------------+                                                                
  +--> | context_parse_ntp_services_from_disk | find /usr/lib/systemd/ntp-units.d/*.list, save in context      
       +--------------------------------------+                                                                
```

```
src/timedate/timedated.c                                                                           
+--------------------------------------+                                                            
| context_parse_ntp_services_from_disk | : find /usr/lib/systemd/ntp-units.d/*.list, save in context
+-|------------------------------------+                                                            
  |    +----------------------+                                                                     
  |--> | conf_files_list_strv | find /usr/lib/systemd/ntp-units.d/*.list                            
  |    +----------------------+                                                                     
  |                                                                                                 
  +--> for each found file                                                                          
       |                                                                                            
       |    +----------------+                                                                      
       |--> | fopen_unlocked |                                                                      
       |    +----------------+                                                                      
       |                                                                                            
       +--> endless loop                                                                            
            |                                                                                       
            |    +-----------+                                                                      
            |--> | read_line |                                                                      
            |    +-----------+                                                                      
            |    +-------------------------+                                                        
            +--> | context_add_ntp_service | alloc and save unit in context                         
                 +-------------------------+                                                        
```

```
src/timedate/timedated.c                                                                                  
+---------------------+                                                                                    
| method_set_timezone | : get timezone from msg, set to kernel                                             
+-|-------------------+                                                                                    
  |    +---------------------+                                                                             
  |--> | sd_bus_message_read | get timezone from msg                                                       
  |    +---------------------+                                                                             
  |    +-------------------+                                                                               
  |--> | timezone_is_valid |                                                                               
  |    +-------------------+                                                                               
  |    +-----------------------------+                                                                     
  |--> | context_write_data_timezone | determine timezone file, create symlink /etc/localtime linking to it
  |    +-----------------------------+                                                                     
  |    +--------------------+                                                                              
  |--> | clock_set_timezone | set timezone to kernel                                                       
  |    +--------------------+                                                                              
  |                                                                                                        
  |--> if ->local_rtc is set                                                                               
  |    |                                                                                                   
  |    |    +-------------------+                                                                          
  |    +--> | clock_set_hwclock | sync from system clock to rtc                                            
  |         +-------------------+                                                                          
  |    +--------------------------------+                                                                  
  |--> | sd_bus_emit_properties_changed | emit timezone change                                             
  |    +--------------------------------+                                                                  
  |    +----------------------------+                                                                      
  +--> | sd_bus_reply_method_return | reply msg                                                            
       +----------------------------+                                                                      
```
