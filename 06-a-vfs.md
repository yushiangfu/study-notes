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
     |         |--> | register_filesystem | register arg (e.g., 'bd_type') to 'file_systems' list
     |         |    +---------------------+
     |         |    +------------+
     |         +--> | kern_mount | prepare 'super_block' and mount nowhere (for internal use)
     |              +--|---------+
     |                 |    +----------------+
     |                 +--> | vfs_kern_mount |
     |                      +----------------+
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
   |       +--> | kernfs_create_root | prepare kernel fs 'root' and its 'node'
   |       |    +--------------------+
   |       |    +---------------------+
   |       +--> | register_filesystem | register arg (e.g., 'sysfs_fs_type') to 'file_systems' list
   |            +---------------------+
   |    +------------+
   |--> | shmem_init |
   |    +--|---------+
   |       |    +-----------------------+
   |       +--> | shmem_init_inodecache | prepare kmem cache for 'shmem_inode_info'
   |            +-----------------------+
   |            +---------------------+
   |            | register_filesystem | register arg (e.g., 'shmem_fs_type') to 'file_systems' list
   |            +---------------------+
   |            +------------+
   |            | kern_mount | prepare 'super_block' and mount nowhere (for internal use)
   |            +--|---------+
   |               |    +----------------+
   |               +--> | vfs_kern_mount |
   |                    +----------------+
   |    +-------------+
   |--> | init_rootfs | determine 'is_tmpfs' (but it's not used elsewhere?)
   |    +-------------+
   |    +-----------------+
   +--> | init_mount_tree |
        +----|------------+
             |    +----------------+
             |--> | vfs_kern_mount | prepare 'super_block' and mount nowhere
             |    +----------------+
             |
             |--> prepare a mount namespace and set the root mount (by what we got from the above)
             |
             |    +------------+
             |--> | set_fs_pwd |
             |    +------------+
             |    +-------------+
             +--> | set_fs_root |
                  +-------------+
```

```
+--------------------+                                                          
| kernfs_create_root | prepare kernel fs 'root' and its 'node'
+----|---------------+                                                          
     |                                                                          
     |--> allocate 'root' (kernfs_root)                                         
     |                                                                          
     |    +-------------------+                                                 
     |--> | __kernfs_new_node | allocate 'node' (kernfs_node)                   
     |    +-------------------+                                                 
     |                                                                          
     |--> set up 'root'                                                         
     |                                                                          
     |    +-----------------+                                                   
     +--> | kernfs_activate | traverse the ndoe(s) and label 'ACTIVATED' on each
          +-----------------+                                                   
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

```
+-------------+                                         
| d_make_root |                                         
+---|---------+                                         
    |    +--------------+                               
    |--> | d_alloc_anon |                               
    |    +---|----------+                               
    |        |    +-----------+                         
    |        +--> | __d_alloc | allocate and init dentry
    |             +-----------+                         
    |    +---------------+                              
    +--> | d_instantiate | fill inode info in dentry    
         +---------------+                              
```

```
+--------------------+
| pseudo_fs_get_tree | allocate 'super_block' and set up (e.g., preapre its inode and dentry)
+----|---------------+
     |    +----------------+
     +--> | get_tree_nodev |
          +---|------------+
              |    +---------------+
              +--> | vfs_get_super |
                   +---|-----------+
                       |    +---------+
                       |--> | sget_fc |
                       |    +--|------+
                       |       |    +-------------+
                       |       +--> | alloc_super |
                       |            +---|---------+
                       |                |
                       |                |--> allocate a super block (sb) and set up it
                       |                |
                       |                |--> call arg set(), e.g.,
                       |                |    +-------------------+
                       |                |    | set_anon_super_fc | assign an free dev_t to the sb
                       |                |    +-------------------+
                       |                |
                       |                +--> add the sb to 'super_blocks' list
                       |
                       +--> if sb doesn't have root yet

                                call arg fill_super(), e.g.,
                                +----------------------+
                                | pseudo_fs_fill_super | prepare indoe and dentry for sb
                                +----------------------+
```

```
+------------+                                                                                                                 
| kern_mount | prepare 'super_block' and mount nowhere (for internal use)                                                      
+--|---------+                                                                                                                 
   |    +----------------+                                                                                                     
   +--> | vfs_kern_mount |                                                                                                     
        +---|------------+                                                                                                     
            |    +----------------------+                                                                                      
            |--> | fs_context_for_mount |                                                                                      
            |    +-----|----------------+                                                                                      
            |          |    +------------------+                                                                               
            |          +--> | alloc_fs_context |                                                                               
            |               +----|-------------+                                                                               
            |                    |                                                                                             
            |                    |--> allcoate fs context (fc)                                                                 
            |                    |                                                                                             
            |                    +--> call ->init_fs_context(), e.g.,                                                          
            |                         +--------------------+                                                                   
            |                         | bd_init_fs_context | prepare pseudo fs context and install 'bdev_sops'                 
            |                         +--------------------+                                                                   
            |    +----------+                                                                                                  
            +--> | fc_mount |                                                                                                  
                 +--|-------+                                                                                                  
                    |    +--------------+                                                                                      
                    |--> | vfs_get_tree |                                                                                      
                    |    +---|----------+                                                                                      
                    |        |                                                                                                 
                    |        +--> call ->get_tree(), e.g.,                                                                     
                    |             +--------------------+                                                                       
                    |             | pseudo_fs_get_tree | allocate 'super_block' and set up (e.g., preapre its inode and dentry)
                    |             +--------------------+                                                                       
                    |    +------------------+                                                                                  
                    +--> | vfs_create_mount | allocate and set up 'mount'                                                      
                         +------------------+                                                                                  
```

```
+-----------------+                                                                 
| populate_rootfs |                                                                 
+----|------------+                                                                 
     |    +-----------------------+                                                 
     +--> | async_schedule_domain | (do_populate_rootfs)                            
          +-----|-----------------+                                                 
                |    +----------------------------+                                 
                +--> | async_schedule_node_domain |                                 
                     +------|---------------------+                                 
                            |    +-----------+                                      
                            |--> | INIT_WORK | (async_run_entry_fn)                 
                            |    +-----------+                                      
                            |                                                       
                            |--> add entry to domain                                
                            |                                                       
                            |    +-----------------+                                
                            +--> | queue_work_node | add work to 'system_unbound_wq'
                                 +-----------------+                                
```

```
+-----------+                                                                                       
| filp_open |                                                                                       
+--|--------+                                                                                       
   |    +----------------+                                                                          
   |--> | getname_kernel | prepare 'filename' and copy name string to its field                     
   |    +----------------+                                                                          
   |    +----------------+                                                                          
   +--> | file_open_name |                                                                          
        +---|------------+                                                                          
            |    +--------------+                                                                   
            +--> | do_filp_open | allocate 'file', find dentry/inode, install ops, and call ->open()
                 +--------------+                                                                   
```

```
+-----------+                                                 
| path_init | set nd's root, path, and inode
+--|--------+                                                 
   |                                                          
   |--> if it's a absolute path (starts with a '/')           
   |                                                          
   |        +--------------+                                  
   |------> | nd_jump_root |                                  
   |        +---|----------+                                  
   |            |    +----------+                             
   |            |--> | set_root | nd->root = current->fs->root
   |            |    +----------+                             
   |            |                                             
   |            +--> nd->path = nd->root                      
   |                                                          
   |--> else (a relative path)                                
   |                                                          
   |------> nd->path = current->fs->pwd                       
   |                                                          
   +------> set nd->inode from nd->path                       
```

```                                                     
 00000000: 3037 3037 3031 3030 3030 3032 4431 3030  070701000002D100
           |------------| |-----------------| |---                  
             cpio magic           ino                               
 00000010: 3030 3431 4544 3030 3030 3030 3030 3030  0041ED0000000000
           -------------| |-----------------| |---                  
                mode              uid                               
 00000020: 3030 3030 3030 3030 3030 3030 3032 3632  0000000000000262
           -------------| |-----------------| |---                  
                gid              nlink                              
 00000030: 3438 3937 3231 3030 3030 3030 3030 3030  4897210000000000
           -------------| |-----------------| |---                  
               mtime           body len                             
 00000040: 3030 3030 3033 3030 3030 3030 3031 3030  0000030000000100
           -------------| |-----------------| |---                  
               major             minor                              
 00000050: 3030 3030 3030 3030 3030 3030 3030 3030  0000000000000000
           -------------| |-----------------| |---                  
              for rdev          for rdev                            
 00000060: 3030 3030 3034 3030 3030 3030 3030 6465  00000400000000de
           -------------|                     |---                  
              name len                                              
 00000070: 7600 0000 3037 3037 3031 3030 3030 3032  v...070701000002
           ---|      |------------| |-------------                  
           name        cpio magic         ino                       
 00000080: 4432 3030 3030 3231 3830 3030 3030 3030  D200002180000000
           ---| |-----------------| |-------------                  
                        mode              uid                       
 00000090: 3030 3030 3030 3030 3030 3030 3030 3030  0000000000000000
           ---| |-----------------| |-------------                  
                        gid              nlink                      
 000000a0: 3031 3632 3438 3937 3231 3030 3030 3030  0162489721000000
           ---| |-----------------| |-------------                  
                       mtime            body len                    
 000000b0: 3030 3030 3030 3030 3033 3030 3030 3030  0000000003000000
           ---| |-----------------| |-------------                  
                       major             minor                      
 000000c0: 3031 3030 3030 3030 3035 3030 3030 3030  0100000005000000
           ---| |-----------------| |-------------                  
                      for rder          for rder                    
 000000d0: 3031 3030 3030 3030 3043 3030 3030 3030  010000000C000000
           ---| |-----------------|                                 
                      name len                                      
 000000e0: 3030 6465 762f 636f 6e73 6f6c 6500 0000  00dev/console...
                |---------------------------|                       
                            name                                 
 (skip the remaining entries ...)
```

```
dir /dev 0755 0 0
nod /dev/console 0600 0 0 c 5 1
dir /root 0700 0 0
```

```
+---------------+                                                                                        
| path_parentat | walk through the path name, update dentry and mnt of the last component in nd
+---|-----------+                                                                                        
    |    +-----------+                                                                                   
    |--> | path_init | set nd's root, path, and inode                                                    
    |    +-----------+                                                                                   
    |    +----------------+                                                                              
    |--> | link_path_walk | walk through the path name, update dentry and mnt of the last component in nd
    |    +----------------+                                                                              
    |    +---------------+                                                                               
    |--> | complete_walk | finalize something that doesn't matter to us                                  
    |    +---------------+                                                                               
    |                                                                                                    
    |--> copy path to argument and reset nd                                                              
    |                                                                                                    
    |    +----------------+                                                                              
    +--> | terminate_walk | finalize something that doesn't matter to us                                 
         +----------------+                                                                              
```

```
+----------------+                                                                          
| link_path_walk | walk through the path name, update dentry and mnt of the last component in nd
+---|------------+                                                                          
    |                                                                                       
    |--> endless loop                                                                       
    |                                                                                       
    +------> update nd last                                                                 
    |                                                                                       
    |        +----------------+                                                             
    |------> | walk_component | update path (dentry + mnt) and inode of nd to next component
    |        +----------------+                                                             
    |                                                                                       
    +------> (ignore link handling)                                                         
```

```
+----------------+                                                                          
| walk_component | update path (dentry + mnt) and inode of nd to next component           
+---|------------+                                                                          
    |                                                                                       
    |--> if it's DOT or DOTDOT                                                              
    |                                                                                       
    |        +-------------+                                                                
    |------> | handle_dots | update path and inode of nd if it's DOTDOT (do nothing for DOT)
    |        +-------------+                                                                
    |                                                                                       
    |------> return                                                                         
    |                                                                                       
    |    +-------------+                                                                    
    |--> | lookup_fast | find child dentry in hash table by parent dentry and child name    
    |    +-------------+                                                                    
    |    +-----------+                                                                      
    +--> | step_into | update path and inode of nd (automount and link cases are considered)
         +-----------+                                                                      
```

```
+-------------+                                                                             
| handle_dots | update path and inode of nd if it's DOTDOT (do nothing for DOT)             
+---|---------+                                                                             
    |                                                                                       
    |--> if it's DOTDOT (do nothing for DOT case)                                           
    |                                                                                       
    |------> ensure nd has root setting                                                     
    |                                                                                       
    |    +---------------+                                                                  
    |--> | follow_dotdot |                                                                  
    |    +---|-----------+                                                                  
    |        |                                                                              
    |        |--> if it's the root, return NULL                                             
    |        |                                                                              
    |        |--> if it's a mount point                                                     
    |        |                                                                              
    |        |        +-------------------+                                                 
    |        |------> | choose_mountpoint | update dentry and mnt in path with mount point  
    |        |        +-------------------+                                                 
    |        |                                                                              
    |        +--> get parent dentry and inode                                               
    |                                                                                       
    |    +-----------+                                                                      
    +--> | step_into | update path and inode of nd (automount and link cases are considered)
         +-----------+                                                                      
```

```
+-----------+                                                                     
| step_into | update path and inode of nd (automount and link cases are considered)
+--|--------+                                                                     
   |    +---------------+                                                         
   |--> | handle_mounts | update mnt and dentry of path if it's mounted           
   |    +---|-----------+                                                         
   |        |    +-----------------+                                              
   |        +--> | traverse_mounts |                                              
   |             +----|------------+                                              
   |                  |    +-------------------+                                  
   |                  +--> | __traverse_mounts |                                  
   |                       +----|--------------+                                  
   |                            |                                                 
   |                            |--> if it's mounted                              
   |                            |                                                 
   |                            |        +------------+                           
   |                            |------> | lookup_mnt | return the first child mnt
   |                            |        +------------+                           
   |                            |                                                 
   |                            |------> update the mnt and dentry of path with it
   |                            |                                                 
   |                            +--> (ignore the case of auto mount)              
   |                                                                              
   |--> update path and inode of nd                                               
   |                                                                              
   +--> (ignore the case of link)                                                 
```

```
+-----------------+                                                                                                              
| vfs_path_lookup |                                                                                                              
+----|------------+                                                                                                              
     |    +-----------------+                                                                                                    
     +--> | filename_lookup |                                                                                                    
          +----|------------+                                                                                                    
               |    +---------------+                                                                                            
               |--> | set_nameidata | set up nd and replace the one of current task                                              
               |    +---------------+                                                                                            
               |    +---------------+                                                                                            
               |--> | path_lookupat |                                                                                            
               |    +---|-----------+                                                                                            
               |        |    +-----------+                                                                                       
               |        |--> | path_init | decide the starting point for lookup                                                  
               |        |    +-----------+                                                                                       
               |        |                                                                                                        
               |        |--> endless loop                                                                                        
               |        |                                                                                                        
               |        |        +----------------+                                                                              
               |        |------> | link_path_walk | walk through the path name, update dentry and mnt of the last component in nd
               |        |        +----------------+                                                                              
               |        |                                                                                                        
               |        |------> break loop if error                                                                             
               |        |                                                                                                        
               |        |        +-------------+                                                                                 
               |        |------> | lookup_last | update path (dentry + mnt) and inode of nd to next component                    
               |        |        +-------------+                                                                                 
               |        |                                                                                                        
               |        |------> break loop if error                                                                             
               |        |                                                                                                        
               |        +--> copy path to argument and reset nd                                                                  
               |                                                                                                                 
               |    +-------------------+                                                                                        
               +--> | restore_nameidata |                                                                                        
                    +-------------------+                                                                                        
```

```
+------------+                                                                         
| vfs_create | prepare an inode of regular file and link it with given dentry    
+--|---------+                                                                         
   |                                                                                   
   |--> if dir inode has no ->create(), return error                                   
   |                                                                                   
   +--> call ->create(), e.g.,                                                         
        +--------------+                                                               
        | shmem_create | prepare an inode of regular file and link it with given dentry
        +--------------+                                                               
```

```
+-----------+                                                                                           
| new_inode | allocate a complex or regular inode, init and add to sb's list                            
+--|--------+                                                                                           
   |    +------------------+                                                                            
   |--> | new_inode_pseudo |                                                                            
   |    +----|-------------+                                                                            
   |         |    +-------------+                                                                       
   |         +--> | alloc_inode |                                                                       
   |              +---|---------+                                                                       
   |                  |                                                                                 
   |                  |--> if sb has ->alloc_inode()                                                    
   |                  |                                                                                 
   |                  |------> call ->alloc_inode(), e.g.,                                              
   |                  |        +-------------------+                                                    
   |                  |        | shmem_alloc_inode | allocate a complex inode containing the regular one
   |                  |        +-------------------+                                                    
   |                  |                                                                                 
   |                  |--> else                                                                         
   |                  |                                                                                 
   |                  |        +------------------+                                                     
   |                  |------> | kmem_cache_alloc | allocate a regular one                              
   |                  |        +------------------+                                                     
   |                  |    +-------------------+                                                        
   |                  +--> | inode_init_always | init inode, e.g., set its sb                           
   |                       +-------------------+                                                        
   |    +-------------------+                                                                           
   +--> | inode_sb_list_add | add the inode to sb's list                                                
        +-------------------+                                                                           
```

```
+-------------+                                                                                    
| vfs_tmpfile | prepare dentry and inode, set dentry's name using ino, and link inode and dentry   
+---|---------+                                                                                    
    |                                                                                              
    |--> if dir inode doesn't have ->tmpfile(), return error                                       
    |                                                                                              
    |    +---------+                                                                               
    |--> | d_alloc | allocate and init a dentry, link with its parent                              
    |    +---------+                                                                               
    |                                                                                              
    +--> call ->tmpfile(), e.g.,                                                                   
         +---------------+                                                                         
         | shmem_tmpfile | prepare an inode, set dentry's name using ino, and link inode and dentry
         +---------------+                                                                         
```

```
+-----------+                                                                           
| vfs_mknod |                                                                           
+--|--------+                                                                           
   |                                                                                    
   |--> if dir inode has no ->mknod(), return error                                     
   |                                                                                    
   +--> call ->mknod(), e.g.,                                                           
        +-------------+                                                                 
        | shmem_mknod | prepare an inode of specified type and link it with given dentry
        +-------------+                                                                 
```

```
+-----------+                                                                      
| vfs_mkdir | prepare an inode of directory and link it with given dentry          
+--|--------+                                                                      
   |                                                                               
   |--> if dir inode has no ->mkdir(), return error                                
   |                                                                               
   +--> call ->mkdir(), e.g.,                                                      
        +-------------+                                                            
        | shmem_mkdir | prepare an inode of directory and link it with given dentry
        +-------------+                                                            
```

```
+-----------+                                                            
| vfs_rmdir |                                                            
+--|--------+                                                            
   |                                                                     
   |--> if dir has no ->rmdir(), return error                            
   |                                                                     
   |--> if it's a mount point, return error                              
   |                                                                     
   |--> call ->rmdir(), e.g.,                                            
   |    +-------------+                                                  
   |    | shmem_rmdir | inode nlink-- -- --, dentry ref count--          
   |    +-------------+                                                  
   |    +----------------------+                                         
   |--> | shrink_dcache_parent | release unused child dentries           
   |    +----------------------+                                         
   |    +---------------+                                                
   +--> | detach_mounts | detach the mount if the dentry is a mount point
        +---------------+                                                
```

```
+------------+                                                           
| vfs_unlink |                                                           
+--|---------+                                                           
   |                                                                     
   |--> if dir has no ->unlink(), return error                           
   |                                                                     
   |--> if dentry is a mount point, return error                         
   |                                                                     
   |--> call ->unlink(), e.g.,                                           
   |    +--------------+                                                 
   |    | shmem_unlink | inode nlink--, dentry ref count--               
   |    +--------------+                                                 
   |    +---------------+                                                
   +--> | detach_mounts | detach the mount if the dentry is a mount point
        +---------------+                                                
```

```
+-------------+                                                                    
| vfs_symlink |                                                                    
+---|---------+                                                                    
    |                                                                              
    |--> if dir has no ->symlink(), return error                                   
    |                                                                              
    +--> call ->symlink(), e.g.,                                                   
         +---------------+                                                         
         | shmem_symlink | preapre inode, save dst path to it, and link with dentry
         +---------------+                                                         
```

```
+----------+                                                          
| vfs_link | fill inode info in the new dentry and link them          
+--|-------+                                                          
   |                                                                  
   +--> call ->link(), e.g.,                                          
        +------------+                                                
        | shmem_link | fill inode info in the new dentry and link them
        +------------+                                                
```

## <a name="reference"></a> Reference

(TBD)
