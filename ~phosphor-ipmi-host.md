```
ipmid-new.cpp                                                                                                               
+------+                                                                                                                     
| main | : load handlers and filters, prepare callback for executing ipmi commands                                           
+-|----+                                                                                                                     
  |                                                                                                                          
  |--> determine 'user' or 'system' bus                                                                                      
  |                                                                                                                          
  |    +---------------------+                                                                                               
  |--> | ipmi::loadProviders | : find all *.so under arg path and dlopen them, which gives them a chance registering handlers
  |    +---------------------+                                                                                               
  |                                                                                                                          
  |--> prepare callback to support deprecated api                                                                            
  |    +-------------------------+                                                                                           
  |    | handleLegacyIpmiCommand | (skip)                                                                                    
  |    +-------------------------+                                                                                           
  |                                                                                                                          
  |--> prepare callback for bus name watching                                                                                
  |    +-------------------------+                                                                                           
  |    | ipmi::nameChangeHandler | uniqueNameToChannelNumber[newOwner] = new channel                                         
  |    +-------------------------+                                                                                           
  |                                                                                                                          
  |--> request service: "xyz.openbmc_project.Ipmi.Host"                                                                      
  |                                                                                                                          
  |--> set up object: "/xyz/openbmc_project/Ipmi"                                                                            
  |                                                                                                                          
  +--> prepare callback for method 'execute'                                                                                 
       +----------------------+                                                                                              
       | ipmi::executionEntry | handle channel, prepare context, exccute ipmi command, prepare response for return           
       +----------------------+                                                                                              
```

```
+---------------------+                                                                                               
| ipmi::loadProviders | : find all *.so under arg path and dlopen them, which gives them a chance registering handlers
+-|-------------------+                                                                                               
  |                                                                                                                   
  |--> for each lib under arg path                                                                                    
  |                                                                                                                   
  |------> continue if it' a link                                                                                     
  |                                                                                                                   
  |------> if it has the extension '.so'                                                                              
  |                                                                                                                   
  |----------> add to local 'libs'                                                                                    
  |                                                                                                                   
  |--> prepare empty list of'IpmiProvider'                                                                            
  |                                                                                                                   
  |--> for each found lib                                                                                             
  |                                                                                                                   
  +------> add lib path to the 'IpmiProvider' list                                                                    
           +----------------------------+                                                                             
           | IpmiProvider::IpmiProvider | dlopen, which triggers the feature of __attribute__((constructor))          
           +----------------------------+ (lib uses this feature to register its handlers)                            
```

```
+----------------------+                                                                                     
| ipmi::executionEntry | : handle channel, prepare context, exccute ipmi command, prepare response for return
+-|--------------------+                                                                                     
  |    +--------------------+                                                                                
  |--> | channelFromMessage | get channel from msg                                                           
  |    +--------------------+                                                                                
  |                                                                                                          
  |--> if it's session-based channel (user_id, priviledge, session_id are required)                          
  |                                                                                                          
  |------> get user_id, priviledge, and session_id from arg                                                  
  |                                                                                                          
  |--> else                                                                                                  
  |                                                                                                          
  |------> set priviledge to admin (max)                                                                     
  |                                                                                                          
  |        +----------------+                                                                                
  |------> | getChannelInfo | get info from channel                                                          
  |        +----------------+                                                                                
  |                                                                                                          
  |------> if medium_type is 'ipmb'                                                                          
  |                                                                                                          
  |----------> get "rqSA" & "hostId" info                                                                    
  |                                                                                                          
  |--> prepare ipmi_context                                                                                  
  |                                                                                                          
  |--> prepare request based on arg data                                                                     
  |                                                                                                          
  |    +--------------------------+                                                                          
  |--> | ipmi::executeIpmiCommand | execute ipmi command                                                     
  |    +--------------------------+                                                                          
  |                                                                                                          
  +--> prepare response and return                                                                           
```

```
+--------------------------+                                                                     
| ipmi::executeIpmiCommand | : execute ipmi command                                              
+-|------------------------+                                                                     
  |                                                                                              
  |--> get net_fn from arg request                                                               
  |                                                                                              
  |--> if it's net_fn_group (0x2c)                                                               
  |                                                                                              
  |               +--------------------------------+                                             
  |------> return |  ipmi::executeIpmiGroupCommand | (skip)                                      
  |               +--------------------------------+                                             
  |                                                                                              
  |--> elif it's net_fn_oem (0x2e)                                                               
  |                                                                                              
  |               +-----------------------------+                                                
  |------> return | ipmi::executeIpmiOemCommand | (skip)                                         
  |               +-----------------------------+                                                
  |    +--------------------------------+                                                        
  +--> | ipmi::executeIpmiCommandCommon | prepare key and find target from arg 'handlers' to call
       +--------------------------------+                                                        
```

```
+--------------------------------+
| ipmi::executeIpmiCommandCommon | : prepare key and find target from arg 'handlers' to call
+-|------------------------------+
  |    +-------------------------+
  |--> | ipmi::filterIpmiCommand | check if the request is blocked for some reasson
  |    +-------------------------+
  |    +------------------+
  |--> | ipmi::makeCmdKey |
  |    +------------------+
  |
  |--> if we can find the key in arg handlers
  |
  |------> to handle request, run ->call(), e.g.,
  |        +-------------------------+
  |        | ipmiStorageWriteFruData | (I think the c++ unpack plays trick here, otherwise the arguments mismatch)
  |        +-------------------------+
  |
  |--> else
  |
  +------> try again with wildcard cmd
```

```
+--------------------------+                                                                                   
| registerStorageFunctions | : create timer, set up fru map, register handlers for a few (net_fn, cmd) pairs   
+-|------------------------+                                                                                   
  |    +--------------+                                                                                        
  |--> | createTimers | prepare timer with callback 'writeFru'                                                 
  |    +--------------+                                                                                        
  |    +------------+                                                                                          
  |--> | startMatch | prepare callbacks for interface change, get all fru objects and set up 'deviceHashes' map
  |    +------------+                                                                                          
  |    +-----------------------+                                               +------------------------------+
  |--> | ipmi::registerHandler | prepare handler for net_fn = 0x0a, cmd = 0x10 | ipmiStorageGetFruInvAreaInfo |
  |    +-----------------------+                                               +------------------------------+
  |    +-----------------------+                                               +------------------------+      
  |--> | ipmi::registerHandler | prepare handler for net_fn = 0x0a, cmd = 0x11 | ipmiStorageReadFruData |      
  |    +-----------------------+                                               +------------------------+      
  |    +-----------------------+                                               +-------------------------+     
  |--> | ipmi::registerHandler | prepare handler for net_fn = 0x0a, cmd = 0x12 | ipmiStorageWriteFruData |     
  |    +-----------------------+                                               +-------------------------+     
  |    +-----------------------+                                               +-----------------------+       
  |--> | ipmi::registerHandler | prepare handler for net_fn = 0x0a, cmd = 0x40 | ipmiStorageGetSELInfo |       
  |    +-----------------------+                                               +-----------------------+       
  |    +-----------------------+                                               +------------------------+      
  |--> | ipmi::registerHandler | prepare handler for net_fn = 0x0a, cmd = 0x43 | ipmiStorageGetSELEntry |      
  |    +-----------------------+                                               +------------------------+      
  |    +-----------------------+                                               +------------------------+      
  |--> | ipmi::registerHandler | prepare handler for net_fn = 0x0a, cmd = 0x44 | ipmiStorageAddSELEntry |      
  |    +-----------------------+                                               +------------------------+      
  |    +-----------------------+                                               +---------------------+         
  |--> | ipmi::registerHandler | prepare handler for net_fn = 0x0a, cmd = 0x47 | ipmiStorageClearSEL |         
  |    +-----------------------+                                               +---------------------+         
  |    +-----------------------+                                               +-----------------------+       
  |--> | ipmi::registerHandler | prepare handler for net_fn = 0x0a, cmd = 0x48 | ipmiStorageGetSELTime |       
  |    +-----------------------+                                               +-----------------------+       
  |    +-----------------------+                                               +-----------------------+       
  +--> | ipmi::registerHandler | prepare handler for net_fn = 0x0a, cmd = 0x49 | ipmiStorageSetSELTime |       
       +-----------------------+                                               +-----------------------+       
```

```
+--------------+                                                  
| createTimers | : prepare timer with callback 'writeFru'         
+-|------------+                                                  
  |                                                               
  +--> prepare write_timer with callback                          
          +------------------------------------------------------+
          |+----------+                                          |
          || writeFru | ask fru_device service to help write fru |
          |+----------+                                          |
          +------------------------------------------------------+
```

```
+----------+                                                   
| writeFru | : ask fru_device service to help write fru        
+-|--------+                                                   
  |    +----------+                                            
  |--> | getSdBus | get system dbus                            
  |    +----------+                                            
  |                                                            
  |--> ->new_method_call "xyz.openbmc_project.FruDevice"       
  |                      "/xyz/openbmc_project/FruDevice"      
  |                      "xyz.openbmc_project.FruDeviceManager"
  |                      "WriteFru"                            
  |                                                            
  |--> ->call                                                  
  |                                                            
  +--> reset bus and addr                                      
```

```
+-------------------+                                    
| recalculateHashes | : set up 'deviceHashes' map        
+-|-----------------+                                    
  |                                                      
  |--> clear 'deviceHashes'                              
  |                                                      
  |--> for each fru object in service                    
  |                                                      
  |------> find interface "xyz.openbmc_project.FruDevice"
  |                                                      
  |------> get bus and addr                              
  |                                                      
  |------> get chassis_type if it exists                 
  |                                                      
  |------> ensure fru_hash is valid (neither 0 nor 0xff) 
  |                                                      
  +------> save (fru_hash, dev) in 'deviceHashes'        
```

```
+------------+                                                                                            
| startMatch | : prepare callbacks for interface change, get all fru objects and set up 'deviceHashes' map
+-|----------+                                                                                            
  |                                                                                                       
  |--> return if 'fruMatches' is already set                                                              
  |                                                                                                       
  |--> prepare match rule (InterfacesAdded) and callback, append to 'fruMatches'                          
  |       +------------------------------------------------+                                              
  |       |read msg and check if it's our target           |                                              
  |       |                                                |                                              
  |       |+-------------------+                           |                                              
  |       || writeFruIfRunning | stop timer to write fru   |                                              
  |       |+-------------------+                           |                                              
  |       |                                                |                                              
  |       |frus[path] = object                             |                                              
  |       |                                                |                                              
  |       |+-------------------+                           |                                              
  |       || recalculateHashes | set up deviceHashes 'map' |                                              
  |       |+-------------------+                           |                                              
  |       +------------------------------------------------+                                              
  |                                                                                                       
  |--> prepare match rule (InterfacesRemoved) and callback, append to 'fruMatches'                        
  |       +------------------------------------------------+                                              
  |       |read msg and check if it's our target           |                                              
  |       |                                                |                                              
  |       |+-------------------+                           |                                              
  |       || writeFruIfRunning | stop timer to write fru   |                                              
  |       |+-------------------+                           |                                              
  |       |                                                |                                              
  |       |remove (path, object) from 'frus'               |                                              
  |       |                                                |                                              
  |       |+-------------------+                           |                                              
  |       || recalculateHashes | set up deviceHashes 'map' |                                              
  |       |+-------------------+                           |                                              
  |       +------------------------------------------------+                                              
  |                                                                                                       
  |    +-----------------+                                                                                
  +--> | replaceCacheFru | get all fru objects and set up 'deviceHashes' map                              
       +-----------------+                                                                                
```

```
+-----------------+                                                    
| replaceCacheFru | : get all fru objects and set up 'deviceHashes' map
+-|---------------+                                                    
  |                                                                    
  |--> ->yield_method_call "xyz.openbmc_project.FruDevice"             
  |                        "/"                                         
  |                        "org.freedesktop.DBus.ObjectManager"        
  |                        "GetManagedObjects"                         
  |                                                                    
  |    +-------------------+                                           
  +--> | recalculateHashes | set up 'deviceHashes' map                 
       +-------------------+                                           
```

```
+-----------------------+                                                                          
| ipmi::registerHandler | : prepare handler and register to handlerMap                             
+-|---------------------+                                                                          
  |    +-------------------+                                                                       
  |--> | ipmi::makeHandler | alloc handler                                                         
  |    +-------------------+                                                                       
  |    +-----------------------+                                                                   
  +--> | impl::registerHandler | encode net_fn & cmd into key, ensure handlerMap[key] has a handler
       +-----------------------+                                                                   
```

```
+-----------------------+                                                                  
| impl::registerHandler | : encode net_fn & cmd into key, ensure handler[key] has a handler
+-|---------------------+                                                                  
  |                                                                                        
  |--> check if net_fn# is valid                                                           
  |                                                                                        
  |    +------------+                                                                      
  |--> | makeCmdKey | encode net_fs# and cmd# into key                                     
  |    +------------+                                                                      
  |                                                                                        
  |--> check if the key is in 'handlerMap' already                                         
  |                                                                                        
  |--> if it doesn't exist yet || we have handler with higher priority                     
  |                                                                                        
  +------> handlerMap[key] = handler                                                       
```

```
+-------------------------+                                                                                               
| ipmiStorageWriteFruData | : read fru file to cache (memory), overwrite specified range, ask service to write back (file)
+-|-----------------------+                                                                                               
  |    +--------+                                                                                                         
  |--> | getFru | get (raw) fru                                                                                           
  |    +--------+                                                                                                         
  |                                                                                                                       
  |--> resize fru cache if it's larger than expected                                                                      
  |                                                                                                                       
  |--> copy fru_cache to arg data_to_write                                                                                
  |                                                                                                                       
  |--> determine if we are at the end                                                                                     
  |                                                                                                                       
  |--> if yes                                                                                                             
  |                                                                                                                       
  |------> stop timer                                                                                                     
  |                                                                                                                       
  |        +----------+                                                                                                   
  |------> | writeFru | ask fru_device service to help write fru                                                          
  |        +----------+                                                                                                   
  |                                                                                                                       
  |------> count_written = fru_cache size                                                                                 
  |                                                                                                                       
  |--> else                                                                                                               
  |                                                                                                                       
  |------> start timer (ask fru_device service to help write fru)                                                         
  |                                                                                                                       
  +------> count_written = 0                                                                                              
```

```
+--------+                                                       
| getFru | : get (raw) fru                                       
+-|------+                                                       
  |                                                              
  |--> clear fru cache                                           
  |                                                              
  |--> ->yield_method_call "xyz.openbmc_project.FruDevice"       
  |                        "/xyz/openbmc_project/FruDevice"      
  |                        "xyz.openbmc_project.FruDeviceManager"
  |                        "GetRawFru"                           
  |                                                              
  +--> udpate 'lastDevId'                                        
```
