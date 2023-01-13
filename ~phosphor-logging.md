### phosphor-log-manager

```
log_manager_main.cpp                                                                   
+------+                                                                                
| main | : restore entries from fs, call each extension's startup func, request bus name
+-|----+                                                                                
  |                                                                                     
  |--> create folder '/var/lib/phosphor-logging/errors'                                 
  |                                                                                     
  |    +------------------+                                                             
  |--> | Manager::restore | restore log entries from folder to memory                   
  |    +------------------+                                                             
  |                                                                                     
  |--> for each extension                                                               
  |    -                                                                                
  |    +--> call its startup function                                                   
  |                                                                                     
  +--> request 'xyz.openbmc_project.Logging'                                            
```
  
```
log_manager.cpp                                                
+------------------+                                            
| Manager::restore | : restore log entries from folder to memory
+-|----------------+                                            
  |                                                             
  |--> dir = /var/lib/phosphor-logging/errors                   
  |                                                             
  |--> if it doesn't exist or is empty, return                  
  |                                                             
  |--> for each file under this folder                          
  |    |                                                        
  |    |--> if entry_severity >= lower_limit                    
  |    |    -                                                   
  |    |    +--> append id to 'infoErrors'                      
  |    |                                                        
  |    |--> else                                                
  |    |    -                                                   
  |    |    +--> append id to 'realErrors'                      
  |    |                                                        
  |    +--> add entry to entries                                
  |                                                             
  +--> if entries isn't empty                                   
       -                                                        
       +--> entry_id = id of the last entry                     
```

```
e.g., what is lg2::info

PHOSPHOR_LOG2_DECLARE_LEVEL(info);
    lg2::info --> lg2::log<info>

lg2::log::log
    lg2::details::log_conversion::start
    lg2::details::log_conversion::step
    lg2::details::log_conversion::step
    ...
    lg2::details::log_conversion::done --> lg2::details::do_log
```

```
lib/lg2_logger.cpp                                                    
+----------------------+                                               
| lg2::details::do_log |                                               
+-|--------------------+                                               
  |                                                                    
  |--> save static info (file_name, line, func_name) in local 'strings'
  |                                                                    
  |--> va_start                                                        
  |                                                                    
  |--> endless loop                                                    
  |    |                                                               
  |    |--> va_arg: get header                                         
  |    |                                                               
  |    |--> if no more hader, break                                    
  |    |                                                               
  |    |--> va_arg: get format_flag                                    
  |    |                                                               
  |    |--> switch format_flag                                         
  |    |    -                                                          
  |    |    +--> get value and convert based on format_flag            
  |    |                                                               
  |    |--> prepare 'header=value' and append to local 'strings'       
  |    |                                                               
  |    +--> if '{header'} is in local 'msg', replace value             
  |                                                                    
  |--> va_end                                                          
  |                                                                    
  |--> append 'msg' to 'strings'                                       
  |                                                                    
  |--> transform strings to io vector                                  
  |                                                                    
  |    +------------------+                                            
  |--> | sd_journal_sendv |                                            
  |    +------------------+                                            
  |    +---------------------+                                         
  +--> | extra_output_method | output to tty as well if we have it     
       +---------------------+                                         
```

### phosphor-rsyslog-conf

```
phosphor-rsyslog-config/main.cpp                                 
+------+                                                          
| main |                                                          
+-|----+                                                          
  |                                                               
  |--> prepare server with arg                                    
  |        obj_path = "/xyz/openbmc_project/logging/config/remote"
  |        file_path = /etc/rsyslog.d/server.conf                 
  |    +----------------+                                         
  |    | Server::Server | restore (addr, port) setting from file  
  |    +----------------+                                         
  |                                                               
  |--> request name "xyz.openbmc_project.Syslog.Config"           
  |                                                               
  +--> endless loop                                               
       |                                                          
       |--> bus.process_discard() <-- ???                         
       |                                                          
       +--> bus.wait()                                            
```
