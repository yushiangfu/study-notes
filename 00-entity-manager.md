## Index

- [Introduction](#introduction)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

(TBD)

### entity-manager

```
+---------------------------+                                                           
| propertiesChangedCallback | : callback of property change                             
+------|--------------------+                                                           
       |                                                                                
       |--> set a timer expiring after 5 sec                                            
       |                                                                                
       +--> timer async wait                                                            
               +-----------------------------------------------------------------------+
               |if we receive a new request while handling an old one                  |
               |                                                                       |
               |    +---------------------------+                                      |
               |--> | propertiesChangedCallback | recursive                            |
               |    +---------------------------+                                      |
               |                                                                       |
               |--> return                                                             |
               |                                                                       |
               |in_progress = true                                                     |
               |                                                                       |
               |+-------------------+                                                  |
               ||loadConfigurations | for each found json: parse and add to arg 'config'
               |+-------------------+                                                  |
               |                                                                       |
               |prepare 'perform scan'                                                 |
               |   +------------------------------------------------------------------+|
               |   |for each (name, dev) in missing_config                            ||
               |   |                                                                  ||
               |   |    +--------------------+                                        ||
               |   |    | pruneConfiguration | release configuration                  ||
               |   |    +--------------------+                                        ||
               |   |+------------------------+                                        ||
               |   || deriveNewConfiguration | ensure new configuration has no old one||
               |   |+------------------------+                                        ||
               |   |+----------------+                                                ||
               |   || logDeviceAdded | add dev log                                    ||
               |   |+----------------+                                                ||
               |   |+---------+                                                       ||
               |   || io.post | ???                                                   ||
               |   |+---------+                                                       ||
               |   +------------------------------------------------------------------+|
               |                                                                       |
               |run the perf scan                                                      |
               +-----------------------------------------------------------------------+
```

```
+--------------------+                                                            
| loadConfigurations | : for each found json: parse and add to arg 'config'       
+----|---------------+                                                            
     |    +-----------+                                                           
     |--> | findFiles | find matched file path and add to arg list                
     |    +-----------+ PACKAGE_DIR="/usr/share/entity-manager/                   
     |                  SYSCONF_DIR="/etc/entity-manager/                         
     |                                                                            
     |--> for each json in 'found jsons'                                          
     |                                                                            
     |        +-----------------------+                                           
     |------> | nlohmann::json::parse |                                           
     |        +-----------------------+                                           
     |                                                                            
     |------> if parsed result is an array                                        
     |                                                                            
     |----------> add each element to arg 'config' separately                     
     |                                                                            
     |------> else                                                                
     |                                                                            
     +----------> add result to arg 'config'                                      
```

```
+--------------------+                                                          
| pruneConfiguration | : release configuration                                  
+----|---------------+                                                          
     |    +-----------------------+                                             
     |--> | deviceRequiresPowerOn | check if dev needs power on or post complete
     |    +-----------------------+                                             
     |                                                                          
     |--> return if it's power off && dev needs power on                        
     |                                                                          
     |    +---------------------+                                               
     |--> | getDeviceInterfaces | get device interfaces                         
     |    +---------------------+                                               
     |                                                                          
     |--> for each interface                                                    
     |                                                                          
     |------> remove interface from obj server if share ptr exists (?)          
     |                                                                          
     |    +---------------------------+                                         
     |--> | systemConfiguration.erase | ???                                     
     |    +---------------------------+                                         
     |    +------------------+                                                  
     +--> | logDeviceRemoved | remove dev log                                   
          +------------------+                                                  
```
  
```
+------------------+                                   
| logDeviceRemoved | : remove dev log                  
+----|-------------+                                   
     |    +------------------+                         
     |--> | deviceHasLogging | check if dev has logging
     |    +------------------+                         
     |                                                 
     |--> return if not                                
     |                                                 
     |--> find 'type' and 'asset' from record          
     |                                                 
     |--> find 'model' and 'serial number'             
     |                                                 
     |    +----------------+                           
     +--> | sd_jounal_send |                           
          +----------------+                           
```

### fru-device

```
FruDevice.cpp
+------+
| main |
+-|----+
  |    +-----------+
  |--> | findFiles | find matched file path (i2c devices) and add to arg list
  |    +-----------+
  |    +---------------+
  |--> | loadBlacklist | (skip)
  |    +---------------+
  |
  |--> ->request_name   service "xyz.openbmc_project.FruDevice"
  |
  |--> .add_interface   add interface "xyz.openbmc_project.FruDeviceManager" to obj "/xyz/openbmc_project/FruDevice"
  |
  |--> ->register_method() for 'rescan'
  |      +----------------------------------------------------------+
  |      |+--------------+                                          |
  |      || rescanBusses | scan i2c busses and register fru to dbus |
  |      |+--------------+                                          |
  |      +----------------------------------------------------------+
  |
  |--> ->register_method() * n, for 'rescan bus', 'get raw fru', 'write fru'
  |
  |-->  prepare event handler
  |        +--------------------------+
  |        |rescan busses if power on |
  |        +--------------------------+
  |
  |-->  register the handler to dbus?
  |
  |-->  prepare monitor for new i2c dev
  |        +----------------+
  |        |+--------------+|
  |        || rescanOneBus ||
  |        |+--------------+|
  |        +----------------+
  |
  |--> register the monitor callback?
  |
  |    +--------------+
  |--> | rescanBusses | scan i2c busses and register fru to dbus
  |    +--------------+
  +--> io.run()
```

```
+-----------+                                             
| findFiles | : find matched file path and add to arg list
+--|--------+                                             
   |                                                      
   |--> for each dir in arg                               
   |                                                      
   |------> for each file in dir                          
   |                                                      
   |----------> if match, add file path to local map      
   |                                                      
   |--> for each (key, value) in map                      
   |                                                      
   +------> add to arg 'found paths'                      
```

```
+--------------+
| rescanBusses | : scan i2c busses and register fru to dbus
+---|----------+
    |
    |--> set 1-second expirationfor the timer
    |
    |    +-------------+
    +--> | .async_wait |
         +-------------+
             +-------------------------------------------------------------------------------------------+
             |+-------------------+                                                                      |
             || getI2cDevicePaths | iterate target dir and add (bus, path) of i2c file to arg 'bus paths'|
             |+-------------------+                                                                      |
             |                                                                                           |
             |for each path in 'bus paths': add to local 'i2c buses'                                     |
             |                                                                                           |
             |clear global 'found devices'                                                               |
             |                                                                                           |
             |prepare 'find devices with callback'                                                       |
             |   +--------------------------------------------------------------+                        |
             |   |+------------------+                                          |                        |
             |   || readBaseboardFRU | read in "/etc/fru/baseboard.fru.bin"     |                        |
             |   |+------------------+                                          |                        |
             |   |                                                              |                        |
             |   |for each device map in bus map                                |                        |
             |   |                                                              |                        |
             |   +--> for each device in device map                             |                        |
             |   |                                                              |                        |
             |   |        +--------------------+                                |                        |
             |   +------> | addFruObjectToDbus | register fru properties to dbus|                        |
             |   |        +--------------------+                                |                        |
             |   +--------------------------------------------------------------+                        |
             |                                                                                           |
             |->run(), it calls findI2CDevices()                                                           |
             +-------------------------------------------------------------------------------------------+
```

```
+-------------------+                                                                        
| getI2cDevicePaths | : iterate target dir and add (bus, path) of i2c file to arg 'bus paths'
+----|--------------+                                                                        
     |                                                                                       
     |--> for each entry under arg 'dir path'                                                
     |                                                                                       
     |------> if it's a i2c device                                                           
     |                                                                                       
     +----------> add the (bus, path) to arg 'bus paths'                                     
```

```
+------------------+                                          
| readBaseboardFRU | : read in "/etc/fru/baseboard.fru.bin"   
+----|-------------+                                          
     |                                                        
     |--> have an ifstream of "/etc/fru/baseboard.fru.bin"    
     |                                                        
     |--> prepare baseboardFRUFile (can't find its definition)
     |                                                        
     |--> if it's good (what's good?)                         
     |                                                        
     +------> read data to baseboardFRUFile                   
```

```
+--------------------+                                                                    
| addFruObjectToDbus | : register fru properties to dbus                                  
+----|---------------+                                                                    
     |    +---------------+                                                               
     |--> | formatIPMIFRU | given fru, format (key, value) pairs and save in arg 'result' 
     |    +---------------+                                                               
     |                                                                                    
     |--> decide 'product name'                                                           
     |                                                                                    
     |--> for each bus-map in dbus-interface-map                                          
     |                                                                                    
     |------> if it's a mux                                                               
     |                                                                                    
     |----------> return if the device is already added                                   
     |                                                                                    
     |------> found = true, continue to see if there's higher match                       
     |                                                                                    
     |--> if found                                                                        
     |                                                                                    
     |------> add suffix to 'product name'                                                
     |                                                                                    
     |--> map[(bus, addr)] = interface                                                    
     |                                                                                    
     |--> for each property in formatted-fru                                              
     |                                                                                    
     |------> if it's 'asset tag'                                                         
     |                                                                                    
     |----------> call ->register_property()
     |                                                                                    
     |                +------------------------------------------------------------------+
     |                |+-------------------+                                             |
     |                || updateFRUProperty | update fru property, write back to somewhere|
     |                |+-------------------+                                             |
     |                +------------------------------------------------------------------+
     |                                                                    
     |--> ->register_property (bus)                                                       
     |                                                                                    
     +--> ->register_property (addr)                                                      
```

```
+---------------+                                                                
| formatIPMIFRU | : given fru, format (key, value) pairs and save in arg 'result'
+---|-----------+                                                                
    |                                                                            
    |--> for each fru area                                                       
    |                                                                            
    |        +--------------+                                                    
    |------> | verifyOffset | (skip)                                             
    |        +--------------+                                                    
    |                                                                            
    |------> switch area                                                         
    |                                                                            
    |------> case chassis                                                        
    |                                                                            
    |----------> set result["CHASSIS_TYPE"]                                      
    |                                                                            
    |----------> use 'chassisFruAreas' for field names                           
    |                                                                            
    |------> case board                                                          
    |                                                                            
    |----------> set result["BOARD_LANGUAGE_CODE"]                               
    |                                                                            
    |----------> set result["BOARD_MANUFACTURE_DATE"]                            
    |                                                                            
    |------> case product                                                        
    |                                                                            
    |----------> set result["PRODUCT_LANGUAGE_CODE"]                             
    |                                                                            
    |------> while decode state is ok                                            
    |                                                                            
    |            +---------------+                                               
    |----------> | decodeFRUData | given type, decode fru data into string       
    |            +---------------+                                               
    |                                                                            
    |----------> decide 'name'                                                   
    |                                                                            
    +----------> set result[name]                                                
```

```
+---------------+                                          
| decodeFRUData | : given type, decode fru data into string
+---|-----------+                                          
    |                                                      
    |--> if iter reaches the end                           
    |                                                      
    |------> return pair(decode_state, string)             
    |                                                      
    |--> decode type and length                            
    |                                                      
    |--> switch type                                       
    |                                                      
    |--> case binary                                       
    |                                                      
    |------> prepare string   
    |
    |--> case language dependent
    |                                                      
    |------> shift iter
    |                                                      
    |--> case bcd plus                                     
    |                                                      
    |------> prepare string                                
    |                                                      
    |--> case six bit ascii                                
    |                                                      
    +------> prepare string                                
```

```
+-------------------+                                                                                      
| updateFRUProperty | : update fru property, write back to somewhere                                       
+----|--------------+                                                                                      
     |    +------------+                                                                                   
     |--> | getFRUInfo | given (bus, addr), get fru info from map                                          
     |    +------------+                                                                                   
     |                                                                                    
     |--> write len and data of requested property                                                         
     |                                                         ENABLE_FRU_AREA_RESIZE isn't defined        
     |--> copy remaining data to main fru area                                                             
     |                                                                                                     
     |    +-----------------------------+                                                                  
     |--> | updateFRUAreaLenAndChecksum | update the final area len and csum                               
     |    +-----------------------------+                                                                  
     |    +----------+                                                                                     
     |--> | writeFRU | given (bus, addr), write fru data accordingly                                       
     |    +----------+                                                                                     
     |    +--------------+                                                                                 
     +--> | rescanBusses | recursive?                                                                      
          +--------------+                                                                                 
```

```
+----------+                                                                      
| writeFRU |  : given (bus, addr), write fru data accordingly                     
+--|-------+                                                                      
   |    +---------------+                                                             
   |--> | formatIPMIFRU | verify if fru format is valid by parsing it
   |    +---------------+                                                             
   |                                                                              
   |--> if bus is 0 && addr is 0 (baseboard fru)                                  
   |                                                                              
   |------> write data to "/etc/fru/baseboard.fru.bin"                      
   |                                                                              
   +------> return                                                                
   |                                                                              
   |--> if that (bus, addr) has a eeprom                                          
   |                                                                              
   |------> write fru to it                                                       
   |                                                                              
   +------> return                                                                
   |                                                                              
   |    +-------+                                                                 
   |--> | ioctl | set slave                                                       
   |    +-------+                                                                 
   |                                                                              
   |--> while the write hans't completed yet                                      
   |                                                                              
   |        +-------+                                                             
   |------> | ioctl | set slave                                                   
   |        +-------+                                                             
   |        +---------------------------+                                         
   +------> | i2c_smbus_write_byte_data | write data to slave through i2c protocol
            +---------------------------+                                         
```

```
+----------------+                                                                
| findI2CDevices | : given bus range and addr range, try read fru from each slave 
+---|------------+                                                                
    |                                                                             
    |--> for each bus in arg 'buses'                                              
    |                                                                             
    |        +------------+                                                       
    |------> | getRootBus | get root adapter                                      
    |        +------------+                                                       
    |                                                                             
    |------> continue if the root bus is in blacklist                             
    |                                                                             
    |        +------------+                                                       
    +------> | getBusFRUs | given bus and addr range, try read fru from each slave
             +------------+                                                       
```

```
+------------+                                                                
| getBusFRUs | : given bus and addr range, try read fru from each slave       
+--|---------+                                                                
   |                                                                          
   |--> prepare function 'future'                                             
   |       +-----------------------------------------------------------------+
   |       |+----------------+                                               |
   |       || findI2CEeproms | （skip)                                        |
   |       |+----------------+                                               |
   |       |                                                                 |
   |       |for each slave in range [first, last]                            |
   |       |                                                                 |
   |       |    +-------+                                                    |
   |       +--> | ioctl | set slave                                          |
   |       |    +-------+                                                    |
   |       |    +---------------------+                                      |
   |       +--> | i2c_smbus_read_byte | probe                                |
   |       |    +---------------------+                                      |
   |       |    +--------------------+                                       |
   |       +--> | makeProbeInterface | for (bus, addr), add interface to dbus|
   |       |    +--------------------+                                       |
   |       |    +-----------------+                                          |
   |       +--> | readFRUContents | read fru contents                        |
   |       |    +-----------------+                                          |
   |       +-----------------------------------------------------------------+
   |                                                                          
   +--> fugure.get()                                                          
```

```
+-----------------+                                          
| readFRUContents | : read fru contents                      
+----|------------+                                          
     |                                                       
     |--> return if we fail to find the fru header           
     |                                                       
     +--> for each fru area                                  
     |                                                       
     +------> read data, e.g., through i2c protocol          
     |                                                       
     |--> if the fru has multi records                       
     |                                                       
     |------> read data, e.g., through i2c protocol          
     |                                                       
     +--> read the remaining data, e.g., through i2c protocol
```

## <a name="reference"></a> Reference

(TBD)
