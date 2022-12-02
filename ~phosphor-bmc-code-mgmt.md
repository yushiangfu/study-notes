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
  |
  |--> prepare match rule and callback on "/xyz/openbmc_project/software"
  |    +-------------------------------+
  |    | ItemUpdater::createActivation |
  |    +-------------------------------+
  |
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
item_updater.cpp                                                                                                      
+-------------------------------+                                                                                       
| ItemUpdater::createActivation |: parse info from msg, prepare objects, add to 'versions' and 'activationis' separately
+-|-----------------------------+                                                                                       
  |                                                                                                                     
  |--> read msg into 'interfaces'                                                                                       
  |                                                                                                                     
  |--> given interfaces, save info of property 'Purpose', 'Version', 'Path', 'ExtendedVersion', 'Names'                 
  |                                                                                                                     
  |--> get version_id from last entry of file path                                                                      
  |                                                                                                                     
  +--> if version_id isn't in 'activations' yet                                                                         
       |                                                                                                                
       |--> if purpose is 'bmc' or 'system'                                                                             
       |    |                                                                                                           
       |    |             +------------------------------------+                                                        
       |    +--> result = | ItemUpdater::validateSquashFSImage | check if files in image_update_list are good           
       |                  +------------------------------------+                                                        
       |                                                                                                                
       |--> else (host?), result = ready                                                                                
       |                                                                                                                
       |    if result is ready                                                                                          
       |    -                                                                                                           
       |    +--> make tuple (inventory, activation, bmcInventoryPath(?))                                                
       |                                                                                                                
       |--> prepare version_class                                                                                       
       |                                                                                                                
       |--> add to 'versions'                                                                                           
       |                                                                                                                
       +--> add tuple (association) to 'activations'                                                                    
```

```
item_updater.cpp                                                                              
+------------------------------------+                                                         
| ItemUpdater::validateSquashFSImage | : check if files in image_update_list are good          
+-|----------------------------------+                                                         
  |                                                                                            
  |--> add "image-bmc" to image_update_list                                                    
  |                                                                                            
  |    +-------------------------+                                                             
  |--> | ItemUpdater::checkImage | check if file in image_update_list is good                  
  |    +-------------------------+                                                             
  |                                                                                            
  +--> if not good                                                                             
       |                                                                                       
       |--> add "image-kernel", "image-rofs", "image-rwfs", "image-u-boot" to image_update_list
       |                                                                                       
       |    +-------------------------+                                                        
       +--> | ItemUpdater::checkImage | check if files in image_update_list are good           
            +-------------------------+                                                        
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

### activation

```
activation.cpp                                                                                                  
+------------------------+                                                                                        
| Activation::activation |                                                                                        
+-|----------------------+                                                                                        
  |                                                                                                               
  +--> if value == activating                                                                                     
       |                                                                                                          
       |    +-----------------------------+                                                                       
       |--> | Activation::verifySignature | prepare key & hash, verify {manifest, pubkey, image files, full-image)
       |    +-----------------------------+                                                                       
       |    +----------------------------+                                                                        
       |--> | minimum_ship_level::verify |                                                                        
       |    +----------------------------+                                                                        
       |                                                                                                          
       |--> ensure activation_progress exists                                                                     
       |                                                                                                          
       |--> if the purpose is 'host'                                                                              
       |    |                                                                                                     
       |    |    +---------------------------------------+                                                        
       |    |--> | Activation::subscribeToSystemdSignals | subscribe systemd signals (?)                          
       |    |    +---------------------------------------+                                                        
       |    |                                                                                                     
       |    |--> set initial progress = 20%                                                                       
       |    |                                                                                                     
       |    |    +----------------------------+                                                                   
       |    |--> | Activation::flashWriteHost | call systemd service to flash host bios                           
       |    |    +----------------------------+                                                                   
       |    |                                                                                                     
       |    +--> return                                                                                           
       |                                                                                                          
       |--> set initial progress = 10%                                                                            
       |                                                                                                          
       |    +---------------------------------------+                                                             
       |--> | Activation::subscribeToSystemdSignals | subscribe systemd signals (?)                               
       |    +---------------------------------------+                                                             
       |    +------------------------+                                                                            
       |--> | Activation::flashWrite | copy image to /run/initramfs, and expect it's flashed during reboot        
       |    +------------------------+                                                                            
       |    +---------------------------------+                                                                   
       +--> | Activation::onFlashWriteSuccess | emove object from software manager, reboot if 'apply_time' is immediate
            +---------------------------------+                                                                   
```

```
+-----------------------------+                                                                         
| Activation::verifySignature | : prepare key & hash, verify {manifest, pubkey, image files, full-image)
+-|---------------------------+                                                                         
  |                                                                                                     
  |--> prepare object 'signature'                                                                       
  |    +----------------------+                                                                         
  |    | Signature::Signature | get key_type, hash_type and purpose from manifest file                  
  |    +----------------------+                                                                         
  |                                                                                                     
  |    +-------------------+                                                                            
  +--> | Signature::verify | verify manifest, pubkey, image files, full-image                           
       +-------------------+                                                                            
```

```
+-------------------+                                                                                                                     
| Signature::verify | : verify manifest, pubkey, image files, full-image                                                                  
+-|-----------------+                                                                                                                     
  |    +------------------------------+                                                                                                   
  |--> | Signature::systemLevelVerify | verify manifest and pubkey files                                                                  
  |    +------------------------------+                                                                                                   
  |    +--------------------------------+                                                                                                 
  |--> | Signature::checkAndVerifyImage | verify each file in given list ("image-bmc")                                                    
  |    +--------------------------------+                                                                                                 
  |                                                                                                                                       
  |--> if failed                                                                                                                          
  |    |                                                                                                                                  
  |    |    +--------------------------------+                                                                                            
  |    +--> | Signature::checkAndVerifyImage | verify each file in given list ("image-kernel", "image-rofs", "image-rwfs", "image-u-boot")
  |         +--------------------------------+                                                                                            
  |                                                                                                                                       
  |--> (skip the verification of optioinal images)                                                                                        
  |                                                                                                                                       
  |    +----------------------------+                                                                                                     
  +--> | Signature::verifyFullImage | merge files into one full image, verify it, remove the merged file                                  
       +----------------------------+                                                                                                     
```

```
+------------------------------+                                                                                        
| Signature::systemLevelVerify | : verify manifest and pubkey files                                                     
+-|----------------------------+                                                                                        
  |    +-------------------------------------------+                                                                    
  |--> | Signature::getAvailableKeyTypesFromSystem | search hash_file and key_file, collect their parent path and return
  |    +-------------------------------------------+                                                                    
  |                                                                                                                     
  |--> build pugkey and its signature file name                                                                         
  |                                                                                                                     
  |--> build manifest and its signature file name                                                                       
  |                                                                                                                     
  +--> for each key_type                                                                                                
       |                                                                                                                
       |    +--------------------------------+                                                                          
       |--> | Signature::getKeyHashFileNames | make pair (hash_path, key_path)                                          
       |    +--------------------------------+                                                                          
       |    +-----------------------+                                                                                   
       |--> | Signature::verifyFile | verify file (manifest signature)                                                  
       |    +-----------------------+                                                                                   
       |                                                                                                                
       +--> if valid                                                                                                    
            |                                                                                                           
            |    +-----------------------+                                                                              
            +--> | Signature::verifyFile | verify file (pubkey signature)                                               
                 +-----------------------+                                                                              
```

```
image_verify.cpp
+-------------------------------------------+
| Signature::getAvailableKeyTypesFromSystem | : search hash_file and key_file, collect their parent path and return
+-|-----------------------------------------+
  |
  |--> check if signed_conf exists (should)
  |
  +--> for each file under signed_conf (is it /etc/activationdata/OpenBMC?)
       -
       +--> if it's a hash_file or public_key_file
            -
            +--> add parent_path to key_types
```

```
image_verify.cpp                                                
+-----------------------+                                        
| Signature::verifyFile | : verify file                          
+-|---------------------+                                        
  |    +----------------------------+                            
  |--> | Signature::createPublicRSA | create rsa                 
  |    +----------------------------+                            
  |    +--------------+                                          
  |--> | rsaVerifyCtx | init digest context                      
  |    +--------------+                                          
  |    +-------------------------+                               
  |--> | OpenSSL_add_all_digests | add all algo to internal table
  |    +-------------------------+                               
  |    +----------------------+                                  
  |--> | EVP_get_digestbyname | create hash struct               
  |    +----------------------+                                  
  |    +----------------------+                                  
  |--> | EVP_DigestVerifyInit |                                  
  |    +----------------------+                                  
  |    +------------------------+                                
  |--> | EVP_DigestVerifyUpdate |                                
  |    +------------------------+                                
  |    +-----------------------+                                 
  +--> | EVP_DigestVerifyFinal |                                 
       +-----------------------+                                 
```

```
image_verify.cpp                                                  
+--------------------------------+                                 
| Signature::checkAndVerifyImage | : verify each file in given list
+-|------------------------------+                                 
  |                                                                
  +--> for each bmc_image                                          
       |                                                           
       |    +-----------------------+                              
       +--> | Signature::verifyFile | verify file                  
            +-----------------------+                              
```

```
+----------------------------+                                                                     
| Signature::verifyFullImage | : merge files into one full image, verify it, remove the merged file
+-|--------------------------+                                                                     
  |                                                                                                
  |--> return if it's not a bmc fw                                                                 
  |                                                                                                
  |--> prepare list of full_images                                                                 
  |                                                                                                
  |    +-------------------+                                                                       
  |--> | utils::mergeFiles | merge files into one                                                  
  |    +-------------------+                                                                       
  |    +-----------------------+                                                                   
  |--> | Signature::verifyFile | verify file                                                       
  |    +-----------------------+                                                                   
  |                                                                                                
  +--> remove the merged file                                                                      
```

```
+----------------------------+                                          
| Activation::flashWriteHost | : call systemd service to flash host bios
+-|--------------------------+                                          
  |                                                                     
  |--> prepare method call org.freedesktop.systemd1                     
  |                        /org/freedesktop/systemd1                    
  |                        org.freedesktop.systemd1.Manager             
  |                        StartUnit                                    
  |                                                                     
  |--> service file = "obmc-flash-host-bios@" + versionId + ".service"  
  |                   reference: obmc-flash-host-bios@.service.in       
  |                                                                     
  +--> call()                                                           
```

```
static/flash.cpp                                                                               
+------------------------+                                                                      
| Activation::flashWrite | : copy image to /run/initramfs, and expect it's flashed during reboot
+-|----------------------+                                                                      
  |                                                                                             
  |--> if running_image_slot != 0                                                               
  |    |                                                                                        
  |    |--> prepare method call org.freedesktop.systemd1                                        
  |    |                        /org/freedesktop/systemd1                                       
  |    |                        org.freedesktop.systemd1.Manager                                
  |    |                        StartUnit                                                       
  |    |                                                                                        
  |    |--> service_file = FLASH_ALT_SERVICE_TMPL + versionId + ".service"                      
  |    |                   reference: obmc-flash-bmc-alt@.service.in? not found                 
  |    |                                                                                        
  |    |--> call()                                                                              
  |    |                                                                                        
  |    +--> return                                                                              
  |                                                                                             
  +--> for each bmc_image in update_list                                                        
       -                                                                                        
       +--> copy to "/run/initramfs"                                                            
            (we expect the updater script handles it during reboot)                             
```

```
activation.cpp                                                                                                      
+---------------------------------+                                                                                  
| Activation::onFlashWriteSuccess | : remove object from software manager, reboot if 'apply_time' is immediate       
+-|-------------------------------+                                                                                  
  |                                                                                                                  
  |--> set progress to 100%                                                                                          
  |                                                                                                                  
  |    +-------------------------------------------+                                                                 
  |--> | Activation::unsubscribeFromSystemdSignals | unsubscribe systemd signals                                     
  |    +-------------------------------------------+                                                                 
  |    +-----------------------+                                                                                     
  |--> | updater::storePurpose | save 'purpose' to /var/lib/phosphor-bmc-code-mgmt/                                  
  |    +-----------------------+                                                                                     
  |    +--------------------------------------+                                                                      
  |--> | Activation::deleteImageManagerObject | delete 'version' object from software manager                        
  |    +--------------------------------------+                                                                      
  |    +--------------------------------------+                                                                      
  |--> | ItemUpdater::createActiveAssociation | associate (active, software_version)                                 
  |    +--------------------------------------+                                                                      
  |    +------------------------------------------+                                                                  
  |--> | ItemUpdater::createUpdateableAssociation | associate (updateable, software_version)                         
  |    +------------------------------------------+                                                                  
  |    +-------------------------------------+                                                                       
  |--> | Activation::checkApplyTimeImmediate | read property 'requested_apply_time' to determine if it is 'immediate'
  |    +-------------------------------------+                                                                       
  |                                                                                                                  
  +--> if it is 'immediate'                                                                                          
       |                                                                                                             
       |    +-----------------------+                                                                                
       +--> | Activation::rebootBmc | call systemd service to reboot bmc                                             
            +-----------------------+                                                                                
```

```
+--------------------------------------+                                                
| Activation::deleteImageManagerObject | : delete 'version' object from software manager
+-|------------------------------------+                                                
  |                                                                                     
  |--> query mapper for object of intf xyz.openbmc_project.Software.Version             
  |                                                                                     
  +--> delete 'version' object                                                          
```

```
+-------------------------------------+                                                                         
| Activation::checkApplyTimeImmediate | : read property 'requested_apply_time' to determine if it is 'immediate'
+-|-----------------------------------+                                                                         
  |                                                                                                             
  |--> get service of object: /xyz/openbmc_project/software/apply_time                                          
  |                   iface: xyz.openbmc_project.Software.ApplyTime                                             
  |                                                                                                             
  |--> get property 'RequestedApplyTime'                                                                        
  |                                                                                                             
  +--> check if it is 'immediate'                                                                               
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
| Manager::processImage | : untar, check manifest, prepare 'version' obj and insert to 'verions'
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
watch.cpp                                                                                
+--------------+                                                                          
| Watch::Watch | : monitor 'image upload' folder, register callback to event 'close_write'
+-|------------+                                                                          
  |                                                                                       
  |--> .imageCallback = arg imageCallback                                                 
  |                                                                                       
  |--> ensure 'IMG_UPLOAD_DIR' folder exists                                              
  |                                                                                       
  |    +---------------+                                                                  
  |--> | inotify_init1 | prepare 'inotify' instance                                       
  |    +---------------+                                                                  
  |    +-------------------+                                                              
  |--> | inotify_add_watch | monitor 'close_write' event happened in IMG_UPLOAD_DIR       
  |    +-------------------+                                                              
  |    +-----------------+                                                                
  +--> | sd_event_add_io | register callback 'Watch::callback' for the event              
       +-----------------+                                                                
```

```
 watch.cpp                                                                                               
+-----------------+                                                                                       
| Watch::callback | : handle event (e.g., untar uploaded tarball, check manifest, prepare 'version' and add to 'versions')      
+-|---------------+                                                                                       
  |                                                                                                       
  |--> read data from fd of inotify instance                                                              
  |                                                                                                       
  +--> while we haven't handled all the data                                                              
       |                                                                                                  
       |--> convert data to event                                                                         
       |                                                                                                  
       |--> assemble tar ball path                                                                        
       |                                                                                                  
       |--> call ->imageCallback(tar_ball_path), e.g.,                                                    
       |    +-----------------------+                                                                     
       |    | Manager::processImage | untar, check manifest, prepare 'version' obj and insert to 'verions'
       |    +-----------------------+                                                                     
       |                                                                                                  
       +--> advance offset                                                                                
```

```
https://github.com/openbmc/phosphor-bmc-code-mgmt
```

```
busctl call --verbose \
  xyz.openbmc_project.ObjectMapper \
  /xyz/openbmc_project/object_mapper \
  xyz.openbmc_project.ObjectMapper \
  GetObject \
  sas \
  /xyz/openbmc_project/software \
  1 \
  xyz.openbmc_project.Common.FactoryReset
```

```
busctl call --verbose \
  xyz.openbmc_project.ObjectMapper \
  /xyz/openbmc_project/object_mapper \
  xyz.openbmc_project.ObjectMapper \
  GetSubTreePaths \
  sias \
  /xyz/openbmc_project/software \
  0 \
  1 \
  xyz.openbmc_project.Software.Version
```

```
busctl call --verbose \
  xyz.openbmc_project.ObjectMapper \
  /xyz/openbmc_project/object_mapper \
  xyz.openbmc_project.ObjectMapper \
  GetSubTreePaths \
  sias \
  /xyz/openbmc_project/inventory/ \
  0 \
  1 \
  xyz.openbmc_project.Inventory.Item.Bmc
```
