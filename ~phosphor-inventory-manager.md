```
app.cpp                                                                                         
+------+                                                                                         
| main |                                                                                         
+-|----+                                                                                         
  |                                                                                              
  |--> prepare manager with obj path = /xyz/openbmc_project/inventory                            
  |    +------------------+                                                                      
  |    | Manager::Manager | register callback for each event, restore association from filesystem
  |    +------------------+                                                                      
  |                                                                                              
  +--> run manager for service (xyz.openbmc_project.Inventory.Manager)                           
```

```
manager.cpp                                                                                
+------------------+                                                                        
| Manager::Manager | : register callback for each event, restore association from filesystem
+-|----------------+                                                                        
  |                                                                                         
  |--> for each group                                                                       
  |    -                                                                                    
  |    +--> for each event in group                                                         
  |         |                                                                               
  |         |--> continue if event_type != signal                                           
  |         |                                                                               
  |         |--> add event to _sigargs                                                      
  |         |                                                                               
  |         +--> register callback for the event (by adding it to _matches?)                
  |                                                                                         
  |    +------------------+                                                                 
  +--> | Manager::restore | restore from filesystem and set up association if necessary     
       +------------------+                                                                 
```

```
manager.cpp                                                                                       
+------------------+                                                                               
| Manager::restore | : restore from filesystem and set up association if necessary                 
+-|----------------+                                                                               
  |                                                                                                
  |--> return if /var/lib/phosphor-inventory-manager doesn't exist                                 
  |                                                                                                
  |--> for each file under /var/lib/phosphor-inventory-manager                                     
  |    -                                                                                           
  |    +--> if it's a regular file                                                                 
  |         |                                                                                      
  |         |--> get obj_path from file_path                                                       
  |         |                                                                                      
  |         |--> if obj_path is in objects already                                                 
  |         |    -                                                                                 
  |         |    +--> udpate the (iface, prop_map) in that obj                                     
  |         |                                                                                      
  |         +--> else                                                                              
  |              -                                                                                 
  |              +--> create a obj with (iface, prop_map), add to objects                          
  |                                                                                                
  +--> if objects has at least one obj                                                             
       -                                                                                           
       +--> if 'associations' is not empty                                                         
            |                                                                                      
            |--> for each condition in associations                                                
            |    -                                                                                 
            |    +-->  get iface, property, and further the value                                  
            |                                                                                      
            +--> if there's any match                                                              
                 -                                                                                 
                 +-->  create association                                                          
```
