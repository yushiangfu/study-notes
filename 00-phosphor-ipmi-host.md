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
