```
drivers/base/devtmpfs.c                                                                                  
+---------------+                                                                                         
| devtmpfs_init | : prepare sb, register fs, fork a daemon to mount devtmpfs and endlessly handle request 
+-|-------------+                                                                                         
  |    +----------------+                                                                                 
  |--> | vfs_kern_mount | prepare 'super_block' and mount nowhere                                         
  |    +----------------+                                                                                 
  |    +---------------------+                                                                            
  |--> | register_filesystem | register arg (e.g., 'dev_fs_type') to 'file_systems' list                  
  |    +---------------------+                                                                            
  |    +-------------+                                                                                    
  |--> | kthread_run | ask kthreadd help creating kthread (name, sched-related, cpu mask, ...), wake it up
  |    +-------------+ +-----------+                                                                      
  |                    | devtmpfsd | mount devtmpfs, endlessly handle request (create or remove file)     
  |                    +-----------+                                                                      
  |                                                                                                       
  +--> print "devtmpfs: initialized\n"                                                                    
```

```
drivers/base/devtmpfs.c                                                        
+-----------+                                                                   
| devtmpfsd | : mount devtmpfs, endlessly handle request (create or remove file)
+-|---------+                                                                   
  |    +----------------+                                                       
  |--> | devtmpfs_setup | mount devtmpfs, change cwd/root path                  
  |    +----------------+                                                       
  |    +--------------------+                                                   
  +--> | devtmpfs_work_loop | endlessly handle request (create or remove file)  
       +--------------------+                                                   
```

```
drivers/base/devtmpfs.c                                                                                
+----------------+                                                                                      
| devtmpfs_setup | : mount devtmpfs, change cwd/root path                                               
+-|--------------+                                                                                      
  |    +--------------+                                                                                 
  |--> | ksys_unshare | based on flags, unshare specified resources and switch to the newly created ones
  |    +--------------+                                                                                 
  |    +------------+                                                                                   
  |--> | init_mount | mount properly based on flags                                                     
  |    +------------+                                                                                   
  |    +------------+                                                                                   
  |--> | init_chdir | update pwd of current task to '/..'                                               
  |    +------------+                                                                                   
  |    +-------------+                                                                                  
  +--> | init_chroot | update root of current task to '.'                                               
       +-------------+                                                                                  
```

```
drivers/base/devtmpfs.c                                                      
+--------------------+                                                        
| devtmpfs_work_loop | : endlessly handle request (create or remove file)     
+-|------------------+                                                        
  |                                                                           
  +--> endless loop                                                           
       |                                                                      
       |--> while 'requests' is valid                                         
       |    |                                                                 
       |    |--> have iterator point to the first req                         
       |    |                                                                 
       |    +--> while req exists                                             
       |         |                                                            
       |         |    +--------+                                              
       |         |--> | handle | given mode, create or remove file accordingly
       |         |    +--------+                                              
       |         |                                                            
       |         +--> advance to the next req                                 
       |                                                                      
       |--> set task state = interruptible                                    
       |                                                                      
       |    +----------+                                                      
       +--> | schedule |                                                      
            +----------+                                                      
```

```
drivers/base/devtmpfs.c                                                                             
+--------+                                                                                           
| handle | : given mode, create or remove file accordingly                                           
+-|------+                                                                                           
  |                                                                                                  
  |--> if mode is specified                                                                          
  |    |                                                                                             
  |    |    +---------------+                                                                        
  |    +--> | handle_create | create dentry, call ->mknod to create inode, update attributes in inode
  |         +---------------+                                                                        
  |                                                                                                  
  +--> else                                                                                          
       |                                                                                             
       |    +---------------+                                                                        
       +--> | handle_remove | remove target file                                                     
            +---------------+                                                                        
```

```
drivers/base/devtmpfs.c                                                                            
+---------------+                                                                                   
| handle_create | : create dentry, call ->mknod to create inode, update attributes in inode         
+-|-------------+                                                                                   
  |    +------------------+                                                                         
  |--> | kern_path_create | ensure target dentry exists (create one if not)                         
  |    +------------------+                                                                         
  |    +-----------+                                                                                
  |--> | vfs_mknod | call inode->mknod(e.g., prepare inode of specified type and link to arg dentry)
  |    +-----------+                                                                                
  |    +---------------+                                                                            
  +--> | notify_change | update attributes in inode (truncate inode is size becomes smaller)        
       +---------------+                                                                            
```

```
drivers/base/devtmpfs.c                                                         
+---------------+                                                                
| handle_remove | : remove target file                                           
+-|-------------+                                                                
  |    +------------------+                                                      
  |--> | kern_path_locked | get dentry of the given path                         
  |    +------------------+                                                      
  |                                                                              
  |--> reset inode attributes                                                    
  |                                                                              
  |    +------------+                                                            
  |--> | vfs_unlink | inode nlink--, dentry ref count--                          
  |    +------------+                                                            
  |    +-------------+                                                           
  +--> | delete_path | if parent folder becomes empty, remove it (all the way up)
       +-------------+                                                           
```
