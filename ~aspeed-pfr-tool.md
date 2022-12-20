not an independent repo, but part of the aspeed openbmc

```
main.c                                                                                                            
+------+                                                                                                           
| main |                                                                                                           
+-|----+                                                                                                           
  |                                                                                                                
  |--> conf_path = "/usr/share/pfrconfig/aspeed-pfr-tool.conf"                                                     
  |                                                                                                                
  |--> handle options                                                                                              
  |                                                                                                                
  |    +---------------------+                                                                                     
  |--> | parseConfigElements | parse config file                                                                   
  |    +---------------------+                                                                                     
  |                                                                                                                
  |--> if bus option is provided, overwite the one from config file                                                
  |                                                                                                                
  |--> if rot_addr option is provided, overwite the one from config file                                           
  |                                                                                                                
  |--> if debug option is provided, print the argument info                                                        
  |                                                                                                                
  |    +------------+                                                                                              
  |--> | i2cOpenDev | open i2c_dev with given (bus, rot_addr)                                                      
  |    +------------+                                                                                              
  |                                                                                                                
  |--> if read option is specified                                                                                 
  |    -                                                                                                           
  |    +--> read i2c data (byte or block), print it                                                                
  |--> if write option is specified                                                                                
  |    -                                                                                                           
  |    +--> write i2c data (byte or block)                                                                         
  |--> if provision option is specified                                                                            
  |    -    +-----------+                                                                                          
  |    +--> | provision | perform provision action (show, lock, or provision)                                      
  |         +-----------+                                                                                          
  |--> if unprovision option is specified                                                                          
  |    -    +-------------+                                                                                        
  |    +--> | unprovision | i2c write byte (provision_erase)                                                       
  |         +-------------+                                                                                        
  |--> if checkpoint option is specified                                                                           
  |    -    +------------+                                                                                         
  |    +--> | checkpoint | given cmd, write the corresponding byte to checkpoint                                   
  |         +------------+                                                                                         
  |--> if status option is specified                                                                               
  |    -    +-------------+                                                                                        
  |    +--> | show_status | read all kinds of status and print                                                     
  |         +-------------+                                                                                        
  |--> if info option is specified                                                                                 
  |    -    +-----------+                                                                                          
  |    +--> | show_info | read and print info of pch_pfm_active, bmc_pfm_active, pch_pfm_recovery, bmc_pfm_recovery
  |         +-----------+                                                                                          
  +--> close i2c_dev file                                                                                          
```

```
main.c                                                            
+---------------------+                                            
| parseConfigElements | : parse config file                        
+-|-------------------+                                            
  |                                                                
  +--> while we can still read a line                              
       -                                                           
       +--> parse below key and save value in 'args' struct        
                                                                   
            i2c_bus, rot_address,                                  
            bmc_active_pfm_offset, staging_offset, recovery_offset,
            pch_active_pfm_offset, staging_offset, recovery_offset 
```

```
provision.c                                                                                     
+-----------+                                                                                    
| provision | : perform provision action (show, lock, or provision)                              
+-|---------+                                                                                    
  |                                                                                              
  |--> if cmd is 'show'                                                                          
  |    |                                                                                         
  |    |    +---------------+                                                                    
  |    +--> | provisionShow | show provision info (bmc offsets, pch offsets, root key)           
  |         +---------------+                                                                    
  |                                                                                              
  |--> elif cmd is 'lock                                                                         
  |    |                                                                                         
  |    |    +---------------+                                                                    
  |    +--> | provisionLock | lock provision                                                     
  |         +---------------+                                                                    
  |                                                                                              
  +--> else                                                                                      
       |                                                                                         
       |    +-------------+                                                                      
       +--> | doProvision | get root key hash, write bmc_offsets/pch_offsets/root_key_hash to ufm
            +-------------+                                                                      
```

```
provision.c                                                                
+---------------+                                                           
| provisionShow | : show provision info (bmc offsets, pch offsets, root key)
+-|-------------+                                                           
  |    +--------------------+                                               
  |--> | readUfmProvFifoCmd | read bmc offset from ufm                      
  |    +--------------------+                                               
  |                                                                         
  |--> print bmc active_pfm_offset, recovery_offset, staging_offset         
  |                                                                         
  |    +--------------------+                                               
  |--> | readUfmProvFifoCmd | read pch offset from ufm                      
  |    +--------------------+                                               
  |                                                                         
  |--> print pch active_pfm_offset, recovery_offset, staging_offset         
  |                                                                         
  |    +--------------------+                                               
  |--> | readUfmProvFifoCmd | read root key from ufm                        
  |    +--------------------+                                               
  |    +--------------+                                                     
  +--> | printRawData |                                                     
       +--------------+                                                     
```

```
provision.c                                                                                  
+--------------------+                                                                        
| readUfmProvFifoCmd | : flush read fifo, trigger provision cmd, i2c read data and save in buf
+-|------------------+                                                                        
  |                                                                                           
  |--> i2c write byte to flush read fifo                                                      
  |                                                                                           
  |    +----------------------------+                                                         
  |--> | waitUntilUfmCmdTriggerExec | wait till the ufm_cmd is executed                       
  |    +----------------------------+                                                         
  |    +-------------------------------+                                                      
  |--> | waitUntilUfmProvStatusCmdDone | wait till cmd is done                                
  |    +-------------------------------+                                                      
  |                                                                                           
  |--> return error if any of the above two fails                                             
  |                                                                                           
  |--> i2c write byte (provision_cmd)                                                         
  |                                                                                           
  |--> i2c write byte (trigger)                                                               
  |                                                                                           
  |--> wait till cmd is executed and done                                                     
  |                                                                                           
  +--> for each byte in len                                                                   
       -                                                                                      
       +--> i2c read byte (read_fifl) and save in buf[]                                       
```

```
provision.c                                                                  
+----------------------------+                                                
| waitUntilUfmCmdTriggerExec | : wait till the ufm_cmd is executed            
+-|--------------------------+                                                
  |                                                                           
  |--> prepare mask = execute | write_fifo | read_fifo                        
  |                                                                           
  +--> while we can still retry                                               
       |                                                                      
       |--> i2c read byte (cmd_trigger)                                       
       |                                                                      
       |--> if read_byte & mask (means it's not yet executed)                 
       |    -                                                                 
       |    +--> print "UFM Command Trigger: Not execute, wait 20ms %d time\n"
       |                                                                      
       |--> else                                                              
       |    -                                                                 
       |    +--> return                                                       
       |                                                                      
       +--> sleep 20ms                                                        
```

```
provision.c                                                                          
+-------------------------------+                                                     
| waitUntilUfmProvStatusCmdDone | : wait till cmd is done                             
+-|-----------------------------+                                                     
  |                                                                                   
  +--> while we can still retry                                                       
       |                                                                              
       |    +-----------------+                                                       
       |--> | i2cReadByteData | i2c read byte (provision status)                      
       |    +-----------------+                                                       
       |                                                                              
       |--> if it's still busy                                                        
       |    -                                                                         
       |    +--> print "UFM Provisioning Status: Command busy..., wait 20ms %d time\n"
       |                                                                              
       |--> else                                                                      
       |    -                                                                         
       |    +--> return                                                               
       |                                                                              
       +--> sleep 20ms                                                                
```

```
provision.c                                         
+---------------+                                    
| provisionLock | : lock provision                   
+-|-------------+                                    
  |                                                  
  |--> i2c write data (provision_cmd = provision_end)
  |                                                  
  |--> i2c write data (trigger)                      
  |                                                  
  +--> wait till the cmd is executed and done        
```

```
provision.c                                                                               
+-------------+                                                                            
| doProvision | : get root key hash, write bmc_offsets/pch_offsets/root_key_hash to ufm    
+-|-----------+                                                                            
  |    +----------------+                                                                  
  |--> | getRootKeyHash | get hash of the root key                                         
  |    +----------------+                                                                  
  |                                                                                        
  |--> if debug option is specified                                                        
  |    |                                                                                   
  |    |    +--------------+                                                               
  |    +--> | printRawData |                                                               
  |         +--------------+                                                               
  |    +--------------------------------+                                                  
  |--> | writeUfmProvBmcPchRegionOffset | prepare bmc offsets and pch offsets, write to ufm
  |    +--------------------------------+                                                  
  |    +---------------------+                                                             
  +--> | writeUfmProvFifoCmd | write (root_key hash) to fifo and provision_cmd             
       +---------------------+                                                             
```

```
provision.c                                                                    
+----------------+                                                              
| getRootKeyHash | : get hash of the root key                                   
+-|--------------+                                                              
  |    +-----------------------+                                                
  |--> | extractQxQyFromPubkey | extrace qx and qy from pubkey                  
  |    +-----------------------+                                                
  |                                                                             
  |--> reverse pubkey_x and pubkey_y and place them in buffer (little endianess)
  |                                                                             
  |    +------------+                                                           
  +--> | hashBuffer | generate data hash                                        
       +------------+                                                           
```

```
provision.c                                                            
+-----------------------+                                               
| extractQxQyFromPubkey | : extrace qx and qy from pubkey               
+-|---------------------+                                               
  |                                                                     
  |--> prepare bio based on file                                        
  |                                                                     
  |    +---------------------+                                          
  |--> | PEM_read_bio_PUBKEY | get enclosed_key from bio                
  |    +---------------------+                                          
  |    +----------------------------------+                             
  |--> | EVP_PKEY_get1_encoded_public_key | get pubkey from enclosed_key
  |    +----------------------------------+                             
  |                                                                     
  +--> fill qx and qy based on pubkey (former half & latter half)       
```

```
 provision.c                       
+------------+                     
| hashBuffer | : generate data hash
+-|----------+                     
  |                                
  |--> prepare md based on sha algo
  |                                
  |    +----------------+          
  |--> | EVP_MD_CTX_new |          
  |    +----------------+          
  |    +-------------------+       
  |--> | EVP_DigestInit_ex |       
  |    +-------------------+       
  |    +------------------+        
  |--> | EVP_DigestUpdate |        
  |    +------------------+        
  |    +--------------------+      
  |--> | EVP_DigestFinal_ex |      
  |    +--------------------+      
  |    +-----------------+         
  +--> | EVP_MD_CTX_free |         
       +-----------------+         
```

```
provision.c                                                                          
+--------------------------------+                                                    
| writeUfmProvBmcPchRegionOffset | : prepare bmc offsets and pch offsets, write to ufm
+-|------------------------------+                                                    
  |                                                                                   
  |--> fill three bmc offsets in bmc_offset[]                                         
  |                                                                                   
  |--> fill three pch offsets in pch_offset[]                                         
  |                                                                                   
  |--> if debug option is specified                                                   
  |    |                                                                              
  |    |    +--------------+                                                          
  |    +--> | printRawData | bmc_offset[]                                             
  |    |    +--------------+                                                          
  |    |    +--------------+                                                          
  |    +--> | printRawData | pch_offset[]                                             
  |         +--------------+                                                          
  |    +---------------------+                                                        
  |--> | writeUfmProvFifoCmd | write (bmc_offset[]) to fifo and provision_cmd         
  |    +---------------------+                                                        
  |    +---------------------+                                                        
  +--> | writeUfmProvFifoCmd | write (pch_offset[]) to fifo and provision_cmd         
       +---------------------+                                                        
```

```
provision.c                                             
+---------------------+                                  
| writeUfmProvFifoCmd | : write to fifo and provision_cmd
+-|-------------------+                                  
  |                                                      
  |--> i2c write byte (trigger = flush_write_fifo)       
  |                                                      
  |--> wait till the cmd is executed and done            
  |                                                      
  |--> for each byte in len                              
  |    -                                                 
  |    +--> i2c write byte (write_fifo)                  
  |                                                      
  |--> i2c write byte (provision_cmd)                    
  |                                                      
  |--> i2c write byte (trigger)                          
  |                                                      
  +--> wait till the cmd is executed and done            
```

```
checkpoint.c                                                            
+------------+                                                           
| checkpoint | : given cmd, write the corresponding byte to checkpoint   
+-|----------+                                                           
  |                                                                      
  |--> if cmd is 'start'                                                 
  |    |                                                                 
  |    |    +-----------------+                                          
  |    +--> | checkpointStart | i2c write byte (checkpoint = start)      
  |         +-----------------+                                          
  |                                                                      
  |--> elif cmd is 'pause'                                               
  |    |                                                                 
  |    |    +-----------------+                                          
  |    +--> | checkpointPause | i2c write byte (checkpoint = pause)      
  |         +-----------------+                                          
  |                                                                      
  |--> elif cmd is 'resume'                                              
  |    |                                                                 
  |    |    +------------------+                                         
  |    +--> | checkpointResume | i2c write byte (checkpoint = resume)    
  |         +------------------+                                         
  |                                                                      
  +--> elif cmd is 'complete'                                            
       |                                                                 
       |    +--------------------+                                       
       +--> | checkpointComplete | i2c write byte (checkpoint = complete)
            +--------------------+                                       
```

```
status.c                                                                                                
+-------------+                                                                                          
| show_status | : read all kinds of status and print                                                     
+-|-----------+                                                                                          
  |    +-------------+                                                                                   
  |--> | get_cpld_id | i2c read byte (cpld_static_id)                                                    
  |    +-------------+                                                                                   
  |    +--------------+                                                                                  
  |--> | get_cpld_ver | i2c read byte (cpld_release_version)                                             
  |    +--------------+                                                                                  
  |    +--------------+                                                                                  
  |--> | get_cpld_svn | i2c read byte (cpld_svn)                                                         
  |    +--------------+                                                                                  
  |    +----------------+                                                                                
  |--> | get_plat_state | i2c read byte (platform_state), and convert to meaningful string               
  |    +----------------+                                                                                
  |    +--------------------+                                                                            
  |--> | get_recovery_count | i2c read byte (recovery_count)                                             
  |    +--------------------+                                                                            
  |    +--------------------------+                                                                      
  |--> | get_last_recovery_reason | i2c read byte (recovery_reason), and convert to meaningful string    
  |    +--------------------------+                                                                      
  |    +-----------------------+                                                                         
  |--> | get_panic_event_count | i2c read byte (panic_event_count)                                       
  |    +-----------------------+                                                                         
  |    +-----------------------+                                                                         
  |--> | get_last_panic_reason | i2c read byte (last_panic_reason), and convert to meaningful string     
  |    +-----------------------+                                                                         
  |    +---------------+                                                                                 
  |--> | get_major_err | i2c read byte (major_error_code), and convert to meaningful string              
  |    +---------------+                                                                                 
  |                                                                                                      
  |--> if major_error_code <= 2                                                                          
  |    |                                                                                                 
  |    |    +--------------------+                                                                       
  |    +--> | get_minor_auth_err | i2c read byte (minor_error_code), and convert to meaningful string    
  |         +--------------------+                                                                       
  |                                                                                                      
  |--> else                                                                                              
  |    |                                                                                                 
  |    |    +----------------------+                                                                     
  |    +--> | get_minor_update_err | i2c read byte (minor_error_code), and convert to meaningful string  
  |         +----------------------+                                                                     
  |    +-----------------------------+                                                                   
  +--> | get_ufm_provisioning_status | i2c read byte (provision_status), and convert to meaningful string
       +-----------------------------+                                                                   
```
