> Study case: Linux version 5.15.0 on AST2500 emulation

## Index

- [Introduction](#introduction)
- [To-Do List](#to-do-list)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

```
+-----------------------+                                                                                                      
| vfs_caches_init_early |                                                                                                      
+-----|-----------------+                                                                                                      
      |                                                                                                                        
      |--> init 'in_lookup_hashtable'                                                                                          
      |                                                                                                                        
      |    +-------------------+                                                                                               
      |--> | dcache_init_early |                                                                                               
      |    +----|--------------+                                                                                               
      |         |    +-------------------------+                                                                               
      |         +--> | alloc_large_system_hash |                                                                               
      |              +------|------------------+                                                                               
      |                     |                                                                                                  
      |                     |--> allocate and init hash table                                                                  
      |                     |                                                                                                  
      |                     +--> print '[    0.000000] Dentry cache hash table entries: 65536 (order: 6, 262144 bytes, linear)'
      |                                                                                                                        
      |    +------------------+                                                                                                
      +--> | inode_init_early |                                                                                                
           +----|-------------+                                                                                                
                |    +-------------------------+                                                                               
                +--> | alloc_large_system_hash |                                                                               
                     +------|------------------+                                                                               
                            |                                                                                                  
                            |--> allocate and init hash table                                                                  
                            |                                                                                                  
                            +--> print '[    0.000000] Inode-cache hash table entries: 32768 (order: 5, 131072 bytes, linear)' 
```

```
+-----------------+                                                          
| vfs_caches_init |                                                          
+----|------------+                                                          
     |                                                                       
     |--> prepare kmem cache for 'path' (4096 characters at most)            
     |                                                                       
     |    +-------------+                                                    
     |--> | dcache_init | preapre kmem cache for 'dentry'                    
     |    +-------------+                                                    
     |    +------------+                                                     
     |--> | inode_init | preapre kmem cache for 'inode'                      
     |    +------------+                                                     
     |    +------------+                                                     
     |--> | files_init | preapre kmem cache for 'file'                       
     |    +------------+                                                     
     |    +---------------------+                                            
     |--> | files_maxfiles_init | determine the max number of files in system
     |    +---------------------+                                            
     |    +----------+                                                       
     |--> | mnt_init | (traced separately)                                   
     |    +----------+                                                       
     |    +-----------------+                                                
     |--> | bdev_cache_init |                                                
     |    +----|------------+                                                
     |         |                                                             
     |         |--> prepare kmem cache for 'bdev_indoe'                      
     |         |                                                             
     |         |    +---------------------+                                  
     |         |--> | register_filesystem | (traced separately)              
     |         |    +---------------------+                                  
     |         |    +------------+                                           
     |         +--> | kern_mount | (traced separately)                       
     |              +------------+                                           
     |    +-------------+                                                    
     +--> | chrdev_init | init 'cdev_map'                                    
          +-------------+                                                    
```

```
+----------+
| mnt_init |
+--|-------+
   |
   |--> prepare kmem cache for 'mount'
   |
   |--> allocate 'mount_hashtable'
   |
   |--> allocate 'mountpoint_hashtable'
   |
   |    +-------------+
   |--> | kernfs_init | prepare kmem cache for 'kernfs_node' and 'kernfs_iattrs'
   |    +-------------+
   |    +------------+
   |--> | sysfs_init |
   |    +--|---------+
   |       |    +--------------------+
   |       +--> | kernfs_create_root | (traced separately)
   |       |    +--------------------+
   |       |    +---------------------+
   |       +--> | register_filesystem | (traced separately)
   |            +---------------------+
   |    +------------+
   |--> | shmem_init |
   |    +--|---------+
   |       |    +-----------------------+
   |       +--> | shmem_init_inodecache | prepare kmem cache for 'shmem_inode_info'
   |            +-----------------------+
   |            +---------------------+
   |            | register_filesystem | (traced separately)
   |            +---------------------+
   |            +------------+
   |            | kern_mount | (traced separately)
   |            +------------+
   |    +-------------+
   |--> | init_rootfs | (not our concern)
   |    +-------------+
   |    +-----------------+
   +--> | init_mount_tree |
        +----|------------+
             |    +----------------+
             |--> | vfs_kern_mount | (traced separately)
             |    +----------------+
             |    +------------+
             |--> | set_fs_pwd |
             |    +------------+
             |    +-------------+
             +--> | set_fs_root |
                  +-------------+

```

```
+----------+                                                                                                
| sys_open |                                                                                                
+--|-------+                                                                                                
   |    +-------------+                                                                                     
   +--> | do_sys_open |                                                                                     
        +---|---------+                                                                                     
            |    +----------------+                                                                         
            +--> | do_sys_openat2 |                                                                         
                 +---|------------+                                                                         
                     |    +---------------------+                                                           
                     |--> | get_unused_fd_flags | find an unused fd from file desc table                    
                     |    +---------------------+                                                           
                     |    +--------------+                                                                  
                     |--> | do_filp_open |                                                                  
                     |    +---|----------+                                                                  
                     |        |    +-------------+                                                          
                     |        +--> | path_openat |                                                          
                     |             +---|---------+                                                          
                     |                 |    +------------------+                                            
                     |                 |--> | alloc_empty_file | allocate 'file'                            
                     |                 |    +------------------+                                            
                     |                 |                                                                    
                     |                 |--> walk path to get the final dentry                               
                     |                 |                                                                    
                     |                 |    +---------+                                                     
                     |                 +--> | do_open |                                                     
                     |                      +--|------+                                                     
                     |                         |    +----------+                                            
                     |                         +--> | vfs_open |                                            
                     |                              +--|-------+                                            
                     |                                 |    +----------------+                              
                     |                                 +--> | do_dentry_open |                              
                     |                                      +---|------------+                              
                     |                                          |                                           
                     |                                          |--> get fops from inode and install to file
                     |                                          |                                           
                     |                                          +--> call ->open(), e.g.,                   
                     |                                               +-------------+                        
                     |                                               | blkdev_open |                        
                     |                                               +-------------+                        
                     |    +------------+                                                                    
                     +--> | fd_install | table[fd] = file                                                   
                          +------------+                                                                    
```



```
+----------+                                         
| sys_read |                                         
+--|-------+                                         
   |    +-----------+                                
   +--> | ksys_read |                                
        +--|--------+                                
           |                                         
           |--> get offset from 'file'               
           |                                         
           |    +----------+                         
           |--> | vfs_read |                         
           |    +--|-------+                         
           |       |                                 
           |       |--> if ->read() exists           
           |       |                                 
           |       |------> call ->read()            
           |       |                                 
           |       |--> else if ->read_iter() exists 
           |       |                                 
           |       +------> call ->read_iter(), e.g.,
           |                +------------------+     
           |                | blkdev_read_iter |     
           |                +------------------+     
           |                                         
           +--> update offset to 'file'              
```

```
+-----------+
| sys_write |
+--|--------+
   |    +------------+
   +--> | ksys_write |
        +--|---------+
           |    +-----------+
           |--> | file_ppos | get file offset
           |    +-----------+
           |    +-----------+
           |--> | vfs_write |
           |    +--|--------+
           |       |
           |       |--> if file has ->write
           |       |
           |       |-------> call ->write()
           |       |               +-----------+
           |       |         e.g., | write_mem |
           |       |               +-----------+
           |       |
           |       +--> else if file has ->write_iter
           |       |
           |       |        +----------------+
           |       +------->| new_sync_write |
           |                +---|------------+
           |                    |    +-----------------+
           |                    +--> | call_write_iter |
           |                         +----|------------+
           |                              |
           |                              +--> call ->write_iter()
           |                                         +-----------------+
           |                                   e.g., | sock_write_iter |
           |                                         +-----------------+
           |
           +--> update file offset                                  
```

```
+-----------+                                                         
| sys_close |                                                         
+--|--------+                                                         
   |    +----------+                                                  
   +--> | close_fd |                                                  
        +--|-------+                                                  
           |    +-----------+                                         
           |--> | pick_file | remove file from table[fd] and return it
           |    +-----------+                                         
           |                                                          
           |--> if ->flush() exists                                   
           |                                                          
           +------> call ->flush()                                    
```

```
+--------------------+                        
| init_special_inode |                        
+----|---------------+                        
     |                                        
     |--> if mode is 'char'                   
     |                                        
     |------> install 'def_chr_fops' to inode 
     |                                        
     |--> else if mode if 'block'             
     |                                        
     +------> install 'def_blk_fops' to inode 
     |                                        
     |--> else if mode if 'fifo'              
     |                                        
     +------> install 'pipefifo_fops' to inode
     |                                        
     |--> else if mode if 'sock'              
     |                                        
     +------> do ntohing                      
     |                                        
     |--> else                                
     |                                        
     +------> error                           
```

## <a name="to-do-list"></a> To-Do List

(TBD)

## <a name="reference"></a> Reference

(TBD)
