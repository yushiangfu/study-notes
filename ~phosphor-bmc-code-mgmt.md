```
item_updater_main.cpp                                        
+------+                                                      
| main |                                                      
+-|----+                                                      
  |                                                           
  |--> SOFTWARE_OBJPATH=/xyz/openbmc_project/software         
  |                                                           
  |--> prepare updater                                        
  |    +--------------------------+                           
  |    | ItemUpdater::ItemUpdater | prepare an item updater   
  |    +--------------------------+                           
  |                                                           
  +--> request name 'xyz.openbmc_project.Software.BMC.Updater'
```

```
+--------------------------+                                                                               
| ItemUpdater::ItemUpdater | : prepare an item updater                                                     
+-|------------------------+                                                                               
  |    +----------------+                                                                                  
  |--> | getRunningSlot | get running slot from "/run/media/slot"                                          
  |    +----------------+                                                                                  
  |    +----------------------------------+                                                                
  |--> | ItemUpdater::setBMCInventoryPath | get bmc_inventory_path from obj_mapper, save in member variable
  |    +----------------------------------+                                                                
  |    +------------------------------+                                                                    
  |--> | ItemUpdater::processBMCImage | create a few associations and a activation instance                
  |    +------------------------------+                                                                    
  |    +-------------------------------------+                                                             
  +--> | ItemUpdater::restoreFieldModeStatus | read uboot env, enable field mode if necessary              
       +-------------------------------------+                                                             
```

```
+----------------------------------+                                                                  
| ItemUpdater::setBMCInventoryPath | : get bmc_inventory_path from obj_mapper, save in member variable
+-|--------------------------------+                                                                  
  |                                                                                                   
  |--> .new_method_call xyz.openbmc_project.ObjectMapper                                              
  |                     /xyz/openbmc_project/object_mapper                                            
  |                     xyz.openbmc_project.ObjectMapper                                              
  |                     GetSubTreePaths                                                               
  |                                                                                                   
  |--> prepare arguments, e.g., filter = xyz.openbmc_project.Inventory.Item.Bmc                       
  |                                                                                                   
  |--> call and read response                                                                         
  |                                                                                                   
  +--> get bmc_inventory_path                                                                         
```

```
+------------------------------+                                                                                         
| ItemUpdater::processBMCImage | : create a few associations and a activation instance                                   
+-|----------------------------+                                                                                         
  |                                                                                                                      
  |--> create folder 'media-dir'                                                                                         
  |                                                                                                                      
  |--> for each entry in 'media-dir'                                                                                     
  |    -                                                                                                                 
  |    +--> if the file starts with '???/rofs-'                                                                          
  |         |                                                                                                            
  |         |    +-----------------------------+                                                                         
  |         |--> | VersionClass::getBMCVersion | get bmc version from file                                               
  |         |    +-----------------------------+                                                                         
  |         |                                                                                                            
  |         |--> id = version + flash_id                                                                                 
  |         |                                                                                                            
  |         |--> continue if id is already in 'versions'                                                                 
  |         |                                                                                                            
  |         |    +-------------------------------------+                                                                 
  |         |--> | VersionClass::getBMCExtendedVersion | get bmc ext version from file                                   
  |         |    +-------------------------------------+                                                                 
  |         |                                                                                                            
  |         |--> if 'functionial'                                                                                        
  |         |    |                                                                                                       
  |         |    |    +------------------------------------------+                                                       
  |         |    +--> | ItemUpdater::createFunctionalAssociation | create association ('functional' & 'software_version')
  |         |         +------------------------------------------+                                                       
  |         |                                                                                                            
  |         |--> if state is active                                                                                      
  |         |    |                                                                                                       
  |         |    |--> create association ('inventory' & 'activation')                                                    
  |         |    |                                                                                                       
  |         |    |    +-------------------------+                                                                        
  |         |    +--> | createActiveAssociation | create association ('active', 'software')                              
  |         |         +-------------------------+                                                                        
  |         |    +------------------------------------------+                                                            
  |         |--> | ItemUpdater::createUpdateableAssociation | create association ('updateable', 'software_version')      
  |         |    +------------------------------------------+                                                            
  |         |                                                                                                            
  |         |--> create activation instance for this version                                                             
  |         |                                                                                                            
  |         +--> if state is active                                                                                      
  |              -                                                                                                       
  |              +--> create redundancy_priority instance for this version                                               
  |                                                                                                                      
  +--> if no functioinal found                                                                                           
       |                                                                                                                 
       |--> create rofs-<versionId>-functional                                                                           
       |                                                                                                                 
       |    +------------------------------+                                                                             
       +--> | ItemUpdater::processBMCImage | (call again to create dbus instance)                                        
            +------------------------------+                                                                             
```

```
+-------------------------------------+                                                 
| ItemUpdater::restoreFieldModeStatus | : read uboot env, enable field mode if necessary
+-|-----------------------------------+                                                 
  |                                                                                     
  |--> read data from /dev/mtd/u-boot-env                                               
  |                                                                                     
  +--> if 'fieldmode=true' exists                                                       
       |                                                                                
       |    +-------------------------------+                                           
       +--> | ItemUpdater::fieldModeEnabled | enable field mode (?)                     
            +-------------------------------+                                           
```

```
+-------------------------------+                        
| ItemUpdater::fieldModeEnabled | : enable field mode (?)
+-|-----------------------------+                        
  |                                                      
  |--> if arg == ture && it's not yet enabled            
  |                                                      
  |--> call service: org.freedesktop.systemd1            
  |         object: /org/freedesktop/systemd1            
  |         iface: org.freedesktop.systemd1.Manager      
  |         StartUnit                                    
  |                                                      
  |--> call service: org.freedesktop.systemd1            
  |         object: /org/freedesktop/systemd1            
  |         iface: org.freedesktop.systemd1.Manager      
  |         StopUnit                                     
  |                                                      
  +--> call service: org.freedesktop.systemd1            
            object: /org/freedesktop/systemd1            
            iface: org.freedesktop.systemd1.Manager      
            MaskUnitFiles                                
```

```
download_manager_main.cpp                                 
+------+                                                   
| main | ???                                               
+-|----+                                                   
  |                                                        
  |--> request name 'xyz.openbmc_project.Software.Download'
  |                                                        
  +--> endless loop                                        
       -                                                   
       +--> process (?)                                    
```

```
+---------------------------+                                  
| Download::downloadViaTFTP | : download image from tftp server
+-|-------------------------+                                  
  |                                                            
  |--> check if folder for upload exists: img-upload-dir       
  |                                                            
  |    +------+                                                
  |--> | fork |                                                
  |    +------+                                                
  |                                                            
  +--> if self is forked child                                 
       |                                                       
       |    +------+                                           
       |--> | fork | (why fork again)                          
       |    +------+                                           
       |                                                       
       +--> if self is forked child                            
            -                                                  
            +--> tftp -g -r <file_name> <server_addr>          
```
