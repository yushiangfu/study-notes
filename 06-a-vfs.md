> Study case: Linux version 5.15.0 on AST2500 emulation

## Index

- [Introduction](#introduction)
- [To-Do List](#to-do-list)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

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
+-------------+                                                           
| blkdev_open |                                                           
+---|---------+                                                           
    |    +-------------------+                                            
    |--> | blkdev_get_by_dev |                                            
    |    +----|--------------+                                            
    |         |    +--------------------+                                 
    |         +--> | blkdev_get_no_open | get bdev by dev#                
    |              +--------------------+                                 
    |              +------------------+                                   
    |              | blkdev_get_whole |                                   
    |              +----|-------------+                                   
    |                   |                                                 
    |                   |--> get gendick from bdev                        
    |                   |                                                 
    |                   +--> call gendisk->open(), e.g.,                  
    |                        +---------------+                            
    |                        | blktrans_open | (from gendisk to mtd layer)
    |                        +---------------+                            
    |                                                                     
    +--> assign bdev mapping to file                                      
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
           |       |         call ->write()                             
           |       |               +-----------+                        
           |       |         e.g., | write_mem |                        
           |       |               +-----------+                        
           |       |                                                    
           |       +--> else if file has ->write_iter                   
           |                                                            
           |                +----------------+                          
           |                | new_sync_write |                          
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
