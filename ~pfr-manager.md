```
service/src/mainapp.cpp                                                                                      
+------+                                                                                                      
| main |                                                                                                      
+-|----+                                                                                                      
  |    +-----------+                                                                                          
  |--> | pfr::init | get pfr bus/addr info                                                                    
  |    +-----------+                                                                                          
  |    +----------------------------+                                                                         
  |--> | pfr::checkPFRandAddObjects | monitor signals, update dbus propeorties (versions and provision_status)
  |    +----------------------------+                                                                         
  |    +------------------------+                                                                             
  |--> | pfr::updateCPLDversion | read cpld version and set to dbus property                                  
  |    +------------------------+                                                                             
  |                                                                                                           
  |--> .add_manager("/xyz/openbmc_project/pfr")                                                               
  |                                                                                                           
  |--> for each ver_component (bmc_recovery, bios_recovery, cpld_recovery, afm_active, afm_recovery)          
  |    -                                                                                                      
  |    +--> add to pfr::pfrVersionObjects                                                                     
  |                                                                                                           
  |--> if pfr_config_obj exists                                                                               
  |    |                                                                                                      
  |    |    +-------------------------------------+                                                           
  |    |--> | PfrConfig::updateProvisioningStatus | read provision status and set to iface properties         
  |    |    +-------------------------------------+                                                           
  |    |                                                                                                      
  |    +--> if it's provisioned                                                                               
  |         -                                                                                                 
  |         +--> prepare pfr_postcode_obj                                                                     
  |                                                                                                           
  +--> ->request_name("xyz.openbmc_project.PFR.Manager")                                                      
```

```
service/src/mainapp.cpp                                                                                                  
+----------------------------+                                                                                            
| pfr::checkPFRandAddObjects | : monitor signals, update dbus propeorties (versions and provision_status)                 
+-|--------------------------+                                                                                            
  |    +------------------------+                                                                                         
  |--> | pfr::checkPfrInterface | check if pfr is supported, exit servcie if not                                          
  |    +------------------------+                                                                                         
  |                                                                                                                       
  +--> timer->async_wait                                                                                                  
       |                                                                                                                  
       |--> if retry_cnt                                                                                                  
       |    |                                                                                                             
       |    |    +----------------------------+                                                                           
       |    +--> | pfr::checkPFRandAddObjects | (recursive)                                                               
       |         +----------------------------+                                                                           
       |                                                                                                                  
       |--> else                                                                                                          
       |    |                                                                                                             
       |    |    +---------------------+                                                                                  
       |    |--> | pfr::monitorSignals | minotir signal (set checkpoint, log event, or update dbus properties accordingly)
       |    |    +---------------------+                                                                                  
       |    |    +--------------------------------+                                                                       
       |    |--> | pfr::updateDbusPropertiesCache | update dbus properties (version of each img_type, provision status)   
       |    |    +--------------------------------+                                                                       
       |    |    +------------------------+                                                                               
       |    +--> | pfr::updateCPLDversion | read cpld version and set to dbus property                                    
       |         +------------------------+                                                                               
       |                                                                                                                  
       +--> retry_cnt--                                                                                                   
```

```
service/src/mainapp.cpp                                                   
+------------------------+                                                 
| pfr::checkPfrInterface | : check if pfr is supported, exit servcie if not
+-|----------------------+                                                 
  |                                                                        
  |--> if i2c_config isn't loaded (no bus/addr info)                       
  |    |                                                                   
  |    |    +-----------+                                                  
  |    +--> | pfr::init | get pfr bus/addr info                            
  |         +-----------+                                                  
  +--> else                                                                
       |                                                                   
       |    +----------------------------+                                 
       |--> | pfr::getProvisioningStatus | get provision status            
       |    +----------------------------+                                 
       |                                                                   
       |--> return if it's supported and provisioned                       
       |                                                                   
       +--> otherwise, exit the service                                    
```

```
service/src/mainapp.cpp                                           
+-----------+                                                      
| pfr::init | : get pfr bus/addr info                              
+-|---------+                                                      
  |                                                                
  +--> .async_method_call                                          
          +-------------------------------------------------------+
          |if obj_path ends with "Baseboard/PFR"                  |
          |                                                       |
          |    .async_method_call                                 |
          |       +----------------------------------------------+|
          |       |for each prop in property_list                ||
          |       |-                                             ||
          |       |+--> if name is 'Bus' or 'Address', save info ||
          |       +----------------------------------------------+|
          |       service_name                                    |
          |       obj_path                                        |
          |       iface: "org.freedesktop.DBus.Properties"        |
          |       method: "GetAll"                                |
          |       arg: "xyz.openbmc_project.Configuration.PFR"    |
          +-------------------------------------------------------+
          service: "xyz.openbmc_project.ObjectMapper"              
          object: "/xyz/openbmc_project/object_mapper"             
          iface: "xyz.openbmc_project.ObjectMapper"                
          method: "GetSubTree"                                     
          arg: "xyz.openbmc_project.Configuration.PFR"             
```

```
libpfr/src/pfr.cpp                                                                
+----------------------------+                                                     
| pfr::getProvisioningStatus | : get provision status                              
+-|--------------------------+                                                     
  |                                                                                
  |--> prepare cpld_dev (i2c_dev)                                                  
  |    +------------------+                                                        
  |    | I2CFile::I2CFile | open target i2c dev and set slave addr                 
  |    +------------------+                                                        
  |    +--------------------------+                                                
  |--> | I2CFile::i2cReadByteData | given offset (provision_status), read byte data
  |    +--------------------------+                                                
  |    +--------------------------+                                                
  |--> | I2CFile::i2cReadByteData | given offset (rot_id), read byte data          
  |    +--------------------------+                                                
  |                                                                                
  +--> read ufm status (locked? provisioned? support?)                             
                                                                                   
```

```
service/src/mainapp.cpp                                                                                           
+---------------------+                                                                                            
| pfr::monitorSignals | : minotir signal (set checkpoint, log event, or update dbus properties accordingly)        
+-|-------------------+                                                                                            
  |                                                                                                                
  |--> preapre match rule (StartupFinished) and callback                                                           
  |       +----------------------------------------------------------------------+                                 
  |       |if finished_setting_checkpoint isn't set yet                          |                                 
  |       ||                                                                     |                                 
  |       ||--> finished_setting_checkpoint = true                               |                                 
  |       ||                                                                     |                                 
  |       ||    +---------------------------+                                    |                                 
  |       |+--> | pfr::setBMCBootCheckpoint | write arg checkpoint to target reg |                                 
  |       |     +---------------------------+                                    |                                 
  |       +----------------------------------------------------------------------+                                 
  |    +----------------------------+                                                                              
  |--> | pfr::checkAndSetCheckpoint | ensure bmc_boot_finished_checkpoint is written to cpld reg                   
  |    +----------------------------+                                                                              
  |                                                                                                                
  |--> prepare match rule (chassis state change)                                                                   
  |       +-------------------------------------------------------------------------------------------------------+
  |       |if it's on                                                                                             |
  |       ||                                                                                                      |
  |       ||    +---------------------------------+                                                               |
  |       |+--> | pfr::monitorPlatformStateChange | log event                                                     |
  |       |     +---------------------------------+                                                               |
  |       |                                                                                                       |
  |       |elif it's off                                                                                          |
  |       ||                                                                                                      |
  |       ||--> cancel timer                                                                                      |
  |       ||                                                                                                      |
  |       ||    +------------------------+                                                                        |
  |       |+--> | pfr::checkAndLogEvents | log event                                                              |
  |       |     +------------------------+                                                                        |
  |       |+--------------------------------+                                                                     |
  |       || pfr::updateDbusPropertiesCache | update dbus properties (version of each img_type, provision status) |
  |       |+--------------------------------+                                                                     |
  |       +-------------------------------------------------------------------------------------------------------+
  |                                                                                                                
  |--> prepare match rule (host state change)                                                                      
  |       +-----------------------------------------+                                                              
  |       |logic is similar to chassis state change |                                                              
  |       +-----------------------------------------+                                                              
  |                                                                                                                
  |--> prepare match rule (os state change)                                                                        
  |       +-----------------------------------------+                                                              
  |       |logic is similar to chassis state change |                                                              
  |       +-----------------------------------------+                                                              
  |    +------------------------+                                                                                  
  +--> | pfr::checkAndLogEvents | log event                                                                        
       +------------------------+                                                                                  
```

```
service/src/mainapp.cpp                                                                   
+----------------------------+                                                             
| pfr::checkAndSetCheckpoint | : ensure bmc_boot_finished_checkpoint is written to cpld reg
+-|--------------------------+                                                             
  |                                                                                        
  +--> ->async_method_call                                                                 
                                                                                           
           if finished_setting_checkpoint isn't set                                        
           |                                                                               
           |--> finished_setting_checkpoint = true                                         
           |                                                                               
           |    +---------------------------+                                              
           +--> | pfr::setBMCBootCheckpoint | write arg checkpoint to target reg           
                +---------------------------+                                              
```

```
libpfr/src/pfr.cpp                                               
+---------------------------+                                     
| pfr::setBMCBootCheckpoint | : write arg checkpoint to target reg
+-|-------------------------+                                     
  |                                                               
  |--> prepare cpld_dev to read rot_version                       
  |                                                               
  |--> determine reg for write                                    
  |                                                               
  +--> i2c write byte (checkpoint) to the reg                     
```

```
service/src/mainapp.cpp                                    
+---------------------------------+                         
| pfr::monitorPlatformStateChange | : log event             
+-|-------------------------------+                         
  |                                                         
  +--> timer->async_wait                                    
          +------------------------------------------------+
          |+------------------------+                      |
          || pfr::checkAndLogEvents | log event            |
          |+------------------------+                      |
          |+---------------------------------+             |
          || pfr::monitorPlatformStateChange | (recursive) |
          |+---------------------------------+             |
          +------------------------------------------------+
```

```
service/src/mainapp.cpp                                                                                      
+--------------------------------+                                                                            
| pfr::updateDbusPropertiesCache | : update dbus properties (version of each img_type, provision status)      
+-|------------------------------+                                                                            
  |                                                                                                           
  |--> for each pfr_ver_obj                                                                                   
  |    |                                                                                                      
  |    |    +---------------------------+                                                                     
  |    +--> | PfrVersion::updateVersion | given img_type, read version from cpld or spi, set to iface property
  |         +---------------------------+                                                                     
  |    +-------------------------------------+                                                                
  +--> | PfrConfig::updateProvisioningStatus | read provision status and set to iface properties              
       +-------------------------------------+                                                                
```

```
service/src/pfr_mgr.cpp                                                                            
+---------------------------+                                                                       
| PfrVersion::updateVersion | : given img_type, read version from cpld or spi, set to iface property
+-|-------------------------+                                                                       
  |                                                                                                 
  +--> if version_iface is initialized                                                              
       |                                                                                            
       |    +-------------------------+                                                             
       |--> | pfr::getFirmwareVersion | given img_type, read version from cpld or spi accordingly   
       |    +-------------------------+                                                             
       |    +-------------------+                                                                   
       |--> | pfr::printVersion | print version                                                     
       |    +-------------------+                                                                   
       |                                                                                            
       +--> ->set_property (version)                                                                
```

```
libpfr/src/pfr.cpp                                                                                  
+-------------------------+                                                                          
| pfr::getFirmwareVersion | : given img_type, read version from cpld or spi accordingly              
+-|-----------------------+                                                                          
  |                                                                                                  
  |--> switch img_type                                                                               
  |    case cpld_active                                                                              
  |    -    +----------------------+                                                                 
  |    +--> | pfr::readCPLDVersion | prepare cpld dev to read version info, assemble and return it   
  |         +----------------------+                                                                 
  |--> case cpld_recovery                                                                            
  |    -    +--------------------------+                                                             
  |    +--> | pfr::readVersionFromCPLD | given reg offset, prepare cpld dev to read svn.rot_ver      
  |         +--------------------------+                                                             
  |--> case bios_active                                                                              
  |    -    +--------------------------+                                                             
  |    +--> | pfr::readVersionFromCPLD | given reg offset, prepare cpld dev to read svn.rot_ver      
  |         +--------------------------+                                                             
  |--> case bios_recovery                                                                            
  |    -    +--------------------------+                                                             
  |    +--> | pfr::readVersionFromCPLD | given reg offset, prepare cpld dev to read svn.rot_ver      
  |         +--------------------------+                                                             
  |--> case bmc_active                                                                               
  |--> case bmc_recovery                                                                             
  |    -    +----------------------------+                                                           
  |    +--> | pfr::readBMCVersionFromSPI | give img_type, determine mtd and read version info from it
  |         +----------------------------+                                                           
  |--> case afm_active                                                                               
  |    -    +--------------------------+                                                             
  |    +--> | pfr::readVersionFromCPLD | given reg offset, prepare cpld dev to read svn.rot_ver      
  |         +--------------------------+                                                             
  +--> case afm_recovery                                                                             
       -    +--------------------------+                                                             
       +--> | pfr::readVersionFromCPLD | given reg offset, prepare cpld dev to read svn.rot_ver      
            +--------------------------+                                                             
```

```
libpfr/src/pfr.cpp                                                                            
+----------------------+                                                                       
| pfr::readCPLDVersion | : prepare cpld dev to read version info, assemble and return it       
+-|--------------------+                                                                       
  |                                                                                            
  |--> for each main_cpld_gpio_line                                                            
  |    |                                                                                       
  |    |    +-------------------+                                                              
  |    |--> | pfr::getGPIOInput | get input value                                              
  |    |    +-------------------+                                                              
  |    |                                                                                       
  |    +--> assemble main_cpld_ver                                                             
  |                                                                                            
  |--> for each pid_gpio_line                                                                  
  |    |                                                                                       
  |    |    +-------------------+                                                              
  |    |--> | pfr::getGPIOInput | get input value                                              
  |    |    +-------------------+                                                              
  |    |                                                                                       
  |    +--> assemble pid_ver                                                                   
  |                                                                                            
  |--> prepare cpld_dev to read pfr_rot_id, assign to cpld_rot_value                           
  |                                                                                            
  |--> if cpld_rot_value == pfr_rot_value (should)                                             
  |    |                                                                                       
  |    |    +--------------------------+                                                       
  |    |--> | pfr::readVersionFromCPLD | given reg offset, prepare cpld dev to read svn.rot_ver
  |    |    +--------------------------+                                                       
  |    |    +-------------------+                                                              
  |    +--> | pfr::readCPLDHash | prepare cpld dev to read hash                                
  |         +-------------------+                                                              
  |                                                                                            
  +--> assemble version and return it                                                          
```

```
libpfr/src/pfr.cpp                                                                        
+----------------------------+                                                             
| pfr::readBMCVersionFromSPI | : give img_type, determine mtd and read version info from it
+-|--------------------------+                                                             
  |                                                                                        
  |--> if img_type == bmc_active                                                           
  |    -                                                                                   
  |    +--> mtd = "/dev/mtd/pfm"                                                           
  |                                                                                        
  |--> elif img_type == bmc_active                                                         
  |    -                                                                                   
  |    +--> mtd = "/dev/mtd/rc-image"                                                      
  |                                                                                        
  |--> prepare spi_dev (open the dev file)                                                 
  |                                                                                        
  |--> rad versio, build_num, build_hash                                                   
  |                                                                                        
  +--> adjust version format and return it                                                 
```

```
service/src/pfr_mgr.cpp                                                                   
+-------------------------------------+                                                    
| PfrConfig::updateProvisioningStatus | : read provision status and set to iface properties
+-|-----------------------------------+                                                    
  |    +----------------------------+                                                      
  |--> | pfr::getProvisioningStatus | get provision status                                 
  |    +----------------------------+                                                      
  |                                                                                        
  |--> ->set_property (provisioned)                                                        
  |                                                                                        
  |--> ->set_property (locked)                                                             
  |                                                                                        
  +--> ->set_property (support)                                                            
```

```
service/src/mainapp.cpp                                                                     
+------------------------+                                                                   
| pfr::updateCPLDversion | : read cpld version and set to dbus property                      
+-|----------------------+                                                                   
  |    +----------------------+                                                              
  |--> | pfr::readCPLDVersion | prepare cpld dev to read version info, assemble and return it
  |    +----------------------+                                                              
  |                                                                                          
  +--> ->async_method_call                                                                   
           service: "xyz.openbmc_project.Settings"                                           
           objecty: "/xyz/openbmc_project/software/cpld_active"                              
           iface: "org.freedesktop.DBus.Properties"                                          
           method: "Set"                                                                     
           arg: "xyz.openbmc_project.Software.Version", "Version"                            
```
