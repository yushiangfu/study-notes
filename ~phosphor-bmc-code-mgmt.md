### obmc-flash-bmc

```
obmc-flash-bmc                                                                          
+----------------+                                                                       
| obmc-flash-bmc |                                                                       
+-|--------------+                                                                       
  |                                                                                      
  |--> switch                                                                            
  |--> case mtduboot                                                                     
  |    -    +-----------+                                                                
  |    +--> | mtd_write | flash image (image-u-boot) to target mtd                       
  |         +-----------+                                                                
  |--> (skip ubi related cases, see no ubi device in my study case)                      
  |    -    +-----------------+                                                          
  |    +--> | backup_env_vars | copy kernelname/ubiblock/root from env_var to config_file
  |         +-----------------+                                                          
  |--> case updateubootvars                                                              
  |    -    +-----------------+                                                          
  |    +--> | update_env_vars |                                                          
  |         +-----------------+                                                          
  |--> case rebootguardenable                                                            
  |    -    +-------------------+                                                        
  |    +--> | rebootguardenable | enable reboot-guard                                    
  |         +-------------------+                                                        
  |--> case rebootguarddisable                                                           
  |    -    +--------------------+                                                       
  |    +--> | rebootguarddisable |                                                       
  |         +--------------------+                                                       
  |--> case mirroruboot                                                                  
  |    -    +-------------+                                                              
  |    +--> | mirroruboot | mirror uboot to alt                                          
  |         +-------------+                                                              
  |--> case mmc                                                                          
  |    -    +------------+                                                               
  |    +--> | mmc_update | flash bmc image to mmc                                        
  |         +------------+                                                               
  |--> case mmc-mount                                                                    
  |    -    +-----------+                                                                
  |    +--> | mmc_mount | mount mmc                                                      
  |         +-----------+                                                                
  |--> case mmc-remove                                                                   
  |    -    +------------+                                                               
  |    +--> | mmc_remove | remove bmc image in mmc                                       
  |         +------------+                                                               
  |--> case mmc-setprimary                                                               
  |    -    +----------------+                                                           
  |    +--> | mmc_setprimary | set bootside=bootside, and next time it boots from there  
  |         +----------------+                                                           
  |--> case static-altfs                                                                 
  |     -                                                                                
  |     +--> mount alternative                                                           
  |                                                                                      
  +--> case umount-static-altfs                                                          
        -                                                                                
        +--> unmount alternative                                                         
```

```
obmc-flash-bmc                                                                
+-----------------+                                                            
| backup_env_vars | : copy kernelname/ubiblock/root from env_var to config_file
+-|---------------+                                                            
  |    +---------------------+                                                 
  |--> | copy_env_var_to_alt | copy kernel name from env_var to config_file    
  |    +---------------------+                                                 
  |    +----------------------+                                                
  |--> | copy_ubiblock_to_alt | copy ubiblock from env_var to config_file      
  |    +----------------------+                                                
  |    +------------------+                                                    
  +--> | copy_root_to_alt | copy root from env_var to config_file              
       +------------------+                                                    
```

```
obmc-flash-bmc                                                              
+-------------+                                                               
| mirroruboot | : mirror uboot to alt                                         
+-|-----------+                                                               
  |                                                                           
  |--> determine device file of 'u-boot' and 'alt-u-boot'                     
  |                                                                           
  |--> calculate checksum for each of them                                    
  |                                                                           
  +--> if the checksums differ                                                
       |                                                                      
       |--> determine device file of 'u-boot-env' and 'alt-u-boot-env'        
       |                                                                      
       |    +----------+                                                      
       |--> | mtd_copy | mirror bmc to alt                                    
       |    +----------+                                                      
       |    +----------+                                                      
       |--> | mtd_copy | mirror bmc env to alt                                
       |    +----------+                                                      
       |    +----------------------+                                          
       |--> | copy_ubiblock_to_alt | copy ubiblock from env_var to config_file
       |    +----------------------+                                          
       |    +------------------+                                              
       +--> | copy_root_to_alt | copy root from env_var to config_file        
            +------------------+                                              
```

```
obmc-flash-bmc                                                             
+------------+                                                              
| mmc_update | : flash bmc image to mmc                                     
+-|----------+                                                              
  |                                                                         
  |--> compare uboot@dev and uboot@img                                      
  |                                                                         
  |--> if they are different                                                
  |    -                                                                    
  |    +--> mirror data from img to dev                                     
  |                                                                         
  |    +-------------------------+                                          
  |--> | mmc_get_secondary_label | get the secondary label (non-running one)
  |    +-------------------------+                                          
  |                                                                         
  |--> extract image-kernel and write to secondary dev                      
  |                                                                         
  |--> extract image-rofs and write to secondary dev                        
  |                                                                         
  |--> (can't find 'sgdisk' utility on system)                              
  |                                                                         
  |    +-----------+                                                        
  |--> | partprobe | inform system of partition table change                
  |    +-----------+                                                        
  |                                                                         
  |--> if host fw exist (bios?)                                             
  |    -                                                                    
  |    +--> copy to somewhere and mount as read-only                        
  |                                                                         
  |    +-------------+                                                      
  +--> | set_flashid | update property xyz.openbmc_project.Common.FilePath  
       +-------------+                                                      
```

```
obmc-flash-bmc                                 
+-----------+                                    
| mmc_mount | : mount mmc                        
+-|---------+                                    
  |                                              
  |--> get primary id and secondary id           
  |                                              
  |--> create primary and secondary folders      
  |                                              
  +--> mount image@mmc to the folders separately?
```

```
obmc-flash-bmc                                                       
+------------+                                                        
| mmc_remove | : remove bmc image in mmc                              
+-|----------+                                                        
  |    +-----------------------+                                      
  |--> | mmc_get_primary_label |                                      
  |    +-----------------------+                                      
  |                                                                   
  |--> return if flash_id == primary_id (don't remove the running one)
  |                                                                   
  |--> use dd to destroy 'boot' and 'rofs'                            
  |                                                                   
  |--> ensure host_fw_alt isn't mounted                               
  |                                                                   
  +--> if host_fw is mounted, remove something in it                  
```

### phosphor-download-manager

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

### phosphor-image-updater

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

### phosphor-version-software-manager

```
image_manager_main.cpp                                   
+------+                                                  
| main |                                                  
+-|----+                                                  
  |                                                       
  |--> request name "xyz.openbmc_project.Software.Version"
  |                                                       
  |                  +-----------------------+            
  |--> prepare watch | Manager::processImage |            
  |                  +-----------------------+            
  |                                                       
  |    +---------------+                                  
  +--> | sd_event_loop |                                  
       +---------------+                                  
```

```
image_manager.cpp                                                   
+-----------------------+                                            
| Manager::processImage |                                            
+-|---------------------+                                            
  |                                                                  
  |--> prepare tmp_path                                              
  |                                                                  
  |--> untar the tarball to tmp_path                                 
  |                                                                  
  |--> check 'version' in MANIFEST (can't be empty)                  
  |                                                                  
  |--> get current machine from /etc/os-release                      
  |                                                                  
  |--> get machine from MANIFEST                                     
  |                                                                  
  |--> these two 'machine' should match                              
  |                                                                  
  |--> check 'purpose' in MANIFEST (can't be empty)                  
  |                                                                  
  |    +--------------------+                                        
  |--> | getSoftwareObjects | call obj_mapper to get software objects
  |    +--------------------+                                        
  |                                                                  
  |--> rename tmp_dir to image_dir                                   
  |                                                                  
  +--> prepare 'Version' obj and insert to 'versions'                
```

```
https://github.com/openbmc/phosphor-bmc-code-mgmt
```
