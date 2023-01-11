### phosphor-log-manager

‵‵‵
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
  ‵‵‵
  
  ‵‵‵
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
  ‵‵‵
