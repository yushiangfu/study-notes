### phosphor-dump-manager

```
dump_manager_main.cpp
```

```
refer to 'meson_options.txt' for path definitions                                                
dump_manager_main.cpp                                                                            
+------+                                                                                          
| main |                                                                                          
+-|----+                                                                                          
  |    +------------------+                                                                       
  |--> | sd_event_default | ???                                                                   
  |    +------------------+                                                                       
  |                                                                                               
  |--> prepare bmc_dump_manager and add to list                                                   
  |                                                                                               
  |--> prepare fault_log_manager and add to list                                                  
  |                                                                                               
  |    +----------------------+                                                                   
  |--> | dump::loadExtensions | do nothing                                                        
  |    +----------------------+                                                                   
  |                                                                                               
  |--> for each manager in list                                                                   
  |    |                                                                                          
  |    |    +------------------+                                                                  
  |    +--> | Manager::restore |                                                                  
  |         +------------------+                                                                  
  |                                                                                               
  |--> prepare eWatch                                                                             
  |    +--------------+                                                                           
  |    | Watch::Watch | register callbacks for interface events, read in file 'elogid' to variable
  |    +--------------+                                                                           
  |                                                                                               
  |--> request service 'xyz.openbmc_project.Dump.Manager'                                         
  |                                                                                               
  |    +---------------+                                                                          
  +--> | sd_event_loop | endless standby and work?                                                
       +---------------+                                                                          
```

```
+------------------+                                                                                     
| Manager::restore | : for each file in dump_dir, ensure they are in 'entries'                           
+-|----------------+                                                                                     
  |                                                                                                      
  |--> return if dump_dir doesn't exist or is empty                                                      
  |                                                                                                      
  |--> prepare iterator for the dir                                                                      
  |                                                                                                      
  +--> for each file in dump_dir                                                                         
       -                                                                                                 
       +--> if file name consists of digit only (otherwise we ignore it)                                 
            |                                                                                            
            |    +----------------------+                                                                
            +--> | Manager::createEntry | ensure there's the corresponding entry of arg file in 'entries'
                 +----------------------+                                                                
```

```
+----------------------+                                                                  
| Manager::createEntry | : ensure there's the corresponding entry of arg file in 'entries'
+-|--------------------+                                                                  
  |                                                                                       
  |--> return if arg file name doesn't match the pattern                                  
  |                                                                                       
  |--> get id from file name                                                              
  |                                                                                       
  |--> if id is in 'entries' already                                                      
  |    -                                                                                  
  |    +--> update info in 'entries'                                                      
  |                                                                                       
  +--> else                                                                               
       -                                                                                  
       +--> alloc entry and insert to 'entries'                                           
```

```
+--------------+                                                                             
| Watch::Watch | : register callbacks for interface events, read in file 'elogid' to variable
+-|------------+                                                                             
  |                                                                                          
  |--> prepare callbacks for interface addition and removal                                  
  |    +--------------------+                                                                
  |    | Watch::addCallback | fork a child running dreport                                   
  |    +--------------------+                                                                
  |    | Watch::delCallback | remove eid from elog_list, then serialize (?)                  
  |    +--------------------+                                                                
  |                                                                                          
  +--> if the file exists (/var/lib/phosphor-debug-collector/elogid)                         
       |                                                                                     
       |    +----------------------+                                                         
       +--> | logging::deserialize | read in file and archive to, e.g., elog_list            
            +----------------------+                                                         
```

```
+--------------------+                                                         
| Watch::addCallback | : fork a child running dreport                          
+-|------------------+                                                         
  |                                                                            
  |--> read msg                                                                
  |                                                                            
  |--> get eid and return if it's in the elog list already                     
  |                                                                            
  |--> find property "Message" of interface "xyz.openbmc_project.Logging.Entry"
  |                                                                            
  |--> given data, find error type                                             
  |                                                                            
  |--> return if the error type isn't support                                  
  |                                                                            
  |--> insert eid into elog list                                               
  |                                                                            
  |--> given error type, find item in bmc::TypeMap                             
  |                                                                            
  |    +-----------------+                                                     
  +--> | Manager::create | fork a child running dreport                        
       +-----------------+                                                     
```

```
+-----------------+                                                                              
| Manager::create | : fork a child running dreport                                               
+-|---------------+                                                                              
  |    +----------------------+                                                                  
  +--> | Manager::captureDump | : fork a child running dreport                                   
       +-|--------------------+                                                                  
         |                                                                                       
         |--> fork                                                                               
         |                                                                                       
         |--> if pid == 0 (the forked child)                                                     
         |    |                                                                                  
         |    |    +-------+ +---------+                                                         
         |    +--> | execl | | dreport | create archive of debug info files (including core file)
         |         +-------+ +---------+                                                         
         |                                                                                       
         |--> else (parent)                                                                      
         |    |    +--------------------+                                                        
         |    |    | sd_event_add_child | ???                                                    
         |    +--> +--------------------+                                                        
         |                                                                                       
         +--> update last_entry_id for next event                                                
```

```
+--------------------+                                                
| Watch::delCallback | : remove eid from elog_list, then serialize (?)
+-|------------------+                                                
  |                                                                   
  |--> read msg                                                       
  |                                                                   
  |--> get eid from obj_path                                          
  |                                                                   
  +--> if eid is in elog_list                                         
       |                                                              
       |--> erase it                                                  
       |                                                              
       |    +-----------------+                                       
       +--> | elog::serialize | ???                                   
            +-----------------+                                       
```

### phosphor-dump-monitor

```
core_manager_main.cpp
```
