```
+-----------------------------+                                                                       
| configfs_register_subsystem | : prepare sb of config_fs, alloc a folder and publish it to user space
+-|---------------------------+                                                                       
  |    +--------------+                                                                               
  |--> | new_fragment | ???                                                                           
  |    +--------------+                                                                               
  |    +-----------------+                                                                            
  |--> | configfs_pin_fs |  prepare sb for config_fs and return its dentry                            
  |    +-----------------+                                                                            
  |    +------------+                                                                                 
  |--> | link_group | ???                                                                             
  |    +------------+                                                                                 
  |    +--------------+                                                                               
  |--> | d_alloc_name | alloc child dentry of sb dentry                                               
  |    +--------------+                                                                               
  |                                                                                                   
  |--> if child dentry                                                                                
  |                                                                                                   
  |        +-------+                                                                                  
  |------> | d_add | add to hash table                                                                
  |        +-------+                                                                                  
  |        +-----------------------+                                                                  
  |------> | configfs_attach_group | ???                                                              
  |        +-----------------------+                                                                  
  |        +------------------------+                                                                 
  +------> | configfs_dir_set_ready | allow user space to create entries under this                   
           +------------------------+                                                                 
```

```
+-----------------+                                                 
| configfs_pin_fs | : prepare sb for config_fs and return its dentry
+-|---------------+                                                 
  |    +---------------+                                            
  +--> | simple_pin_fs | prepare sb for arg fs_type                 
       +---------------+                                            
```

```
+---------------+                                                    
| simple_pin_fs | : prepare sb for arg fs_type                       
+-|-------------+                                                    
  |                                                                  
  |--> if mount doesn't exist yet                                    
  |                                                                  
  |        +----------------+                                        
  +------> | vfs_kern_mount | prepare 'super_block' and mount nowhere
           +----------------+                                        
```
