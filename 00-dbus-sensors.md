## Thresholds

```
+---------------------------+
| parseThresholdsFromConfig | : given sensor data, find pairs of (hysteresis, direction, severity, value),
+------|--------------------+   and push bask to arg vector
       |
       |--> for (intf, cfg) in sensor data
       |
       |------> continue if can't find "Thresholds"
       |
       |------> ignore arg 'match label'
       |
       |------> ignore arg 'sensorr index'
       |
       |------> find "Hysteresis", "Direction", "Severity", "Value" and parse
       |
       +------> append to arg 'thresholdVector'                                                                                    
```

## Utils

```
+------------------------+
| GetSensorConfiguration |
|  --------------------  |
|    getPath()           |
|    getConfiguration()  |
|    dbusConnection      |
|    callback            |
|    respData            |
+------------------------+
```

```
+------------------------------------------+
| GetSensorConfiguration::getConfiguration | : get configuration path if it with matches any of the interfaces
+----------|-------------------------------+
           |
           |--> for each type, add to interface
           |
           |    +-----------------------------------+
           +--> | dbusConnection->async_method_call |
                +------------------------------------------+
                | for each (path, dict) in ret             |
                |                                          |
                |     for each interface in dict           |
                |                                          |
                |         find something in interface      |
                |                                          |
                |         continue if not found            |
                |                                          |
                |         +---------------+                |
                |         | self->getPath | get what path? |
                |         +---------------+                |
                +------------------------------------------+                                                     
```

```
+---------------------------------+                     
| GetSensorConfiguration::getPath | : get what path?    
+--------|------------------------+                     
         |                                              
         +--> ->async_method_call                       
                 +-----------------------------------+  
                 |->respData[path][iface] = arg data |  
                 +-----------------------------------+  
                 srv = owner                            
                 obj = path                             
                 ifc = "org.freedesktop.DBus.Properties"
                 mth = "GetAll"                         
```

```
+-----------------------------+                                                          
| setupManufacturingModeMatch | : prepare handlers for manufacturing mode match          
+-------|---------------------+                                                          
        |                                                                                
        |--> special mode intf = "xyz.openbmc_project.Security.SpecialMode"              
        |                                                                                
        |--> prepare handler for special mode intf add                                   
        |    +------------------------------------------------------------------------+  
        |    | +--------+                                                             |  
        |    | | m.read | read msg into path and interfaces                           |  
        |    | +--------+                                                             |  
        |    | +---------------------+                                                |  
        |    | | interfaceAdded.find | find special mode intf from interfaces         |  
        |    | +---------------------+                                                |  
        |    | +-------------------+                                                  |  
        |    | | propertyList.find | find "SpecialMode" from property of interface    |  
        |    | +-------------------+                                                  |  
        |    | +-------------------------+                                            |  
        |    | | handleSpecialModeChange | determine 'manufacturingMode' (global var) |  
        |    | +-------------------------+                                            |  
        |    +------------------------------------------------------------------------+  
        |                                                                                
        |-->  prepare handler for mode change                                            
        |     +-------------------------------------------------------------------------+
        |     |+--------+                                                               |
        |     || m.read | read msg into interface and property                          |
        |     |+--------+                                                               |
        |     |+------------------------+                                               |
        |     || propertiesChanged.find | find "SpecialMode" from property of interface |
        |     |+------------------------+                                               |
        |     |+-------------------------+                                              |
        |     || handleSpecialModeChange | determine 'manufacturingMode' (global var)   |
        |     |+-------------------------+                                              |
        |     +-------------------------------------------------------------------------+
        |                                                                                
        +--> prepare handler for manufacturing mode                                      
             +-----------------------------------------------------------------------+   
             |+-------------------------+                                            |   
             || handleSpecialModeChange | determine 'manufacturingMode' (global var) |   
             |+-------------------------+                                            |   
             +-----------------------------------------------------------------------+   
```

```
+-------------------------------+                                                   
| setupPropertiesChangedMatches | : for each types: prepare match and add to matches
+-------|-----------------------+                                                   
        |                                                                           
        |--> for each type in arg 'types'                                           
        |                                                                           
        |------> prepare match = (bus, string, handler)                             
        |                                                                           
        +------> add to the end of arg 'matches'                                    
```

### adcsensor

```
+------+                                              
| main | : register callbacks to create sensors       
+-|----+ ADCSensorMain.cpp                            
  |                                                   
  |--> prepare service "xyz.openbmc_project.ADCSensor"
  |                                                   
  |--> io.post                                        
  |       +---------------------------------+         
  |       |+---------------+                |         
  |       || createSensors | create sensors |         
  |       |+---------------+                |         
  |       +---------------------------------+         
  |                                                   
  |--> prepare event handler                          
  |       +------------------------------------------+
  |       |insert msg path to local 'sensor changed' |
  |       |                                          |
  |       |.async_wait                               |
  |       |   +---------------------------------+    |
  |       |   |+---------------+                |    |
  |       |   || createSensors | create sensors |    |
  |       |   |+---------------+                |    |
  |       |   +---------------------------------+    |
  |       +------------------------------------------+
  |                                                   
  |--> prepare cpu presence handler                   
  |       +------------------------------------------+
  |       |read msg to local 'values'                |
  |       |                                          |
  |       |if 'Presence' found in 'values'           |
  |       |                                          |
  |       |    add it to global 'cpuPresence'        |
  |       |                                          |
  |       |    .async_wait                           |
  |       |       +---------------------------------+|
  |       |       |+---------------+                ||
  |       |       || createSensors | create sensors ||
  |       |       |+---------------+                ||
  |       |       +---------------------------------+|
  |       +------------------------------------------+
  |                                                   
  |--> for each sensor types                          
  |                                                   
  |------> register the event handler                 
  |                                                   
  +--> register cpu presence handler                  
```

```
+---------------+                                                                                                           
| createSensors | : create sensors                                                                                          
+---|-----------+                                                                                                           
    |                                                                                                                       
    |--> prepare callback for 'sensor config getter'                                                                        
    |       +--------------------------------------------------------------------------------------------------------------+
    |       |check if "/sys/class/hwmon" has files (sensors)                                                               |
    |       |                                                                                                              |
    |       |for path of each adc sensor                                                                                   |
    |       |                                                                                                              |
    |       |--> continue if it's not "iio_hwmon"                                                                          |
    |       |                                                                                                              |
    |       |--> for sensor in configurations                                                                              |
    |       |                                                                                                              |
    |       |------> attempt to get sensor_data and iface_path of sensor                                                   |
    |       |                                                                                                              |
    |       |------> break if found                                                                                        |
    |       |                                                                                                              |
    |       |    +----------------------------+                                                                            |
    |       |--> | parseThresholdsFromConfig  | given sensor data, find pairs of (hysteresis, direction, severity, value), |
    |       |    +----------------------------+ and push bask to arg vector                                                |
    |       |                                                                                                              |
    |       |--> determine 'scale factor', 'poll rate'                                                                     |
    |       |                                                                                                              |
    |       |--> handle requirement of 'power state' and 'cpu' if specified                                                |
    |       |                                                                                                              |
    |       |--> handle gpio if needed                                                                                     |
    |       |                                                                                                              |
    |       |    +----------------------+                                                                                  |
    |       +--> | ADCSensor::setupRead | alloc buffer and handle response                                                 |
    |       |    +----------------------+                                                                                  |
    |       +--------------------------------------------------------------------------------------------------------------+
    |    +------------------------------------------+                                                                       
    +--> | GetSensorConfiguration::getConfiguration | get configuration path if it with matches any of the interfaces       
         +------------------------------------------+                                                                       
```

```
+----------------------+                                          
| ADCSensor::setupRead | : alloc buffer and handle response       
+-----|----------------+                                          
      |                                                           
      |--> alloc buffer                                           
      |                                                           
      |--> if it's the case of bridge gpio                        
      |                                                           
      |------> (skip)                                             
      |                                                           
      |--> else                                                   
      |                                                           
      +------> ::async_read_until                                 
                  +----------------------------------------------+
                  |+---------------------------+                 |
                  || ADCSensor::handleResponse | handle response |
                  |+---------------------------+                 |
                  +----------------------------------------------+
```

```
+---------------------------+                                              
| ADCSensor::handleResponse | : handle response                            
+------|--------------------+                                              
       |                                                                   
       |--> open file                                                      
       |                                                                   
       +--> .async_wait                                                    
               +----------------------------------------------------------+
               |+----------------------+                                  |
               || ADCSensor::setupRead | alloc buffer and handle response |
               |+----------------------+                                  |
               +----------------------------------------------------------+
```

### cpusensor

```
+------+
| main | : create cpu sensors
+-|----+ CPUSensorMain.cpp
  |
  |--> .async_wait
  |
  |        +--------------+
  |------> | getCpuConfig | create sensors, read peci adapter name(s) from file to cpu_configs
  |        +--------------+
  |
  |------> if it gets nothing in the above func
  |
  |            +----------------+
  +----------> | detectCpuAsync | : try to get dimm temp and cpu id, create sensors
  |            +---|------------+
  |                |
  |                +--> timer.async_wait
  |                        +--------------------------------------------------------------+
  |                        |+-----------+                                                 |
  |                        || detectCpu | try to get dimm temp and cpu id, create sensors |
  |                        |+-----------+                                                 |
  |                        +--------------------------------------------------------------+
  |
  |--> prepare event handler
  |       +-----------------------------------------------------------------------------------------+
  |       |timer.async_wait                                                                         |
  |       |   +------------------------------------------------------------------------------------+|
  |       |   |+--------------+                                                                    ||
  |       |   || getCpuConfig | create sensors, read peci adapter name(s) from file to cpu_configs ||
  |       |   |+--------------+                                                                    ||
  |       |   |+----------------+                                                                  ||
  |       |   || detectCpuAsync | try to get dimm temp and cpu id, create sensors                  ||
  |       |   |+----------------+                                                                  ||
  |       |   +------------------------------------------------------------------------------------+|
  |       +-----------------------------------------------------------------------------------------+
  |
  |--> for each sensor type
  |
  |------> register the handler
  |
  |--> ->request_name("xyz.openbmc_project.CPUSensor")
  |
  |    +-----------------------------+
  |--> | setupManufacturingModeMatch | prepare handlers for manufacturing mode match
  |    +-----------------------------+
  |    +--------+
  +--> | io.run |
       +--------+
```

```
+--------------+                                                                                                    
| getCpuConfig | : create sensors, read peci adapter name(s) from file to cpu_configs                               
+---|----------+                                                                                                    
    |                                                                                                               
    |--> for each sensor types                                                                                      
    |                                                                                                               
    |        +------------------------+                                                                             
    |------> | getSensorConfiguration | update cache if needed, find type-matched one and add to arg                
    |        +------------------------+                                                                             
    |                                                                                                               
    |--> for each sensor types                                                                                      
    |                                                                                                               
    |------> for each sensor in config                                                                              
    |                                                                                                               
    |----------> for each base_conf in sensor                                                                       
    |                                                                                                               
    |--------------> handle 'Name' from base_conf                                                                   
    |                                                                                                               
    |--------------> check if cpu is present through gpio                                                           
    |                                                                                                               
    |--------------> if inventory iface && present                                                                  
    |                                                                                                               
    |------------------> .add_interface("xyz.openbmc_project.Inventory.Item")                                       
    |                                                                                                               
    |------------------> ->register_property("PrettyName")                                                          
    |                                                                                                               
    |------------------> ->register_property("Present")                                                             
    |                                                                                                               
    |------------------> move iface to inventoryIfaces[name]                                                        
    |                                                                                                               
    |------------------> if present                                                                                 
    |                                                                                                               
    |----------------------> under /sys/, find hwmon sensor name with 'input'                                       
    |                                                                                                               
    |                        +------------------+                                                                   
    |----------------------> | createSensorName |                                                                   
    |                        +------------------+                                                                   
    |                                                                                                               
    |----------------------> create a sensor and save it in 'gCpuSensors'                                           
    |                                                                                                               
    |----------------------> find (bus, addr) from config and save them to cpu_config                               
    |                                                                                                               
    |                        +--------------------------------+                                                     
    +----------------------> | addConfigsForOtherPeciAdapters | read peci adapter name from file, add to cpu_configs
    |                        +--------------------------------+                                                     
    |                                                                                                               
    |--> if we did parse something                                                                                  
    |                                                                                                               
    +------> print 'CPU config is parsed' or 'CPU configs are parsed'                                               
```

```
+------------------------+                                                               
| getSensorConfiguration | : update cache if needed, find type-matched one and add to arg
+-----|------------------+                                                               
      |                                                                                  
      |--> if not use cache                                                              
      |                                                                                  
      |------> ->new_method_call("GetManagedObjects")                                    
      |                                                                                  
      |------> ->call(arg is the output of the above func)                               
      |                                                                                  
      |------> read reply                                                                
      |                                                                                  
      |--> for each path_pair in managed_obj                                             
      |                                                                                  
      |------> find type-matched one                                                     
      |                                                                                  
      +------> if found, add to arg 'response' and break                                 
```

```
+--------------------------------+                                                       
| addConfigsForOtherPeciAdapters | : read peci adapter name from file, add to cpu_configs
+-------|------------------------+                                                       
        |    +-----------+                                                               
        |--> | findFiles | find files with name 'peci' (peci adapter)                    
        |    +-----------+                                                               
        |                                                                                
        |--> for each peci adapter                                                       
        |                                                                                
        |        +-----------------------------+                                         
        |------> | readPeciAdapterNameFromFile | read adapter name from file             
        |        +-----------------------------+                                         
        |                                                                                
        +------> add to cpu_configs                                                      
```

```
+-----------+                                                   
| detectCpu | : try to get dimm temp and cpu id, create sensors 
+-|---------+                                                   
  |                                                             
  |--> for each cpu config                                      
  |                                                             
  |------> continue if it's aleady 'ready'                      
  |                                                             
  |------> open peci dev file                                   
  |                                                             
  |------> ioctl(ping)                                          
  |                                                             
  |------> if normal                                            
  |                                                             
  |----------> get dimm temp and set state = 'ready'            
  |                                                             
  |------> else                                                 
  |                                                             
  |----------> set state = 'off'                                
  |                                                             
  |------> if state changes                                     
  |                                                             
  +----------> if it's 'off' to 'ready' or 'on'                 
  |                                                             
  |--------------> if old state is 'off'                        
  |                                                             
  |------------------> ioctl: get cpu id                        
  |                                                             
  +--------------> determine rescan delay                       
  |                                                             
  |----------> save new state in config                         
  |                                                             
  |--> if rescan_delay is set                                   
  |                                                             
  |------> timer.async_wait                                     
  |           +---------------------------------+               
  |           |+---------------+                |               
  |           || createSensors | create sensors |               
  |           |+---------------+                |               
  |           +---------------------------------+               
  |                                                             
  |--> if not yet all cpu are pinged                            
  |                                                             
  |        +----------------+                                   
  +------> | detectCpuAsync | recursive call                    
           +----------------+                                   
```

### exitairtempsensor

```
ExitAirTempSensor.cpp                                                              
+------+                                                                            
| main | : create sensors and update sensor reading                                 
+-|----+                                                                            
  |                                                                                 
  |--> ->request_name("xyz.openbmc_project.ExitAirTempSensor")                      
  |                                                                                 
  |--> io.post                                                                      
  |                                                                                 
  |        +--------------+                                                         
  |------> | createSensor | create sensors                                          
  |        +--------------+                                                         
  |                                                                                 
  |--> prepare event handler                                                        
  |       +-------------------------------------+                                   
  |       |.async_wait                          |                                   
  |       |   +--------------------------------+|                                   
  |       |   |+--------------+                ||                                   
  |       |   || createSensor | create sensors ||                                   
  |       |   |+--------------+                ||                                   
  |       |   +--------------------------------+|                                   
  |       +-------------------------------------+                                   
  |                                                                                 
  |--> for each monitor interfaces                                                  
  |                                                                                 
  |------> prepare match rule and register the event handler                        
  |                                                                                 
  |    +-----------------------------+                                              
  +--> | setupManufacturingModeMatch | prepare handlers for manufacturing mode match
       +-----------------------------+                                              
```

```
+--------------+                                                                                                                 
| createSensor | : create sensors                                                                                                
+-|------------+                                                                                                                 
  |                                                                                                                              
  |--> prepare a GetSensorConfiguration with callback                                                                            
  |       +---------------------------------------------------------------------------------------------------------------------+
  |       |for each pair in response                                                                                            |
  |       |                                                                                                                     |
  |       |--> for each entry in the pair                                                                                       |
  |       |                                                                                                                     |
  |       +------> if the entry is a exit_air_iface                                                                             |
  |       |                                                                                                                     |
  |       |            +---------------------------+                                                                            |
  |       +----------> | parseThresholdsFromConfig | given sensor data, find pairs of (hysteresis, direction, severity, value), |
  |       |            +---------------------------+ and push bask to arg vector                                                |
  |       |                                                                                                                     |
  |       +----------> set up threshold                                                                                         |
  |       |                                                                                                                     |
  |       +------> elif it's a cfm_iface                                                                                        |
  |       |                                                                                                                     |
  |       |            +---------------------------+                                                                            |
  |       +----------> | parseThresholdsFromConfig | given sensor data, find pairs of (hysteresis, direction, severity, value), |
  |       |            +---------------------------+ and push bask to arg vector                                                |
  |       |                                                                                                                     |
  |       +----------> set up threshold                                                                                         |
  |       |                                                                                                                     |
  |       |if there's a exit_air_sensor                                                                                         |
  |       |                                                                                                                     |
  |       |    +-------------------------+                                                                                      |
  |       +--> | CFMSensor::setupMatches | set match rule and register callback to update sensor reading                        |
  |       |    +-------------------------+                                                                                      |
  |       |    +-------------------------+                                                                                      |
  |       +--> | CFMSensor::updateReading| update sensor reading (even it's nan)                                                |
  |       |    +-------------------------+                                                                                      |
  |       +---------------------------------------------------------------------------------------------------------------------+
  |    +------------------------------------------+                                                                              
  +--> | GetSensorConfiguration::getConfiguration | get configuration path if it with matches any of the interfaces              
       +------------------------------------------+                                                                         
```

```
+-------------------------+                                                                                           
| CFMSensor::setupMatches | : set match rule and register callback to update sensor reading                           
+-|-----------------------+                                                                                           
  |    +------------------+                                                                                           
  |--> | setupSensorMatch | prepare event handler calling arg callback, register to 'xyz.openbmc_project.Sensor.Value'
  |    +------------------+                                                                                           
  |        +--------------------------------------------------------------------------+                               
  |        |if tach_path isn't in range yet                                           |                               
  |        |                                                                          |                               
  |        |    +--------------------------+                                          |                               
  |        +--> | CFMSensor::addTachRanges | add tach range and update sensor reading |                               
  |        |    +--------------------------+                                          |                               
  |        |                                                                          |                               
  |        |else                                                                      |                               
  |        |                                                                          |                               
  |        |    +--------------------------+                                          |                               
  |        +--> | CFMSensor::updateReading | update sensor reading (even it's nan)    |                               
  |        |    +--------------------------+                                          |                               
  |        +--------------------------------------------------------------------------+                               
  |                                                                                                                   
  |--> ->async_method_call                                                                                            
  |       +----------------------------------------------+                                                            
  |       |->getMaxRpm                                   |                                                            
  |       |                                              |                                                            
  |       |->register_property "Limit"                   |                                                            
  |       |                                              |                                                            
  |       | +-----------+                                |                                                            
  |       | | setMaxPWM | get class to set out limit max |                                                            
  |       | +-----------+                                |                                                            
  |       +----------------------------------------------+                                                            
  |       get "Limit"                                                                                                 
  |                                                                                                                   
  +--> set match rule and register callback                                                                           
          +---------------------------------------------+                                                             
          |get reading  from arg value                  |                                                             
          |                                             |                                                             
          |->getMaxRpm                                  |                                                             
          |                                             |                                                             
          |->set_property"Limit"                        |                                                             
          |                                             |                                                             
          |+-----------+                                |                                                             
          || setMaxPWM | get class to set out limit max |                                                             
          |+-----------+                                |                                                             
          +---------------------------------------------+                                                             
```

```
+------------------+                                                                   
| setupSensorMatch | : prepare callback, register to 'xyz.openbmc_project.Sensor.Value'
+-|----------------+                                                                   
  |                                                                                    
  |--> prepare event handler                                                           
  |       +--------------------+                                                       
  |       |read value from msg |                                                       
  |       |                    |                                                       
  |       |call arg callback   |                                                       
  |       +--------------------+                                                       
  |                                                                                    
  +--> reigster the handler to 'xyz.openbmc_project.Sensor.Value'                      
```

```
+--------------------------+                                                     
| CFMSensor::addTachRanges | : add tach range and update sensor reading          
+-|------------------------+                                                     
  |                                                                              
  +--> ->async_method_call                                                       
          +---------------------------------------------------------------------+
          |+-------------------------------------------------------------------+|
          ||prepare pair (min, max) and add to range                           ||
          ||                                                                   ||
          ||+--------------------------+                                       ||
          ||| CFMSensor::updateReading | update sensor reading (even it's nan) ||
          ||+--------------------------+                                       ||
          |+-------------------------------------------------------------------+|
          |service: serviceName                                                 |
          |object: path                                                         |
          |interface: org.freedesktop.DBus.Properties                           |
          |method: GetAll                                                       |
          +---------------------------------------------------------------------+
```

```
+--------------------------+                                                                        
| CFMSensor::updateReading | : update sensor reading (even it's nan)                                
+-|------------------------+                                                                        
  |    +----------------------+                                                                     
  |--> | CFMSensor::calculate | calculate total cfm (cubic feet per minute)                         
  |    +----------------------+                                                                     
  |                                                                                                 
  |--> if value changes && parent exists                                                            
  |                                                                                                 
  |------> parent->updateReading                                                                    
  |                                                                                                 
  |        +---------------------+                                                                  
  |------> | Sensor::updateValue | update property "Value", update instrumentation, check thresholds
  |        +---------------------+                                                                  
  |                                                                                                 
  |--> else                                                                                         
  |                                                                                                 
  |        +---------------------+                                                                  
  +------> | Sensor::updateValue | update property "Value", update instrumentation, check thresholds
           +---------------------+                                                                  
```

```
+----------------------+                                              
| CFMSensor::calculate | : calculate total cfm (cubic feet per minute)
+-|--------------------+                                              
  |                                                                   
  |--> for each tach                                                  
  |                                                                   
  |------> calculate the cfm (cubic feet per minute)                  
  |                                                                   
  |------> accumulate to total                                        
  |                                                                   
  +--> value = total / 100 (percent)                                  
```

```
+---------------------+                                                                                              
| Sensor::updateValue | : update property "Value", update instrumentation, check thresholds                          
+-|-------------------+                                                                                              
  |    +-----------------------------+                                                                               
  |--> | Sensor::updateValueProperty | update property "Value" if needed                                             
  |    +-----------------------------+                                                                               
  |    +-------------------------------+                                                                             
  |--> | Sensor::updateInstrumentation | update instrumentation attributes                                           
  |    +-------------------------------+                                                                             
  |    +------------------------------------+                                                                        
  |--> | ExitAirTempSensor::checkThresholds |                                                                        
  |    +-|----------------------------------+                                                                        
  |      |    +-----------------------------+                                                                        
  |      +--> | thresholds::checkThresholds | compare arg value to each threshold of sensor, return whatever we found
  |           +-----------------------------+                                                                        
  |                                                                                                                  
  |--> if the valud is valid                                                                                         
  |                                                                                                                  
  +------> label 'functional' and 'available'                                                                        
```

```
+-----------------------------+                                    
| Sensor::updateValueProperty | : update property "Value" if needed
+-|---------------------------+                                    
  |    +------------------------+                                  
  +--> | Sensor::updateProperty | update property if needed        
       +------------------------+                                  
```

```
+------------------------+                                            
| Sensor::updateProperty | : update property if needed                
+-|----------------------+                                            
  |    +------------------------+                                     
  |--> | Sensor::requiresUpdate | determine if we need to update value
  |    +------------------------+                                     
  |                                                                   
  |--> if we need                                                     
  |                                                                   
  +------> ->set_property                                             
```

```
+------------------------+                                       
| Sensor::requiresUpdate | : determine if we need to update value
+-|----------------------+                                       
  |                                                              
  |--> return true if we have a nan                              
  |                                                              
  |--> return true if difference > threshold                     
  |                                                              
  +--> return false                                              
```

```
+-----------------------------+                                                                          
| thresholds::checkThresholds | : compare arg value to each threshold of sensor, return whatever we found
+-|---------------------------+                                                                          
  |                                                                                                      
  | for each threshold in sensor                                                                         
  |                                                                                                      
  |------> if the direction is 'high'                                                                    
  |                                                                                                      
  |----------> if arg value >= threshold                                                                 
  |                                                                                                      
  |--------------> append value to local list                                                            
  |                                                                                                      
  |----------> elif value < (threshold - hysteresis)                                                     
  |                                                                                                      
  +--------------> append value to local list                                                            
  |                                                                                                      
  |------> elif the direction is 'low'                                                                   
  |                                                                                                      
  |----------> if arg value <= threshold                                                                 
  |                                                                                                      
  |--------------> append value to local list                                                            
  |                                                                                                      
  |----------> elif value > (threshold - hysteresis)                                                     
  |                                                                                                      
  +--------------> append value to local list                                                            
```

```
+-----------+                                 
| setMaxPWM | : get class to set out limit max
+-|---------+                                 
  |                                           
  +--> ->async_method_call                    
          +---------------------------------+ 
          |for each (path, obj_dict) in ret | 
          |                                 | 
          |    ->async_method_call          | 
          |       +------------------+      | 
          |       |set "OutLimitMax" |      | 
          |       +------------------+      | 
          +---------------------------------+ 
          get "Class"                         
```

### fansensor

```
FanMain.cpp                                                    
+------+                                                        
| main |                                                        
+-|----+                                                        
  |                                                             
  |--> ->request_name("xyz.openbmc_project.FanSensor")          
  |                                                             
  |--> io.post                                                  
  |       +-------------------------------------+               
  |       |+---------------+                    |               
  |       || createSensors | create fan sensors |               
  |       |+---------------+                    |               
  |       +-------------------------------------+               
  |                                                             
  |--> prepare event handler                                    
  |       +-------------------------------------+               
  |       |+---------------+                    |               
  |       || createSensors | create fan sensors |               
  |       |+---------------+                    |               
  |       +-------------------------------------+               
  |                                                             
  |--> for each sensor type                                     
  |                                                             
  |------> prepare match rule and register event handler        
  |                                                             
  |--> prepare event handler                                    
  |       +----------------------------------------------------+
  |       |+------------------------+                          |
  |       || createRedundancySensor | create redundancy sensor |
  |       |+------------------------+                          |
  |       +----------------------------------------------------+
  |                                                             
  |--> prepare match rule and register event handler            
  |                                                             
  |    +-----------------------------+                          
  +--> | setupManufacturingModeMatch |                          
       +-----------------------------+                          
```

```
+---------------+                                                                                                  
| createSensors | : create fan sensors                                                                             
+-|-------------+                                                                                                  
  |                                                                                                                
  |--> prepare callback for GetSensorConfiguration                                                                 
  |       +------------------------------------------------------------------------------+                         
  |       |+-----------+                                                                 |                         
  |       || findFiles | find files under "/sys/class/hwmon" and add to arg paths        |                         
  |       |+-----------+                                                                 |                         
  |       |                                                                              |                         
  |       |for each found path                                                           |                         
  |       |                                                                              |                         
  |       |    +------------+                                                            |                         
  |       |    | getFanType | get fan type (aspeed, nuvoton or i2c) based on device name |                         
  |       |    +------------+                                                            |                         
  |       |                                                                              |                         
  |       |    for each sensor in configurations                                         |                         
  |       |                                                                              |                         
  |       |        if type is aspeed or nuvoton                                          |                         
  |       |                                                                              |                         
  |       |            save sensor data and break (bc there's only one sucn sensor)      |                         
  |       |                                                                              |                         
  |       |        if type is i2c                                                        |                         
  |       |                                                                              |                         
  |       |            if (bus, addr) of configuration and device match                  |                         
  |       |                                                                              |                         
  |       |                save sensor data and break                                    |                         
  |       |                                                                              |                         
  |       |    +---------------------------+                                             |                         
  |       |    | parseThresholdsFromConfig |                                             |                         
  |       |    +---------------------------+                                             |                         
  |       |                                                                              |                         
  |       |    handle presence sensor if there's any                                     |                         
  |       |                                                                              |                         
  |       |    set power state if "PowerState" is found                                  |                         
  |       |                                                                              |                         
  |       |    prepare tach sensor                                                       |                         
  |       |                                                                              |                         
  |       |    prepare pwm sensor                                                        |                         
  |       |                                                                              |                         
  |       |+------------------------+                                                    |                         
  |       || createRedundancySensor | create redundancy sensor                           |                         
  |       |+------------------------+                                                    |                         
  |       +------------------------------------------------------------------------------+                         
  |    +------------------------------------------+                                                                
  |--> | GetSensorConfiguration::getConfiguration | get configuration path if it with matches any of the interfaces
  |    +------------------------------------------+                                                                
  |    +-------------------------------------------------+                                                         
  +--> | GetSensorConfiguration::~GetSensorConfiguration | call callback                                           
       +-------------------------------------------------+                                                         
```

```
+------------------------+                                     
| createRedundancySensor | : create redundancy sensor          
+-|----------------------+                                     
  |                                                            
  +--> ->async_method_call                                     
          +---------------------------------------------------+
          |for each path_pair in manage_obj                   |
          |                                                   |
          |    for each interface in path_pair                |
          |                                                   |
          |        if interface matches the redundancy config |
          |                                                   |
          |            save it in systemRedundancy            |
          |                                                   |
          |            return (bc currently only support one) |
          +---------------------------------------------------+
          "xyz.openbmc_project.EntityManager"                  
          "/"                                                  
          "org.freedesktop.DBus.ObjectManager"                 
          "GetManagedObjects"                                  
```

### nvmesensor

```
+---------------------------------------------------+                                            
|                      Sensor                       |                                            
|                       ----                        |                                            
| checkThresholds()         errCount                |                                            
| name                      instrumentation         |                                            
| configurationPath         externalSetHook()       |                                            
| objectType                Level                   |                                            
| isSensorSettable          Direction               |                                            
| isValueMutable            thresholdInterfaces     |                       +-------------------+
| maxValue                  getThresholdInterface() |                       |    NVMeSensor     |
| minValue                  updateInstrumentation() |                       |     --------      |
| thresholds                setSensorValue()        |                       | sensorType        |
| sensorInterface           setInitialProperties()  |         inherit       | sample()          |
| association               propertyLevel()         |   <----------------   | bus               |
| availableInterface        propertyAlarm()         |                       | scanDelayTicks    |
| operationalInterface      readingStateGood()      |                       | objServer         |
| valueMutabilityInterface  markFunctional()        |                       | scanDelay         |
| value                     markAvailable()         |                       | checkThresholds() |
| rawValue                  incrementError()        |                       +-------------------+
| overriddenState           inError()               |                                            
| internalSet               updateValue()           |                                            
| hysteresisTrigger         updateProperty()        |                                            
| hysteresisPublish         requiresUpdate()        |                                            
| dbusConnection            fillMissingThresholds   |                                            
| readState                 updateValueProperty     |                                            
+---------------------------------------------------+                                            
```

```
+-----------------------------+                                                    
|        NVMeContext          |                      +----------------------------+
|         ---------           |                      |     NVMeBasicContext       |
|  addSensor()                |                      |      --------------        |
|  getSensorAtPath()          |                      | pollNVMeDevices()          |
|  removeSensor()             |                      | readAndProcessNVMeSensor() |
|  close()                    |         inherit      | processResponse()          |
|  pollNVMeDevices()          |   <----------------  | NVMeBasicContext()         |
|  readAndProcessNVMeSensor() |                      | io                         |
|  processResponse()          |                      | thread                     |
|  scanTimer                  |                      | reqStream                  |
|  rootBus                    |                      | respStream                 |
|  sensors                    |                      +----------------------------+
|  pollCursor                 |                                                    
+-----------------------------+                                                    
```

```
+------+                                                                                            
| main | : create sensors from configuration, prepare handlers for ???, intf removal, and mode match
+-|----+ NVMeSensorMain.cpp                                                                         
  |                                                                                                 
  |--> systemBus->request_name("xyz.openbmc_project.NVMeSensor")   <= to specify service?           
  |                                                                                                 
  |    +---------+                                                                                  
  |--> | io.post | send handler to io context for execution                                         
  |    +---------------+                                                                            
  |    | createSensors | get configuration, add sensors for each conf, read value and update sensors
  |    +---------------+                                                                            
  |                                                                                                 
  |--> set up event handler                                                                         
  |    +---------------+                                                                            
  |    | createSensors | get configuration, add sensors for each conf, read value and update sensors
  |    +---------------+                                                                            
  |    +-------------------------------+                                                            
  |--> | setupPropertiesChangedMatches | for each types: prepare match and add to matches           
  |    +-------------------------------+                                                            
  |                                                                                                 
  |--> register handler for interface removal                                                       
  |    +------------------+                                                                         
  |    | interfaceRemoved | for each context, remove interface-matched sensor value                 
  |    +------------------+                                                                         
  |    +-----------------------------+                                                              
  |--> | setupManufacturingModeMatch | prepare handlers for manufacturing mode match                
  |    +-----------------------------+                                                              
  |    +--------+                                                                                   
  +--> | io.run |                                                                                   
       +--------+                                                                                   
```

```
+---------------+                                                                                         
| createSensors | : get configuration, add sensors for each conf, read value and update sensors           
+---|-----------+                                                                                         
    |                     +----------------------------+                                                  
    |--> prepare a getter | handleSensorConfigurations | for each conf: add sensor to context             
    |                     +----------------------------+ for each context: read value and update to sensor
    |    +--------------------------+                                                                     
    |--> | getter->getConfiguration | get configuration path if it with matches any of the interfaces     
    |    +--------------------------+                                                                     
    |                                                                                                     
    +--> (its destructor calls callback: handleSensorConfigurations)                                      
```

```
+-----------------------------+                                                                                          
| handleSensorConfigurations  | : for each conf: add sensor to context, for each context: read value and update to sensor
+-------|---------------------+                                                                                          
        |                                                                                                                
        |--> clear nvmeDeviceMap (global var)                                                                            
        |                                                                                                                
        |--> for each found configuration                                                                                
        |                                                                                                                
        |------> find target sensor base in the configuration                                                            
        |                                                                                                                
        |------> continue if not found                                                                                   
        |                                                                                                                
        |------> get bus number, sensor name, and root bus                                                               
        |                                                                                                                
        |------> continue if any of the three isn't found                                                                
        |                                                                                                                
        |        +---------------------------+                                                                           
        |------> | parseThresholdsFromConfig | given sensor data, find pairs of (hysteresis, direction, severity, value) 
        |        +---------------------------+ and push bask to arg vector                                               
        |        +-----------------------+                                                                               
        |------> | provideRootBusContext | ensure map[rootBus] = context is there, return context                        
        |        +-----------------------+                                                                               
        |                                                                                                                
        |------> prepare an nvme sensor                                                                                  
        |                                                                                                                
        |------> add the sensor to context                                                                               
        |                                                                                                                
        |--> for each context in nvme dev map                                                                            
        |                                                                                                                
        |        +--------------------------+                                                                            
        +------> | context->pollNVMeDevices | for each sensor, send command to read value and update to sensor           
                 +--------------------------+                                                                            
```

```
+-----------------------------------+                                                                            
| NVMeBasicContext::pollNVMeDevices | : for each sensor, send command to read value and update to sensor         
+--------|--------------------------+                                                                            
         |    +--------------------------------+                                                                 
         +--> | self->readAndProcessNVMeSensor | for each sensor, send command to read value and update to sensor
              +--------------------------------+                                                                 
```

```
+--------------------------------------------+                                                                   
| NVMeBasicContext::readAndProcessNVMeSensor | : for each sensor, send command to read value and update to sensor
+----------|---------------------------------+                                                                   
           |                                                                                                     
           |--> get the sensor from current cursor                                                               
           |                                                                                                     
           +--> cursor++                                                                                         
           |                                                                                                     
           |--> return if the reading state isn't good                                                           
           |                                                                                                     
           |--> return if it's not the time to sample                                                            
           |                                                                                                     
           |    +------------------+                                                                             
           |--> | encodeBasicQuery | encode 'bus', 'addr', and 'offset' into  one command                        
           |    +------------------+                                                                             
           |    +--------------------------+                                                                     
           |--> | boost::asio::async_write | send command (where did it send to?)                                
           |    +--------------------------+                                                                     
           |    +-------------------------+                                                                      
           |--> | boost::asio::async_read | read response                                                        
           |    +-------------------------+                                                                      
           |    +-----------------------+                                                                        
           |--> | self->processResponse | get value from msg, update to sensor if it's valid                     
           |    +-----------------------+                                                                        
           |    +--------------------------------+                                                               
           +--> | self->readAndProcessNVMeSensor | recursive, but note that the cursor is updated                
                +--------------------------------+                                                               
```

```
+-----------------------------------+                                                     
| NVMeBasicContext::processResponse | : get value from msg, update to sensor if it's valid
+--------|--------------------------+                                                     
         |                                                                                
         |--> if return status shows it has an issue                                      
         |                                                                                
         |------> sensor->markFunctional(false)                                           
         |                                                                                
         |------> return                                                                  
         |                                                                                
         |    +-----------------------+                                                   
         |--> | getTemperatureReading | check if reading is valid                         
         |    +-----------------------+                                                   
         |                                                                                
         +--> sensor->updateValue(value)                                                  
```

```
+------------------+                                                            
| interfaceRemoved | : for each context, remove interface-matched sensor value  
+----|-------------+                                                            
     |    +--------------+                                                      
     |--> | message.read | read path and interfaces from msg                    
     |    +--------------+                                                      
     |                                                                          
     |--> for each context in arg 'contexts'                                    
     |                                                                          
     |        +--------------------------+                                      
     |------> | context->getSensorAtPath | get sensor from context based on path
     |        +--------------------------+                                      
     |                                                                          
     |------> find sensor obj type from interfaces                              
     |                                                                          
     |        +-----------------------+                                         
     +------> | context->removeSensor | remove sensor value                     
              +-----------------------+                                         
```
