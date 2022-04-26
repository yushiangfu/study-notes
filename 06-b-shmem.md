> Study case: Linux version 5.15.0 on AST2500 emulation

## Index

- [Introduction](#introduction)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

```
+--------------+                                                                                        
| shmem_create |  prepare an inode of regular file and link it with given dentry                                        
+---|----------+                                                                                        
    |    +-------------+                                                                                
    +--> | shmem_mknod | prepare an inode of specified type and link it with given dentry
         +-------------+                                                                                
```

```
+---------------+                                                                         
| shmem_tmpfile | prepare an inode, set dentry's name using ino, and link inode and dentry
+---|-----------+                                                                         
    |    +-----------------+                                                              
    |--> | shmem_get_inode | allocate an inode, init and install operation sets           
    |    +-----------------+                                                              
    |    +-----------+                                                                    
    +--> | d_tmpfile |                                                                    
         +--|--------+                                                                    
            |                                                                             
            |--> use inode# as dentry name                                                
            |                                                                             
            |    +---------------+                                                        
            +--> | d_instantiate | fill inode info in dentry and link them                
                 +---------------+                                                        
```

```
+-------------+
| shmem_mknod | prepare an inode of specified type and link it with given dentry
+---|---------+
    |    +-----------------+
    |--> | shmem_get_inode |
    |    +----|------------+
    |         |    +---------------------+
    |         |--> | shmem_reserve_inode | get a free inode number
    |         |    +---------------------+
    |         |    +-----------+
    |         |--> | new_inode | allocate a complex or regular inode, init and add to sb's list
    |         |    +-----------+
    |         |
    |         +--> further set up inode and install operation sets of different file types
    |
    |    +---------------+
    +--> | d_instantiate | fill inode info in dentry and link them
         +---------------+
```

```
+-------------+                                                                          
| shmem_mkdir | prepare an inode of directory and link it with given dentry              
+---|---------+                                                                          
    |    +-------------+                                                                 
    +--> | shmem_mknod | prepare an inode of specified type and link it with given dentry
         +-------------+                                                                 
```

```
+--------------+                                              
| shmem_unlink | inode nlink--, dentry ref count--
+---|----------+                                              
    |    +------------------+                                 
    |--> | shmem_free_inode | update info only (free_inodes++)
    |    +------------------+                                 
    |    +------------+                                       
    |--> | drop_nlink | inode nlink--                         
    |    +------------+                                       
    |    +------+                                             
    +--> | dput | dentry ref count--, release might happen    
         +------+                                             
```

```
+-------------+                                                  
| shmem_rmdir | inode nlink-- -- --, dentry ref count--          
+---|---------+                                                  
    |                                                            
    |--> if the dir has at least one child dentry, return error  
    |                                                            
    |--> inode nlink--                                           
    |                                                            
    |--> inode nlink-- (why twice? inode of dentry != dir inode?)
    |                                                            
    |    +--------------+                                        
    +--> | shmem_unlink | inode nlink--, dentry ref count--      
         +--------------+                                        
```

```
+---------------+                                                                
| shmem_symlink | preapre inode, save dst path to it, and link with dentry       
+---|-----------+                                                                
    |    +-----------------+                                                     
    |--> | shmem_get_inode | allocate an inode, init and install operation sets  
    |    +-----------------+                                                     
    |                                                                            
    |--> save the dst path to inode                                              
    |                                                                            
    |    +---------------+                                                       
    +--> | d_instantiate | fill inode info in dentry and link them               
         +---------------+                                                       
```

```
+------------+                                                   
| shmem_link | fill inode info in the new dentry and link them   
+--|---------+                                                   
   |    +---------------+                                        
   +--> | d_instantiate | fill inode info in dentry and link them
        +---------------+                                        
```

```
 +---------------+                                          
 | shmem_rename2 | unlink dst dentry if it exists           
 +---|-----------+                                          
     |                                                      
     +--> if target dentry exists already                   
     |                                                      
     |    +--------------+                                  
     +--> | shmem_unlink | inode nlink--, dentry ref count--
          +--------------+                                                                 
```

```
+----------------+                                                       
| shmem_get_link | get the virtual address of inode mapping at offset = 0
+---|------------+                                                       
    |                                                                    
    |--> get page of inode mapping at offset 0                           
    |                                                                    
    +--> return the virtual address that it represents                   
```

```
+----------------------+                                                                                                  
| shmem_truncate_range | for each page in truncated range, unmap and detatch it from mapping
+-----|----------------+                                                                                                  
      |    +------------------+                                                                                           
      +--> | shmem_undo_range | for each page in truncated range, unmap and detatch it from mapping
           +----|-------------+                                                                                           
                |                                                                                                         
                |--> while we can still fill pvec from mapping                                                            
                |                                                                                                         
                |------> for each page in pvec                                                                            
                |                                                                                                         
                |            +---------------------+                                                                      
                |----------> | truncate_inode_page | unmap and invalidate page, detatch it from mapping, and has its ref--
                |            +---------------------+                                                                      
                |                                                                                                         
                |--> for index between start and end                                                                      
                |                                                                                                         
                |        +------------------+                                                                             
                |------> | find_get_entries | fill pvec from mapping                                                      
                |        +------------------+                                                                             
                |                                                                                                         
                +------> break loop if nothing to fill                                                                    
                |                                                                                                         
                |------> for each page in pvec                                                                            
                |                                                                                                         
                |            +-----------------+                                                                          
                |----------> | shmem_free_swap | detach from mapping (what's the difference from 'truncate_inode_page'?)  
                |            +-----------------+                                                                          
                |    +--------------------+                                                                               
                +--> | shmem_recalc_inode | update inode info                                                             
                     +--------------------+                                                                               
```

```
+---------------+                                                                                            
| shmem_setattr | truncate inode if size becomes to smaller, copy attributes to inode                        
+---|-----------+                                                                                            
    |                                                                                                        
    |--> if it's about size change on regular file                                                           
    |                                                                                                        
    |------> update size in inode                                                                            
    |                                                                                                        
    |------> if new size is smaller                                                                          
    |                                                                                                        
    |            +----------------------+                                                                    
    |--------->  | shmem_truncate_range | for each page in truncated range, unmap and detatch it from mapping
    |            +----------------------+                                                                    
    |    +--------------+                                                                                    
    +--> | setattr_copy | copy attributes to inode                                                           
         +--------------+                                                                                    
```

## <a name="reference"></a> Reference

(TBD)
