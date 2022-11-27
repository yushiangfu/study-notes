## Index

- [Introduction](#introduction)
- [Methods](#methods)
- [Association](#association)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

(TBD)

## <a name="methods"></a> Methods

### GetAncestors

Instead of a generic description, let's take the below example for introduction, and our target is `/xyz/openbmc_project/software/10e36fd2`.

```
busctl call --verbose \
   xyz.openbmc_project.ObjectMapper \         # service
   /xyz/openbmc_project/object_mapper \       # object
   xyz.openbmc_project.ObjectMapper \         # interface
   GetAncestors \                             # method
   sas \                                      # signature
   /xyz/openbmc_project/software/10e36fd2 \   # target object
   1 \                                        # number of the following interface(s)
   xyz.openbmc_project.Common.FactoryReset    # interface(s)
```       

The valid ancestors include:

- `(empty)`
  - why is this necessary? Let's ignore this one for now.
- `/`
- `/xyz`
- `/xyz/openbmc_project`
- `/xyz/openbmc_project/software`

For each object path, the mapper will list all the services that implement it, and obviously, the shorter path involves more services. 
With the help of the last argument: interface(s), we can limit the output to services that contain the interfaces intersecting our selection.

<details><summary> More Details </summary>

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
  |    | anon func2 | handle added interface
  |    +------------+                                                                  
  |                                                                                    
  |--> prepare handler and match rule for removed iface                                
  |    +------------+                                                                  
  |    | anon func3 | handle entry removal
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
| anon func2 | : handle added interface
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
| anon func3 | : handle entry removal
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

</details>

## <a name="association"></a> Association

(TBD)

## <a name="cheat-sheet"></a> Cheat Sheet

```
busctl call --verbose \
   xyz.openbmc_project.ObjectMapper \
   /xyz/openbmc_project/object_mapper \
   xyz.openbmc_project.ObjectMapper \
   GetAncestors \
   sas \
   /xyz/openbmc_project/software/10e36fd2 \
   1 \
   xyz.openbmc_project.Common.FactoryReset
```

```
busctl call --verbose \
  xyz.openbmc_project.ObjectMapper \
  /xyz/openbmc_project/object_mapper \
  xyz.openbmc_project.ObjectMapper \
  GetObject sas /xyz/openbmc_project/FruDevice 0
```

## <a name="reference"></a> Reference

- [The Mapper](https://github.com/openbmc/phosphor-objmgr)
