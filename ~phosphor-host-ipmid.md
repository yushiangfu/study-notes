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

### registerStorageFunctions()

| NetFn + Cmd | Callback                        | Note |
| ---         | ---                             | ---  |
| 0x04 0x02   | ipmiSenPlatformEvent            |      |
| 0x04 0x30   | ipmiSetSensorReading            |      |
| 0x04 0x2D   | ipmiSenGetSensorReading         |      |
| 0x04 0x27   | ipmiSenGetSensorThresholds      |      |
| 0x04 0x26   | ipmiSenSetSensorThresholds      |      |
| 0x04 0x29   | ipmiSenGetSensorEventEnable     |      |
| 0x04 0x2B   | ipmiSenGetSensorEventStatus     |      |
| 0x0A 0x20   | ipmiStorageGetSDRRepositoryInfo |      |
| 0x04 0x20   | ipmiSensorGetDeviceSdrInfo      |      |
| 0x0A 0x21   | ipmiStorageGetSDRAllocationInfo |      |
| 0x04 0x22   | ipmiStorageReserveSDR           |      |
| 0x0A 0x22   | ipmiStorageReserveSDR           |      |
| 0x04 0x21   | ipmiStorageGetSDR               |      |
| 0x0A 0x23   | ipmiStorageGetSDR               |      |
| 0xDC 0x07   | getSensorInfo                   |      |

### registerStorageFunctions()

| NetFn + Cmd | Callback                     | Note |
| ---         | ---                          | ---  |
| 0x0A 0x10   | ipmiStorageGetFruInvAreaInfo |      |
| 0x0A 0x11   | ipmiStorageReadFruData       |      |
| 0x0A 0x12   | ipmiStorageWriteFruData      |      |
| 0x0A 0x40   | ipmiStorageGetSELInfo        |      |
| 0x0A 0x43   | ipmiStorageGetSELEntry       |      |
| 0x0A 0x44   | ipmiStorageAddSELEntry       |      |
| 0x0A 0x47   | ipmiStorageClearSEL          |      |
| 0x0A 0x48   | ipmiStorageGetSELTime        |      |
| 0x0A 0x49   | ipmiStorageSetSELTime        |      |

### registerChannelFunctions()

| NetFn + Cmd | Callback                     | Note |
| ---         | ---                          | ---  |
| 0x06 0x40   | ipmiSetChannelAccess         |      |
| 0x06 0x41   | ipmiGetChannelAccess         |      |
| 0x06 0x42   | ipmiGetChannelInfo           |      |
| 0x06 0x4E   | ipmiGetChannelPayloadSupport |      |
| 0x06 0x4F   | ipmiGetChannelPayloadVersion |      |

### registerUserIpmiFunctions()

| NetFn + Cmd | Callback                                 | Note |
| ---         | ---                                      | ---  |
| 0x06 0x43   | ipmiSetUserAccess                        |      |
| 0x06 0x44   | ipmiGetUserAccess                        |      |
| 0x06 0x46   | ipmiGetUserName                          |      |
| 0x06 0x45   | ipmiSetUserName                          |      |
| 0x06 0x47   | ipmiSetUserPassword                      |      |
| 0x06 0x38   | ipmiGetChannelAuthenticationCapabilities |      |
| 0x06 0x4C   | ipmiSetUserPayloadAccess                 |      |
| 0x06 0x4D   | ipmiGetUserPayloadAccess                 |      |

### register_netfn_app_functions()

| NetFn + Cmd | Callback                  | Note |
| ---         | ---                       | ---  |
| 0x06 0x01   | ipmiAppGetDeviceId        |      |
| 0x06 0x36   | ipmiAppGetBtCapabilities  |      |
| 0x06 0x22   | ipmiAppResetWatchdogTimer |      |
| 0x06 0x3D   | ipmiAppGetSessionInfo     |      |
| 0x06 0x24   | ipmiSetWatchdogTimer      |      |
| 0x06 0x3C   | ipmiAppCloseSession       |      |
| 0x06 0x25   | ipmiGetWatchdogTimer      |      |
| 0x06 0x04   | ipmiAppGetSelfTestResults |      |
| 0x06 0x08   | ipmiAppGetDeviceGuid      |      |
| 0x06 0x06   | ipmiSetAcpiPowerState     |      |
| 0x06 0x07   | ipmiGetAcpiPowerState     |      |
| 0x06 0x52   | ipmiMasterWriteRead       |      |
| 0x06 0x37   | ipmiAppGetSystemGuid      |      |
| 0x06 0x54   | getChannelCipherSuites    |      |
| 0x06 0x59   | ipmiAppGetSystemInfo      |      |
| 0x06 0x58   | ipmiAppSetSystemInfo      |      |

### register_netfn_chassis_functions()

| NetFn + Cmd | Callback                         | Note |
| ---         | ---                              | ---  |
| 0x00 0x00   | ipmiGetChassisCap                |      |
| 0x00 0x0A   | ipmiSetFrontPanelButtonEnables   |      |
| 0x00 0x05   | ipmiSetChassisCap                |      |
| 0x00 0x09   | ipmiChassisGetSysBootOptions     |      |
| 0x00 0x01   | ipmiGetChassisStatus             |      |
| 0x00 0x07   | ipmiGetSystemRestartCause        |      |
| 0x00 0x02   | ipmiChassisControl               |      |
| 0x00 0x04   | ipmiChassisIdentify              |      |
| 0x00 0x08   | ipmiChassisSetSysBootOptions     |      |
| 0x00 0x0F   | ipmiGetPOHCounter                |      |
| 0x00 0x06   | ipmiChassisSetPowerRestorePolicy |      |

### register_netfn_dcmi_functions()

| NetFn + Cmd | Callback            | Note |
| ---         | ---                 | ---  |
| 0x2C 0x03   | getPowerLimit       |      |
| 0x2C 0x04   | setPowerLimit       |      |
| 0x2C 0x05   | applyPowerLimit     |      |
| 0x2C 0x06   | getAssetTag         |      |
| 0x2C 0x08   | setAssetTag         |      |
| 0x2C 0x09   | getMgmntCtrlIdStr   |      |
| 0x2C 0x0A   | setMgmntCtrlIdStr   |      |
| 0x2C 0x01   | getDCMICapabilities |      |
| 0x2C 0x10   | getTempReadings     |      |
| 0x2C 0x02   | getPowerReading     |      |
| 0x2C 0x07   | getSensorInfo       |      |
| 0x2C 0x13   | getDCMIConfParams   |      |
| 0x2C 0x12   | setDCMIConfParams   |      |

### register_netfn_global_functions()

| NetFn + Cmd | Callback        | Note |
| ---         | ---             | ---  |
| 0x06 0x02   | ipmiGlobalReset |      |

### register_netfn_groupext_functions()

| NetFn + Cmd | Callback      | Note |
| ---         | ---           | ---  |
| 0x2C 0x00   | ipmi_groupext |      |

### register_netfn_sen_functions()

| NetFn + Cmd | Callback                      | Note |
| ---         | ---                           | ---  |
| 0x04 0x30   | ipmiSetSensorReading          |      |
| 0x04 0x2D   | ipmiSensorGetSensorReading    |      |
| 0x04 0x22   | ipmiSensorReserveSdr          |      |
| 0x04 0x20   | ipmiSensorGetDeviceSdrInfo    |      |
| 0x04 0x27   | ipmiSensorGetSensorThresholds |      |
| 0x04 0x26   | ipmiSenSetSensorThresholds    |      |
| 0x04 0x21   | ipmi_sen_get_sdr              |      |
| 0x04 0x02   | ipmicmdPlatformEvent          |      |
| 0x04 0x2F   | ipmiGetSensorType             |      |

### register_netfn_storage_functions()

| NetFn + Cmd | Callback                     | Note |
| ---         | ---                          | ---  |
| 0x0A 0x40   | ipmiStorageGetSelInfo        |      |
| 0x0A 0x48   | ipmiStorageGetSelTime        |      |
| 0x0A 0x49   | ipmiStorageSetSelTime        |      |
| 0x0A 0x43   | getSELEntry                  |      |
| 0x0A 0x46   | deleteSELEntry               |      |
| 0x0A 0x44   | ipmiStorageAddSEL            |      |
| 0x0A 0x47   | clearSEL                     |      |
| 0x0A 0x10   | ipmiStorageGetFruInvAreaInfo |      |
| 0x0A 0x11   | ipmiStorageReadFruData       |      |
| 0x0A 0x20   | ipmiGetRepositoryInfo        |      |
| 0x0A 0x22   | ipmiSensorReserveSdr         |      |
| 0x0A 0x23   | ipmi_sen_get_sdr             |      |
| 0x0A 0x42   | ipmiStorageReserveSel        |      |

### register_netfn_app_functions()

| NetFn + Cmd | Callback                  | Note |
| ---         | ---                       | ---  |
| 0x06 0x35   | ipmi_app_read_event       |      |
| 0x06 0x2E   | ipmiAppSetBMCGlobalEnable |      |
| 0x06 0x2F   | ipmiAppGetBMCGlobalEnable |      |
| 0x06 0x31   | ipmiAppGetMessageFlags    |      |

### register_netfn_transport_functions()

| NetFn + Cmd | Callback         | Note |
| ---         | ---              | ---  |
| 0x0C 0x01   | setLan           |      |
| 0x0C 0x02   | getLan           |      |
| 0x0C 0x21   | setSolConfParams |      |
| 0x0C 0x22   | getSolConfParams |      |

```
sensorhandler.cpp                                                                                         
+----------------------+                                                                                   
| ipmicmdPlatformEvent | : parse request data and ready argument, dbus-call service method to add sel entry
+-|--------------------+                                                                                   
  |                                                                                                        
  |--> ready request, generate id, and source path ('system' or 'ipmb')                                    
  |                                                                                                        
  |--> determine data count (1 ~ 3)                                                                        
  |                                                                                                        
  |--> ready (de)assert and event_data                                                                     
  |                                                                                                        
  |--> prepare method call                                                                                 
  |    +-------------------------------------------+                                                       
  |    |service: "xyz.openbmc_project.Logging.IPMI"|                                                       
  |    |object: "/xyz/openbmc_project/Logging/IPMI"|                                                       
  |    |iface: "xyz.openbmc_project.Logging.IPMI"  |                                                       
  |    |method: "IpmiSelAdd"                       |                                                       
  |    +-------------------------------------------+                                                       
  |                                                                                                        
  +--> call it                                                                                             
```
