```
+------+                                                                               
| main |                                                                               
+-|----+                                                                               
  |                                                                                    
  |--> prepare signal handlers for SIGINT, SIGTERM                                     
  |                                                                                    
  |--> prepare handler and match rule for name change                                  
  |    +------------+                                                                  
  |    | anon func1 | update 'name_owners', introspect target and update associaion map
  |    +------------+                                                                  
  |                                                                                    
  |--> prepare handler and match rule for added iface                                  
  |    +------------+                                                                  
  |    | anon func2 |                                                                  
  |    +------------+                                                                  
  |                                                                                    
  |--> prepare handler and match rule for removed iface                                
  |    +------------+                                                                  
  |    | anon func3 |                                                                  
  |    +------------+                                                                  
  |                                                                                    
  |--> prepare handler and match rule for changed assoc                                
  |    +--------------------+                                                          
  |    | associationChanged | update association map                                   
  |    +--------------------+                                                          
  |                                                                                    
  |--> set up object: "/xyz/openbmc_project/object_mapper"                             
  |                                                                                    
  |--> prepare callback for method "GetAncestors"                                      
  |        +--------------+                                                            
  |        | getAncestors | get ancestors of req_path                                  
  |        +--------------+                                                            
  |                                                                                    
  |--> prepare callback for method "GetObject"                                         
  |        +-----------+                                                               
  |        | getObject | given path & iface_map, get object(s)                         
  |        +-----------+                                                               
  |                                                                                    
  |--> prepare callback for method "GetSubTree"                                        
  |        +------------+                                                              
  |        | getSubTree | get sub tree                                                 
  |        +------------+                                                              
  |                                                                                    
  |--> prepare callback for method "GetSubTreePaths"                                   
  |        +-----------------+                                                         
  |        | getSubTreePaths | get sub tree paths                                      
  |        +-----------------+                                                         
  |                                                                                    
  +--> request service: "xyz.openbmc_project.ObjectMapper"                             
```

```
+------------+                                                                      
| anon func1 | : update 'name_owners', introspect target and update associaion map  
+-|----------+                                                                      
  |                                                                                 
  |--> read 'name', 'old owner' and 'new owner' from msg                            
  |                                                                                 
  |--> if 'old owner'                                                               
  |                                                                                 
  |        +-------------------------+                                              
  |------> | processNameChangeDelete | (skip)                                       
  |        +-------------------------+                                              
  |                                                                                 
  |--> if 'new owner'                                                               
  |                                                                                 
  |        +------------------+                                                     
  |------> | needToIntrospect | check if it's in whitelist only (not in blacklist)  
  |        +------------------+                                                     
  |                                                                                 
  |------> if yes                                                                   
  |                                                                                 
  |----------> name_owners[new_owner] = name                                        
  |                                                                                 
  |            +----------------------+                                             
  +----------> | start_new_introspect | introspect target and update association map
               +----------------------+                                             
```

```
+----------------------+                                                                             
| start_new_introspect | : introspect target and update association map                              
+-|--------------------+                                                                             
  |    +------------------+                                                                          
  |--> | needToIntrospect | check if it's in whitelist only (not in blacklist)                       
  |    +------------------+                                                                          
  |                                                                                                  
  |--> if yes                                                                                        
  |                                                                                                  
  |        +---------------+                                                                         
  +------> | do_introspect | introspect target, for each node: for each iface: update association map
           +---------------+                                                                         
```

```
+---------------+                                                                           
| do_introspect | : introspect target, for each node: for each iface: update association map
+-|-------------+                                                                           
  |                                                                                         
  |--> prepare callback for method call                                                     
  |       +-----------------------------------------------------------------------+         
  |       |for each 'interface' node of root node                                 |         
  |       |                                                                       |         
  |       |    add iface_name to thisPathMap[process_name]                        |         
  |       |                                                                       |         
  |       |    if iface_name == "xyz.openbmc_project.Association.Definitions"     |         
  |       |                                                                       |         
  |       |        +-----------------+                                            |         
  |       |        | do_associations | get target info and update association map |         
  |       |        +-----------------+                                            |         
  |       |+---------------------------+                                          |         
  |       || checkIfPendingAssociation | (skip)                                   |         
  |       |+---------------------------+                                          |         
  |       |                                                                       |         
  |       |for each 'node' of root node                                           |         
  |       |                                                                       |         
  |       |    +---------------+                                                  |         
  |       |    | do_introspect | (recursive)                                      |         
  |       |    +---------------+                                                  |         
  |       +-----------------------------------------------------------------------+         
  |                                                                                         
  +--> call esrvice: process_name                                                           
            object: path                                                                    
            iface: "org.freedesktop.DBus.Introspectable"                                    
            method: "Introspect"                                                            
```

```
+-----------------+                                             
| do_associations | : get target info and update association map
+-|---------------+                                             
  |                                                             
  |--> prepare callback for method call                         
  |       +----------------------------------------------+      
  |       |+--------------------+                        |      
  |       || associationChanged | update association map |      
  |       |+--------------------+                        |      
  |       +----------------------------------------------+      
  |                                                             
  +--> call service: processName                                
            object: path                                        
            iface: "org.freedesktop.DBus.Properties"            
            method: "Get"                                       
```

```
+--------------------+                                                                                        
| associationChanged | : update association map                                                               
+-|------------------+                                                                                        
  |                                                                                                           
  |--> for each association                                                                                   
  |                                                                                                           
  |------> get 'forward', 'reverse', and 'endpoint' from association                                          
  |                                                                                                           
  |------> if endpoint isn't in arg continue                                                                  
  |                                                                                                           
  |            +-----------------------+                                                                      
  |----------> | addPendingAssociation | (skip)                                                               
  |            +-----------------------+                                                                      
  |                                                                                                           
  |----------> continue                                                                                       
  |                                                                                                           
  |------> if 'forward'                                                                                       
  |                                                                                                           
  |----------> add 'endpoint' to objects[path + "/" + forward]                                                
  |                                                                                                           
  |------> if 'reverse'                                                                                       
  |                                                                                                           
  |            add arg path to objects[path + "/" + reverse]                                                  
  |                                                                                                           
  |--> for each object                                                                                        
  |                                                                                                           
  |        +---------------------------+                                                                      
  |------> | addEndpointsToAssocIfaces | give arg endpointPaths, ensure they are all in endpoints of interface
  |        +---------------------------+                                                                      
  |    +---------------------------------+                                                                    
  |--> | checkAssociationEndpointRemoves | check if endpoints being removed                                   
  |    +---------------------------------+                                                                    
  |                                                                                                           
  |--> if objects isn't empty                                                                                 
  |                                                                                                           
  +------> ensure (path, owner, objects) exists                                                               
```

```
+---------------------------+                                                                        
| addEndpointsToAssocIfaces | : give arg endpointPaths, ensure they are all in endpoints of interface
+-|-------------------------+                                                                        
  |                                                                                                  
  |--> get iface from arg assocMaps                                                                  
  |                                                                                                  
  |--> get endpoints from iface                                                                      
  |                                                                                                  
  |--> for each endpoint_path in arg 'endpointPaths'                                                 
  |                                                                                                  
  |------> add endpoint_path to endpoints if it's not there yet                                      
  |                                                                                                  
  |--> if iface exists already                                                                       
  |                                                                                                  
  |------> set_property("endpoints") with endpoints                                                  
  |                                                                                                  
  |--> else                                                                                          
  |                                                                                                  
  |------> add interface "xyz.openbmc_project.Association" for arg assocPath                         
  |                                                                                                  
  +------> register_property("endpoints") with endpoints                                             
```

```
+------------+                                                                 
| anon func2 |                                                                 
+--|---------+                                                                 
   |                                                                           
   |--> read msg                                                               
   |                                                                           
   |    +------------------+                                                   
   |--> | needToIntrospect | check if it's in whitelist only (not in blacklist)
   |    +------------------+                                                   
   |    +-----------------------+                                              
   +--> | processInterfaceAdded | handle added interface                       
        +-----------------------+                                              
```

```
+-----------------------+
| processInterfaceAdded | : handle added interface
+-|---------------------+
  |
  |--> for each added_iface
  |    -
  |    +--> if the iface == "xyz.openbmc_project.Association.Definitions"
  |         |
  |         |--> for each iface (property?)
  |         |    -
  |         |    +--> if it's "Associations"
  |         |         -
  |         |         +--> svae it in variantAssociations
  |         |
  |         |    +--------------------+
  |         +--> | associationChanged | update association map
  |              +--------------------+
  |
  |--> while we haven't reached root obj yet
  |    |
  |    |--> prepare default iface for current parent
  |    |
  |    +--> continue for higher parent
  |
  |    +---------------------------+
  +--> | checkIfPendingAssociation |
       +---------------------------+
```

```
+------------+                                                           
| anon func3 |                                                           
+-|----------+                                                           
  |    +--------------+                                                  
  |--> | getWellKnown | check if the owner is 'well-known'               
  |    +--------------+                                                  
  |                                                                      
  |--> return if not                                                     
  |                                                                      
  |--> for each removed iface                                            
  |    |                                                                 
  |    |--> continue if the sender isn't in connection_map               
  |    |                                                                 
  |    |--> if the iface is "xyz.openbmc_project.Association.Definitions"
  |    |    |                                                            
  |    |    |    +-------------------+                                   
  |    |    +--> | removeAssociation | remove association                
  |    |         +-------------------+                                   
  |    |                                                                 
  |    +--> erase iface from interface_set                               
  |                                                                      
  |         if interface_set becomes empty                               
  |         -                                                            
  |         +--> erase interface_set from connection_map                 
  |                                                                      
  |--> if connection_map becomes empty                                   
  |    -                                                                 
  |    +--> remove connection_map from interface_map                     
  |                                                                      
  |    +-----------------------+                                         
  +--> | removeUnneededParents | remove unneeded parents                 
       +-----------------------+                                         
```

```
+-------------------+                                                                                         
| removeAssociation | : remove association                                                                    
+-|-----------------+                                                                                         
  |                                                                                                           
  +--> find services (owners) that have the object (path)                                                     
  |                                                                                                           
  |--> find assoccs of the arg owner                                                                          
  |                                                                                                           
  |--> for each (assoc_path, ep_to_remove)                                                                    
  |    |                                                                                                      
  |    |    +----------------------------+                                                                    
  |    +--> | removeAssociationEndpoints | remove endpoints from iface, and further remove iface if it's empty
  |         +----------------------------+                                                                    
  |                                                                                                           
  |--> erase assoc from owners                                                                                
  |                                                                                                           
  |--> if 'owners' becomes empty                                                                              
  |    -                                                                                                      
  |    +--> erase it from assocMaps                                                                           
  |                                                                                                           
  |    +-------------------------------+                                                                      
  +--> | removeFromPendingAssociations | (skip)                                                               
       +-------------------------------+                                                                      
```

```
+----------------------------+                                                                      
| removeAssociationEndpoints | : remove endpoints from iface, and further remove iface if it's empty
+-|--------------------------+                                                                      
  |                                                                                                 
  |--> get 'endpointsInDBus' from assoc                                                             
  |                                                                                                 
  |--> for each ep_to_remove                                                                        
  |    -                                                                                            
  |    +--> if the ep is in 'endpointsInDBus', erase it                                             
  |                                                                                                 
  |--> if endpointsInDBus becomes empty afterwards                                                  
  |    -                                                                                            
  |    +--> remove iface as well                                                                    
  |                                                                                                 
  +--> else                                                                                         
       -                                                                                            
       +--> set_property("endpoints", endpointsInDBus)                                              
```

```
+-----------------------+                               
| removeUnneededParents | : remove unneeded parents     
+-|---------------------+                               
  |                                                     
  +--> endless loop                                     
       |                                                
       |--> break if an object has more than three iface
       |                                                
       |--> remove object from interface_map            
       |                                                
       +--> continue to check upper level               
```

```
+--------------+                                         
| getAncestors | : get ancestors of req_path             
+-|------------+                                         
  |                                                      
  |--> ensure there's no tailing '/' of req_path         
  |                                                      
  +--> for each obj_path in iface_map                    
       -                                                 
       +--> add to 'ret' if it's the ancestor of req_path
```

```
+------------+                                                                                               
| getSubTree | : get sub tree                                                                                
+-|----------+                                                                                               
  |                                                                                                          
  |--> sort interfaces for later intersect() to work                                                         
  |                                                                                                          
  |--> ensure there's no tailing '/' of req_path                                                             
  |                                                                                                          
  +--> for each obj_path in iface_map                                                                        
       |                                                                                                     
       |--> calculate this_depth                                                                             
       |                                                                                                     
       +--> if the this_depth is valid (less than the specified one)                                         
            -                                                                                                
            +--> for each iface_map in obj_path (???)                                                        
                 |                                                                                           
                 |    +--------------------+                                                                 
                 +--> | addObjectMapResult | ensure there's an entry in 'objectmap' contains the interfaceMap
                      +--------------------+                                                                 
```

```
+--------------------+                                                                   
| addObjectMapResult | : ensure there's an entry in 'objectmap' contains the interfaceMap
+-|------------------+                                                                   
  |                                                                                      
  |--> check if entry is already in 'objectMap'                                          
  |                                                                                      
  |--> if found                                                                          
  |    -                                                                                 
  |    +--> add 'interfaceMap' to it                                                     
  |                                                                                      
  +--> else                                                                              
       |                                                                                 
       |--> prepare entry first                                                          
       |                                                                                 
       |--> add 'interfaceMap' to it                                                     
       |                                                                                 
       +--> add entry to 'objectMap'                                                     
```

```
+-----------------+                                          
| getSubTreePaths | : get sub tree paths                     
+-|---------------+                                          
  |                                                          
  |--> sort interfaces for later intersect() to work         
  |                                                          
  |--> ensure there's no tailing '/' of req_path             
  |                                                          
  +--> for each obj_path in iface_map                        
       |                                                     
       |--> calculate depth and check if it's valid          
       |                                                     
       |--> for each iface_map in obj_path                   
       |    -                                                
       |    +--> break if intersect() returns true           
       |                                                     
       +--> add this_path to ret if intersect() returned true
```

```
busctl call --verbose \
  xyz.openbmc_project.ObjectMapper \
  /xyz/openbmc_project/object_mapper \
  xyz.openbmc_project.ObjectMapper \
  GetObject sas /xyz/openbmc_project/FruDevice 0
```