## Index

- [Introduction](#introduction)
- [Methods](#methods)
- [Association](#association)
- [Cheat Sheet](#cheat-sheet)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

The main components of D-Bus are service, object, and interface; the object mapper helps collect much of the lengthy information and respond. 
The query is object-oriented and relies on the database, a two-level data structure built and maintained by the mapper.

- object-to-services map
   - Since the service can be used as a namespace, there's no rule preventing multiple services from potentially creating the same object path.
- service-to-interfaces map
   - Because of the database design, the way we locate interfaces of a target object is impacted, and service becomes the input instead.

<p align="center"><img src="images/openbmc/phosphor-objmgr.png" /></p>

We use `/xyz/openbmc_project/software` as an example to illustrate and distinguish the coverage of each method implemented by the object mapper.

```
             "/"                                                       |                  
             "/xyz"                                                    | GetAncestors()   
             "/xyz/openbmc_project"                                    |                  
                                                                                          
 target -->  "/xyz/openbmc_project/software"                           | GetObject()      
                                                                                          
             "/xyz/openbmc_project/software/10e36fd2"                  | GetSubTree() and 
             "/xyz/openbmc_project/software/10e36fd2/software_version" | GetSubTreePaths()
```

## <a name="methods"></a> Methods

### GetAncestors

Instead of a generic description, let's take the below example for introduction, and our target is `/xyz/openbmc_project/software/10e36fd2`.

```
busctl call --verbose \
   xyz.openbmc_project.ObjectMapper \        # service
   /xyz/openbmc_project/object_mapper \      # object
   xyz.openbmc_project.ObjectMapper \        # interface
   GetAncestors \                            # method
   sas \                                     # signature
   /xyz/openbmc_project/software \           # target object
   1 \                                       # number of the following interface(s)
   xyz.openbmc_project.Common.FactoryReset   # interface(s)
```       

For each object path, the mapper will list all the services that implement it, and obviously, the shorter path involves more services. 
With the help of the last argument: interface(s), we can limit the output to services that contain the interfaces intersecting our selection.

### GetObject

Given the argument object, we can query the mapper about the service that implements the object path; multiple services in the output list is possible. 
Like `GetAncestors`, it's optional to provide interfaces as the last argument, but it's rarely necessary for a practical scenario.

```
busctl call --verbose \
  xyz.openbmc_project.ObjectMapper \      # service
  /xyz/openbmc_project/object_mapper \    # object
  xyz.openbmc_project.ObjectMapper \      # interface
  GetObject \                             # method
  sas \                                   # signature
  /xyz/openbmc_project/FruDevice \        # target object
  0                                       # number of the following interface(s)
```

### GetSubTree

Based on the argument object, e.g., `/`, the method traverses every child object path starting with it and adds to the output.

```
busctl call --verbose \
   xyz.openbmc_project.ObjectMapper \     # service
   /xyz/openbmc_project/object_mapper \   # object
   xyz.openbmc_project.ObjectMapper \     # interface
   GetSubTree \                           # method
   sias \                                 # signature
   / \                                    # every object starts with the given path is targeted
   0 \                                    # valid depth to search (0 means unlimited)
   1 \                                    # number of the following interface(s)
   xyz.openbmc_project.FruDeviceManager   # interface(s) as constraint
```

### GetSubTreePaths

It's pretty much `GetSubTree` in every perspective, except the output is limited to the object path and has no service or interface information.

```
busctl call --verbose \
   xyz.openbmc_project.ObjectMapper \     # service
   /xyz/openbmc_project/object_mapper \   # object
   xyz.openbmc_project.ObjectMapper \     # interface
   GetSubTreePaths \                      # method
   sias \                                 # signature
   / \                                    # every object starts with the given path is targeted
   0 \                                    # valid depth to search (0 means unlimited)
   1 \                                    # number of the following interface(s)
   xyz.openbmc_project.FruDeviceManager   # interface(s) as constraint
```

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
  |    +-------------+
  |--> | doListNames |
  |    +-------------+
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
   
```
src/main.cpp
+-------------+                                                                             
| doListNames | : add iface to map, update owners                                           
+-|-----------+                                                                             
  |                                                                                         
  +--> ->async_method_call                                                                  
          +--------------------------------------------------------------------------------+
          |sort process_names                                                              |
          |                                                                                |
          |for each process_name                                                           |
          ||                                                                               |
          ||    +--------------------+                                                     |
          ||--> | startNewIntrospect | recursively add iface to map and handle association |
          ||    +--------------------+                                                     |
          ||    +--------------+                                                           |
          |+--> | updateOwners | owners[nameOwner] = newObject                             |
          |     +--------------+                                                           |
          +--------------------------------------------------------------------------------+
          service: org.freedesktop.DBus                                                     
          object: /org/freedesktop/DBus                                                     
          iface: org.freedesktop.DBus                                                       
          method: org.freedesktop.DBus                                                      
```
   
```
src/main.cpp
+--------------------+                                                          
| startNewIntrospect | : recursively add iface to map and handle association    
+-|------------------+                                                          
  |    +--------------+                                                         
  +--> | doIntrospect | : recursively add iface to map and handle association   
       +-|------------+                                                         
         |                                                                      
         +--> ->async_method_call                                               
                 +-------------------------------------------------------------+
                 |parse the input xml                                          |
                 |                                                             |
                 |start from element 'interface'                               |
                 |                                                             |
                 |while element != null                                        |
                 ||                                                            |
                 ||--> get iface from 'name'                                   |
                 ||                                                            |
                 ||--> save in 'interfaceMap'                                  |
                 ||                                                            |
                 ||--> if iface == xyz.openbmc_project.Association.Definitions |
                 ||    |                                                       |
                 ||    |    +----------------+                                 |
                 ||    +--> | doAssociations | (skip)                          |
                 ||         +----------------+                                 |
                 ||                                                            |
                 |+--> element = next sibling                                  |
                 |                                                             |
                 |+---------------------------+                                |
                 || checkIfPendingAssociation | (skip)                         |
                 |+---------------------------+                                |
                 |                                                             |
                 |start from element 'node'                                    |
                 |                                                             |
                 |while element != null                                        |
                 ||                                                            |
                 ||--> get child_path from 'name'                              |
                 ||                                                            |
                 ||--> if child_path exists                                    |
                 ||    |                                                       |
                 ||    |    +--------------+                                   |
                 ||    +--> | doIntrospect | (recursive)                       |
                 ||         +--------------+                                   |
                 ||                                                            |
                 |+--> element = next sibling                                  |
                 +-------------------------------------------------------------+
                 |service: process_name                                        |
                 |object: path                                                 |
                 |iface: org.freedesktop.DBus.Introspectable                   |
                 |method: Introspect                                           |
                 +-------------------------------------------------------------+
```

</details>

## <a name="association"></a> Association

(TBD)

## <a name="cheat-sheet"></a> Cheat Sheet

- GetAncestors

```
busctl call --verbose \
   xyz.openbmc_project.ObjectMapper \
   /xyz/openbmc_project/object_mapper \
   xyz.openbmc_project.ObjectMapper \
   GetAncestors \
   sas \
   /xyz/openbmc_project/software \
   1 \
   xyz.openbmc_project.Common.FactoryReset
```

- GetObject

```
busctl call --verbose \
  xyz.openbmc_project.ObjectMapper \
  /xyz/openbmc_project/object_mapper \
  xyz.openbmc_project.ObjectMapper \
  GetObject \
  sas \
  /xyz/openbmc_project/FruDevice \
  0
```

- GetSubTree

```
busctl call --verbose \
   xyz.openbmc_project.ObjectMapper \
   /xyz/openbmc_project/object_mapper \
   xyz.openbmc_project.ObjectMapper \
   GetSubTree \
   sias \
   / \
   0 \
   1 \
   xyz.openbmc_project.FruDeviceManager
```

- GetSubTreePaths

```
busctl call --verbose \
   xyz.openbmc_project.ObjectMapper \
   /xyz/openbmc_project/object_mapper \
   xyz.openbmc_project.ObjectMapper \
   GetSubTreePaths \
   sias \
   / \
   0 \
   1 \
   xyz.openbmc_project.FruDeviceManager
```

## <a name="reference"></a> Reference

- [The Mapper](https://github.com/openbmc/phosphor-objmgr)