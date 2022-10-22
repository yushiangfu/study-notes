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