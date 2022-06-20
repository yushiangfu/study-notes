> Study case: Linux version 5.15.0 on AST2500 emulation

## Index

- [Introduction](#introduction)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

```
+----------+                                                                                       
| sys_sync |                                                                                       
+--|-------+                                                                                       
   |    +-----------+                                                                              
   +--> | ksys_sync |                                                                              
        +--|--------+                                                                              
           |    +------------------------+                                                         
           +--> | wakeup_flusher_threads | flush task plug list, wake up flusher for each bdi      
                +-----|------------------+                                                         
                      |                                                                            
                      |--> if task plug list isn't empty                                           
                      |                                                                            
                      |        +-------------------------+                                         
                      |------> | blk_schedule_flush_plug | flush both cb and mq list in task plug  
                      |        +-------------------------+                                         
                      |                                                                            
                      |--> for each bdi on list                                                    
                      |                                                                            
                      |        +------------------------------+                                    
                      +------> | __wakeup_flusher_threads_bdi |                                    
                               +-------|----------------------+                                    
                                       |                                                           
                                       |--> return if bdi has no dirty io                          
                                       |                                                           
                                       |--> for each wb on bdi                                     
                                       |                                                           
                                       |        +--------------------+                             
                                       +------> | wb_start_writeback | steal a work and add to pool
                                                +--------------------+                             
```

```
+-------------------------+                                                                                 
| blk_schedule_flush_plug | : flush both cb and mq list in task plug                                        
+------|------------------+                                                                                 
       |                                                                                                    
       |--> if task has a plug                                                                              
       |                                                                                                    
       |        +---------------------+                                                                     
       +------> | blk_flush_plug_list |                                                                     
                +-----|---------------+                                                                     
                      |    +----------------------+                                                         
                      |--> | flush_plug_callbacks | excute each cb node in plug list                        
                      |    +----------------------+                                                         
                      |                                                                                     
                      |--> if plug->mq_list isn't empty                                                     
                      |                                                                                     
                      |        +------------------------+                                                   
                      +------> | blk_mq_flush_plug_list | deliver each rq in plug list to elevator or driver
                               +------------------------+                                                   
```

```
+-----------+
| wb_workfn | : rename worker, and wake it up to handle the work
+--|--------+
   |
   |--> set worker name = "flush-%s"
   |
   |    +-----------------+
   |--> | wb_do_writeback |
   |    +-----------------+
   |
   |--> if there's work on the wb
   |
   |        +-----------+
   +------> | wb_wakeup | steal a work and queue it
   |        +-----------+
   |
   |--> else if wb has dirty io ?
   |
   |        +--------------------+
   +------> | wb_wakeup_delayed  | steal a work and add to a timer
            +--------------------+                              
```

```
+-----------------+                                                                                       
| wb_do_writeback | perform all kinds of writeback                                                        
+----|------------+                                                                                       
     |                                                                                                    
     |--> while there's work in wb                                                                        
     |                                                                                                    
     |        +--------------+                                                                            
     |------> | wb_writeback | plug, writeback inodes on b_io list and wait till 'sync' is cleared, unplug
     |        +--------------+                                                                            
     |    +--------------------+                                                                          
     |--> | wb_check_start_all | writeback if 'start_all'                                                 
     |    +--------------------+                                                                          
     |    +-------------------------+                                                                     
     |--> | wb_check_old_data_flush | writeback if dirty_writeback_interval is set (yes, 500)             
     |    +-------------------------+                                                                     
     |    +---------------------------+                                                                   
     |--> | wb_check_background_flush | writeback if thresh is exceeded                                   
     |    +---------------------------+                                                                   
     |                                                                                                    
     +--> clear 'writeback_running'                                                                       vv
```

```
+--------------+                                                                                
| wb_writeback | : plug, writeback inodes on b_io list and wait till 'sync' is cleared, unplug  
+---|----------+                                                                                
    |    +----------------+                                                                     
    |--> | blk_start_plug |                                                                     
    |    +----------------+                                                                     
    |                                                                                           
    |--> endless loop                                                                           
    |                                                                                           
    |------> break if we finish the work                                                        
    |                                                                                           
    |------> break if we are in the background and below the dirty threshold                    
    |                                                                                           
    |------> if wb b_io list is empty                                                           
    |                                                                                           
    |            +----------+                                                                   
    |----------> | queue_io | move inodes from b_more_io, b_dirty, and b_dirty_time to b_io list
    |            +----------+                                                                   
    |                                                                                           
    |------> if work has sb specified                                                           
    |                                                                                           
    |            +---------------------+                                                        
    |----------> | writeback_sb_inodes | writeback part of inode@b_io that belongs to sb        
    |            +---------------------+                                                        
    |                                                                                           
    |------> else                                                                               
    |                                                                                           
    |            +-----------------------+                                                      
    |----------> | __writeback_inodes_wb | writeback part of inode@b_io                         
    |            +-----------------------+                                                      
    |                                                                                           
    |------> continu if we have progress                                                        
    |                                                                                           
    |------> break if no inode on b_more_io                                                     
    |                                                                                           
    |        +--------------------------+                                                       
    |------> | inode_sleep_on_writeback | wait till 'sync' is cleared                           
    |        +--------------------------+                                                       
    |    +-----------------+                                                                    
    +--> | blk_finish_plug |                                                                    
         +-----------------+                                                                    
```

```
+----------+                                                                           
| queue_io | : move inodes from b_more_io, b_dirty, and b_dirty_time to b_io list      
+--|-------+                                                                           
   |                                                                                   
   |--> splice list from b_more_io to b_io                                             
   |                                                                                   
   |    +---------------------+                                                        
   |--> | move_expired_inodes | move expired inodes from b_dirty list to arg list      
   |    +---------------------+                                                        
   |    +---------------------+                                                        
   +--> | move_expired_inodes | move expired inodes from b_dirty_time list to arg list 
        +---------------------+                                                        
```

```
+---------------------+                                                                               
| move_expired_inodes | : move expired inodes to arg list                                             
+-----|---------------+                                                                               
      |                                                                                               
      |--> while list has something              the youngest                               the oldest
      |                                                                                               
      |------> get the last inode from list         +-----+       +-----+       +-----+       +-----+ 
      |                                             |inode| <---> |inode| <---> |inode| <---> |inode| 
      |------> break if inode is old enough         +-----+       +-----+       +-----+       +-----+ 
      |                                                                                               
      |        +-----------+                                                                          
      |------> | list_move | move younger inode to a tmp list                                         
      |        +-----------+                                                                          
      |                                                                                               
      +--> splic list from tmp to arg dispatch_queue, sort first if necessary                         
```

```
+---------------------+                                                                  
| writeback_sb_inodes | : writeback part of inode@b_io that belongs to sb                
+-----|---------------+                                                                  
      |                                                                                  
      |--> set up wbc (writeback control)                                                
      |                                                                                  
      |--> while b_io list has something                                                 
      |                                                                                  
      |------> break if the inode belongs to a different sb                              
      |                                                                                  
      |------> if inode isn't suitable for writeback                                     
      |                                                                                  
      |            +------------+                                                        
      |----------> | requeue_io | move inode to b_more_io list                           
      |            +------------+                                                        
      |                                                                                  
      |----------> continue                                                              
      |                                                                                  
      |------> if inode is in 'sync' already                                             
      |                                                                                  
      |            +--------------------------+                                          
      |----------> | inode_sleep_on_writeback | wait for its finish                      
      |            +--------------------------+                                          
      |                                                                                  
      |----------> continue                                                              
      |                                                                                  
      |------> label inode 'sync'                                                        
      |                                                                                  
      |        +--------------------------+                                              
      |------> | __writeback_single_inode | writeback the given inode and its dirty pages
      |        +--------------------------+                                              
      |        +---------------+                                                         
      |------> | requeue_inode | (skip)                                                  
      |        +---------------+                                                         
      |        +---------------------+                                                   
      |------> | inode_sync_complete | wait till 'sync' is cleared                       
      |        +---------------------+                                                   
      |                                                                                  
      +------>  break if we spent too much time already                                  
```

```
+--------------------------+
| __writeback_single_inode | : writeback the given inode and its dirty pages
+------|-------------------+
       |    +---------------+
       |--> | do_writepages | with specified range, write dirty pages back
       |    +---------------+
       |
       |--> if mode is 'sync_all'
       |
       |        +-------------------+
       |------> | filemap_fdatawait | wait till WRITEBACK pages finish
       |        +-------------------+
       |
       |--> mark inode 'dirty_sync' if it expires
       |
       |--> clear inode 'dirty'
       |
       |--> label inode 'dirty_pages' if mapping has 'dirty' tag
       |
       |--> if there are dirty bits other than 'dirty_pages'
       |
       |        +-------------+
       +------> | write_inode | call ->write_inode() if it exists
                +-------------+
```

```
+-----------------------+                                                             
| __writeback_inodes_wb | ： writeback part of inode@b_io                
+-----|-----------------+                                                             
      |                                                                               
      |--> while b_io list has something                                              
      |                                                                               
      |        +---------------------+                                                
      |------> | writeback_sb_inodes | writeback part of inode@b_io that belongs to sb
      |        +---------------------+                                                
      |                                                                               
      +------> break if we spent too much time already                                
```

```
     +-----------+                                               
+----| sys_fsync |                                               
|    +-----------+                                               
|    +---------------+                                           
+----| sys_fdatasync |                                           
|    +---------------+                                           
|    +----------+                                                
+--> | do_fsync |                                                
     +--|-------+                                                
        |    +-----------+                                       
        +--> | vfs_fsync |                                       
             +--|--------+                                       
                |    +-----------------+                         
                +--> | vfs_fsync_range |                         
                     +----|------------+                         
                          |                                      
                          +--> call ->fsync() if it exists, e.g.,
                               +----------------+                
                               | ext4_sync_file |                
                               +----------------+                
```

```
+-----------+                                                      
| sys_msync | for vma within range, call ->fsync()                 
+--|--------+                                                      
   |                                                               
   |--> for vma within range                                       
   |                                                               
   |------> if user specifies 'sync' and it's a shared file mapping
   |                                                               
   |            +-----------------+                                
   +----------> | vfs_fsync_range | call ->fsync()                 
                +-----------------+                                
```

```
+---------------------+                                                                                                            
| writeback_inodes_sb | : writeback dirty inodes of given sb                                                                         
+-----|---------------+                                                                                                            
      |    +------------------------+                                                                                              
      +--> | writeback_inodes_sb_nr |                                                                                              
           +-----|------------------+                                                                                              
                 |    +--------------------------+                                                                                 
                 +--> | __writeback_inodes_sb_nr |                                                                                 
                      +------|-------------------+                                                                                 
                             |    +-----------------------+                                                                        
                             |--> | bdi_split_work_to_wbs |                                                                        
                             |    +-----|-----------------+                                                                        
                             |          |    +---------------+                                                                     
                             |          +--> | wb_queue_work |                                                                     
                             |               +---|-----------+                                                                     
                             |                   |                                                                                 
                             |                   |--> if the bdi wb is labeled 'registered'                                        
                             |                   |                                                                                 
                             |                   |        +---------------+                                                        
                             |                   |------> | list_add_tail | queue work to bdi wb                                   
                             |                   |        +---------------+                                                        
                             |                   |        +------------------+                                                     
                             |                   |------> | mod_delayed_work | steal a work, and either add to a pool or to a timer
                             |                   |        +------------------+                                                     
                             |                   |                                                                                 
                             |                   |--> else                                                                         
                             |                   |                                                                                 
                             |                   |        +-----------------------+                                                
                             |                   +------> | finish_writeback_work | wake up all the tasks waiting on this          
                             |                            +-----------------------+                                                
                             |    +------------------------+                                                                       
                             +--> | wb_wait_for_completion | wait for the completion of bdi writeback works                        
                                  +------------------------+                                                                       
```

## <a name="reference"></a> Reference

(TBD)
