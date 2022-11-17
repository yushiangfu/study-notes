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

### phosphor-dump-monitor

```
core_manager_main.cpp
```
