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
     +--> clear 'writeback_running' 
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
                             |          +--> | wb_queue_work | : queue work to pool
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

```
+---------------+                                                                                        
| filemap_flush | : writeback dirty pages in mapping                                                       
+---|-----------+                                                                                        
    |    +----------------------+                                                                        
    +--> | __filemap_fdatawrite |                                                                        
         +-----|----------------+                                                                        
               |    +----------------------------+                                                       
               +--> | __filemap_fdatawrite_range |                                                       
                    +------|---------------------+                                                       
                           |                                                                             
                           |--> set up writeback control                                                 
                           |                                                                             
                           |    +------------------------+                                               
                           +--> | filemap_fdatawrite_wbc |                                               
                                +-----|------------------+                                               
                                      |    +---------------+                                             
                                      +--> | do_writepages | with specified range, write dirty pages back
                                           +---------------+                                             
```

```
+------------+                                                                        
| sys_syncfs | : writeback dirty pages and wait till 'writeback' cleared              
+--|---------+                                                                        
   |                                                                                  
   |--> get sb from file                                                              
   |                                                                                  
   |    +-----------------+                                                           
   +--> | sync_filesystem | given sb, writeback dirty inodes and wait till it finishes
        +-----------------+                                                           
```

```
+-----------------+                                                                    
| sync_filesystem | : given sb, writeback dirty inodes and wait till it finishes       
+----|------------+                                                                    
     |                                                                                 
     |--> return if it's read only                                                     
     |                                                                                 
     |    +---------------------+                                                      
     |--> | writeback_inodes_sb | writeback dirty inodes of given sb                   
     |    +---------------------+                                                      
     |                                                                                 
     |--> if ->sync_fs() exists                                                        
     |                                                                                 
     +------> call ->sync_fs(), e.g.,                                                  
     |        +--------------+                                                         
     |        | ext4_sync_fs | (arg = no wait)                                         
     |        +--------------+                                                         
     |    +----------------------+                                                     
     |--> | sync_blockdev_nowait | writeback dirty pages in mapping                    
     |    +----------------------+                                                     
     |    +----------------+                                                           
     |--> | sync_inodes_sb | given sb, writeback dirty inodes and wait till it finishes
     |    +----------------+                                                           
     |                                                                                 
     |--> if ->sync_fs() exists                                                        
     |                                                                                 
     |------> call ->sync_fs(), e.g.,                                                  
     |        +--------------+                                                         
     |        | ext4_sync_fs | (arg = wait)                                            
     |        +--------------+                                                         
     |    +---------------+                                                            
     +--> | sync_blockdev | writeback dirty pages and wait till 'writeback' cleared    
          +---------------+                                                            
```

```
+----------------+                                                                                             
| sync_inodes_sb | : given sb, writeback dirty inodes and wait till it finishes                                
+---|------------+                                                                                             
    |                                                                                                          
    |--> set up wb work                                                                                        
    |                                                                                                          
    |--> return if bdi is noop_backing_dev_info                                                                
    |                                                                                                          
    |   +-----------------------+                                                                              
    |-->| bdi_split_work_to_wbs | queue work to pool                                                           
    |   +-----------------------+                                                                              
    |   +------------------------+                                                                             
    |-->| wb_wait_for_completion |                                                                             
    |   +------------------------+                                                                             
    |   +----------------+                                                                                     
    +-->| wait_sb_inodes | for each inode on wb list, weait till each page with 'writeback' tag loses the label
        +----------------+                                                                                     
```

```
+----------------+                                                                                              
| wait_sb_inodes | : for each inode on wb list, weait till each page with 'writeback' tag loses the label       
+---|------------+                                                                                              
    |                                                                                                           
    |--> move the writeback inode list from sb to local                                                         
    |                                                                                                           
    |--> while the list has something                                                                           
    |                                                                                                           
    |------> take the first inode on list                                                                       
    |                                                                                                           
    |------> move inode back to sb                                                                              
    |                                                                                                           
    |------> continue if the inode mapping has no 'writeback' tag                                               
    |                                                                                                           
    |        +-------------------------------+                                                                  
    +------> | filemap_fdatawait_keep_errors | for each page with 'writeback' tag, wait till their label cleared
             +-------------------------------+                                                                  
```

```
+---------------------------+                                                                          
| __filemap_fdatawait_range | : for each page with 'writeback' tag, wait till their label cleared        
+------|--------------------+                                                                          
       |                                                                                               
       |--> while index <= end                                                                         
       |                                                                                               
       |        +--------------------------+                                                           
       |------> | pagevec_lookup_range_tag | find page with specified tag from mapping and fill pagevec
       |        +--------------------------+                                                           
       |                                                                                               
       |------> for each page in pagevec                                                               
       |                                                                                               
       |            +------------------------+                                                         
       +----------> | wait_on_page_writeback | wait till the page 'writeback' cleared                  
                    +------------------------+                                                         
```

```
+---------------------+                                                                                                       
| sys_sync_file_range | writeback the file, wait if it's specified                                                            
+-----|---------------+                                                                                                       
      |    +----------------------+                                                                                           
      +--> | ksys_sync_file_range |                                                                                           
           +-----|----------------+                                                                                           
                 |    +-----------------+                                                                                     
                 +--> | sync_file_range |                                                                                     
                      +----|------------+                                                                                     
                           |                                                                                                  
                           |--> if flag specifies 'wait_before'                                                               
                           |                                                                                                  
                           |        +----------------------+                                                                  
                           |------> | file_fdatawait_range | for each page with 'writeback' tag, wait till their label cleared
                           |        +----------------------+                                                                  
                           |                                                                                                  
                           |--> if flag specifies 'write'                                                                     
                           |                                                                                                  
                           |        +----------------------------+                                                            
                           |------> | __filemap_fdatawrite_range | writeback dirty pages in mapping                           
                           |        +----------------------------+                                                            
                           |                                                                                                  
                           |--> if flag specifes 'wait_after'                                                                 
                           |                                                                                                  
                           |        +----------------------+                                                                  
                           +------> | file_fdatawait_range | for each page with 'writeback' tag, wait till their label cleared
                                    +----------------------+                                                                  
```

```
+----------------------+                                                               
| sync_mapping_buffers | submit all the dirty bh and wait for them to finish           
+-----|----------------+                                                               
      |    +--------------------+                                                      
      +--> | fsync_buffers_list |                                                      
           +----|---------------+                                                      
                |    +----------------+                                                
                |--> | blk_start_plug |                                                
                |    +----------------+                                                
                |                                                                      
                |--> while arg list has something                                      
                |                                                                      
                |------> get bh from list                                              
                |                                                                      
                |        +----------------------+                                      
                |------> | __remove_assoc_queue | remove bh from associated mapping    
                |        +----------------------+                                      
                |                                                                      
                |------> if bh is dirty or locked                                      
                |                                                                      
                |            +----------+                                              
                |----------> | list_add | add bh to the local list                     
                |            +----------+                                              
                |            +--------------------+                                    
                |----------> | write_dirty_buffer | clear 'dirty' and submit bh        
                |            +--------------------+                                    
                |    +-----------------+                                               
                |--> | blk_finish_plug |                                               
                |    +-----------------+                                               
                |                                                                      
                |--> while tmp list has something                                      
                |                                                                      
                |------> reelase bh or add it back to mapping if it's dirty            
                |                                                                      
                |    +--------------------+                                            
                +--> | osync_buffers_list | wait for the already-submitted bh to finish
                     +--------------------+                                            
```

```
+---------------+                                 
| page_anon_vma | : get anon_vma of the given page
+---|-----------+                                 
    |    +---------------+                        
    |--> | compound_head | get head page          
    |    +---------------+                        
    |    +-----------------+                      
    +--> | __page_rmapping | get page rmapping    
         +-----------------+                      
```

```
+--------------+                                                                       
| try_to_unmap | : unmap the given page in each mapping                                
+---|----------+                                                                       
    |                                                                                  
    |--> set up rwc                                                                    
    |                                                                                  
    |    +-----------+                                                                 
    +--> | rmap_walk | for each vma related to the give page, clear pte in each mapping
         +-----------+                                                                 
```

```
+------------------+                                                                           
| shrink_page_list | : try to reclaim the given page list                                      
+----|-------------+                                                                           
     |                                                                                         
     |--> while page_list has something                                                        
     |                                                                                         
     |------> remove the last page from list                                                   
     |                                                                                         
     |------> if page is mapped                                                                
     |                                                                                         
     |            +--------------+                                                             
     |----------> | try_to_unmap | unmap the given page in each mapping                        
     |            +--------------+                                                             
     |                                                                                         
     |------> if page is dirty                                                                 
     |                                                                                         
     |            +---------+                                                                  
     |----------> | pageout | call ->writepage(), return status (e.g., 'success', 'clean', ...)
     |            +---------+                                                                  
     |                                                                                         
     |------> if page has buffer                                                               
     |                                                                                         
     |            +---------------------+                                                      
     |----------> | try_to_release_page | detatch bh list from page and free them              
     |            +---------------------+                                                      
     |                                                                                         
     |--> handle pages on demote list (what is demotion?)                                      
     |                                                                                         
     |--> handle pages on free list (what is demotion?)                                        
     |                                                                                         
     +--> handle pages on ret list (what is demotion?)                                         
```

```
+---------+                                                                        
| pageout | : call ->writepage(), return status (e.g., 'success', 'clean', ...)    
+--|------+                                                                        
   |                                                                               
   |--> return 'keep' if the page isn't freeable                                   
   |                                                                               
   |--> if mapping isn't provided                                                  
   |                                                                               
   |------> if page has private                                                    
   |                                                                               
   |            +---------------------+                                            
   |----------> | try_to_free_buffers | detatch bh list from page and free those bh
   |            +---------------------+                                            
   |                                                                               
   |----------> return 'clean' if buffer is dropped                                
   |                                                                               
   |------> return 'keep'                                                          
   |                                                                               
   |--> if mapping has no ->writepage()                                            
   |                                                                               
   |------> return 'activate'                                                      
   |                                                                               
   |--> if page is dirty                                                           
   |                                                                               
   |------> set up wbc                                                             
   |                                                                               
   |------> call ->writepage(), e.g.,                                              
   |        +----------------+                                                     
   |        | ext4_writepage |                                                     
   |        +----------------+                                                     
   |                                                                               
   |------> return 'success'                                                       
   |                                                                               
   +--> return 'clean'       
   
+-------------------+                                                                                                
| try_to_free_pages | : set up scan control and shrink memory node                                                   
+----|--------------+                                                                                                
     |                                                                                                               
     |--> set up scan control                                                                                        
     |                                                                                                               
     |    +-------------------------+                                                                                
     |--> | throttle_direct_reclaim | check if we should throttole (disallow) task to reclaim                        
     |    +-------------------------+                                                                                
     |                                                                                                               
     |--> return if it should be throttled                                                                           
     |                                                                                                               
     |    +----------------------+                                                                                   
     +--> | do_try_to_free_pages | : shrink memory node                                                              
          +-----|----------------+                                                                                   
                |                                                                                                    
                |--> while priority >= 0                                                                             
                |                                                                                                    
                |        +--------------+                                                                            
                +------> | shrink_zones | : for each zone zonelist, shrink its memory node                           
                         +---|----------+                                                                            
                             |                                                                                       
                             |--> for each zone zonelist                                                             
                             |                                                                                       
                             |        +-------------+                                                                
                             +------> | shrink_node | shrink lru lists and slabs of the given memory node till enough
                                      +-------------+                                                                

 +-------------+                                                                      
 | shrink_node | ： shrink lru lists and slabs of the given memory node till enough    
 +---|---------+                                                                      
     |                                                                                
     |--> get lru list from node                                                      
 again:                                                                               
     |    +--------------------+                                                      
     |--> | shrink_node_memcgs | : shrink lru lists and slabs of the given memory node
     |    +----|---------------+                                                      
     |         |                                                                      
     | `       |--> get lru list from node                                            
     |         |                                                                      
     |         |    +---------------+                                                 
     |         |--> | shrink_lruvec | shrink active and inactive lists                
     |         |    +---------------+                                                 
     |         |    +-------------+                                                   
     |         +--> | shrink_slab | ask each shrinker to reclaim memory               
     |              +-------------+                                                   
     |    +-------------------------+                                                 
     |--> | should_continue_reclaim |                                                 
     |    +-------------------------+                                                 
     |                                                                                
     +--> go to 'again:' if we should                                                 

+---------------+                                                                  
| shrink_lruvec | : shrink active and inactive lists                               
+---|-----------+                                                                  
    |    +----------------+                                                        
    |--> | get_scan_count | determine how much we should scan the lru lists        
    |    +----------------+                                                        
    |    +----------------+                                                        
    |--> | blk_start_plug |                                                        
    |    +----------------+                           nr[0] = anon inactive        
    |                                                 nr[1] = anon active          
    |--> while nr[0] or nr[2] or nr[3] has value      nr[2] = file inactive        
    |                                                 nr[3] = file active          
    |------> for each evictable lru list                                           
    |                                                                              
    |            +-------------+                                                   
    |----------> | shrink_list | shrink the active or inactive list                
    |            +-------------+                                                   
    |                                                                              
    |------> update nr[ ]                                                          
    |                                                                              
    |    +-----------------+                                                       
    |--> | blk_finish_plug |                                                       
    |    +-----------------+                                                       
    |                                                                              
    |--> if inactive lru is low                                                    
    |                                                                              
    |        +--------------------+                                                
    +------> | shrink_active_list | move pages from inactive lru to active lru list
             +--------------------+                                                
```

```
+-------------+                                                                    
| shrink_list | : shrink the active or inactive list                               
+---|---------+                                                                    
    |                                                                              
    |--> if the given lru is active                                                
    |                                                                              
    |        +--------------------+                                                
    |------> | shrink_active_list | r
    |        +--------------------+                                                
    |                                                                              
    |--> else                                                                      
    |                                                                              
    |        +----------------------+                                              
    +------> | shrink_inactive_list | reclaim the inactive list                    
             +----------------------+                                              
```

```
+--------------------+                                                                         
| shrink_active_list | : move pages from inactive lru to active lru list                       
+----|---------------+                                                                         
     |                                                                                         
     |-> if the give lru list is active                                                        
     |                                                                                         
     |    +---------------+                                                                    
     |--> | lru_add_drain | collect pages from other lru lists to memory node, and release them
     |    +---------------+                                                                    
     |    +-------------------+                                                                
     |--> | isolate_lru_pages | move a number of pages to dst list                             
     |    +-------------------+                                                                
     |                                                                                         
     |--> while the 'hold' list has something                                                  
     |                                                                                         
     |------> remove the first page from it                                                    
     |                                                                                         
     |------> clear page 'active' label                                                        
     |                                                                                         
     |------> move page to local 'inactive' list                                               
     |                                                                                         
     |    +-------------------+                                                                
     |--> | move_pages_to_lru | for each page in list, label 'lru' and move to lru list        
     |    +-------------------+                                                                
     |    +-------------------+                                                                
     |--> | move_pages_to_lru | for each page in list, label 'lru' and move to lru list        
     |    +-------------------+                                                                
     |    +----------------------+                                                             
     +--> | free_unref_page_list | free those unused pages                                     
          +----------------------+                                                             
```

```
+-------------------+                                     
| isolate_lru_pages | : move a number of pages to dst list
+----|--------------+                                     
     |                                                    
     +--> while the work isn't done                       
     |                                                    
     |        +-------------+                             
     |------> | lru_to_page | get page from lru list      
     |        +-------------+                             
     |        +-----------+                               
     +------> | list_move | move page to dst list         
              +-----------+                               
```

```
+-------------------+                                                          
| move_pages_to_lru | : for each page in list, label 'lru' and move to lru list
+----|--------------+                                                          
     |                                                                         
     |--> while list has something                                             
     |                                                                         
     |------> remove the first page from list                                  
     |                                                                         
     |------> set label 'lru' on page                                          
     |                                                                         
     |        +----------------------+                                         
     +------> | add_page_to_lru_list | move page to the corresponding lru list 
              +----------------------+                                         
```

```
+----------------------+                                                                        
| shrink_inactive_list | : reclaim the inactive list                                            
+-----|----------------+                                                                        
      |    +---------------+                                                                    
      |--> | lru_add_drain | collect pages from other lru lists to memory node, and release them
      |    +---------------+                                                                    
      |    +-------------------+                                                                
      |--> | isolate_lru_pages | move a number of pages to dst list                             
      |    +-------------------+                                                                
      |    +------------------+                                                                 
      |--> | shrink_page_list | try to reclaim the given page list                              
      |    +------------------+                                                                 
      |    +-------------------+                                                                
      |--> | move_pages_to_lru | for each page in list, label 'lru' and move to lru list        
      |    +-------------------+                                                                
      |    +----------------------+                                                             
      +--> | free_unref_page_list | free those unused pages                                     
           +----------------------+                                                             

+-------------+                                                                        
| shrink_slab | : ask each shrinker to reclaim memory                                  
+---|---------+                                                                        
    |                                                                                  
    |--> for each shrinker                                                             
    |                                                                                  
    |        +----------------+                                                        
    +------> | do_shrink_slab | : shrink slab                                          
             +---|------------+                                                        
                 |                                                                     
                 +--> call ->scan_objects(), e.g.,                                     
                      +------------------+                                             
                      | super_cache_scan | : shrink dentries and inodes of the given sb
                      +------------------+                                             

+------------------+                                                                           
| super_cache_scan | : shrink dentries and inodes of the given sb                              
+----|-------------+                                                                           
     |                                                                                         
     |--> get 'sb' struct                                                                      
     |                                                                                         
     |    +-----------------+                                                                  
     |--> | prune_dcache_sb | isolate dentries to a local list, and try to release each of them
     |    +-----------------+                                                                  
     |    +-----------------+                                                                  
     +--> | prune_icache_sb | isolate inodes to a local list, and try to release each of them  
          +-----------------+                                                                  

+-----------------+                                                                                 
| prune_dcache_sb | : isolate dentries to a local list, and try to release each of them             
+----|------------+                                                                                 
     |    +----------------------+                                                                  
     |--> | list_lru_shrink_walk | for each item in list, apply callback isolate(), return isolated#
     |    +----------------------+                                                                  
     |    +--------------------+                                                                    
     +--> | shrink_dentry_list | try to release each dentry in list                                 
          +--------------------+                                                                    
```

```
+----------------------+                                                                                        
| list_lru_shrink_walk | : for each item in list, apply callback isolate(), return isolated#                    
+-----|----------------+                                                                                        
      |    +-------------------+                                                                                
      +--> | list_lru_walk_one |                                                                                
           +----|--------------+                                                                                
                |    +---------------------+                                                                    
                +--> | __list_lru_walk_one | : for each item in list, apply callback isolate(), return isolated#
                     +-----|---------------+                                                                    
                           |                                                                                    
                           |--> for each item in list                                                           
                           |                                                                                    
                           |------> break if no more to talk                                                    
                           |                                                                                    
                           |------> nr_to_walk--                                                                
                           |                                                                                    
                           +------> call isolate(), e.g.,                                                       
                                    +--------------------+                                                      
                                    | dentry_lru_isolate | either remove dentry from list, or add to arg list   
                                    +--------------------+                                                      
```

```
+--------------------+                                                            
| dentry_lru_isolate | : either remove dentry from list, or add to arg list       
+----|---------------+                                                            
     |                                                                            
     |--> if dentry still has ref count                                           
     |                                                                            
     |        +---------------+                                                   
     |------> | d_lru_isolate | clear 'lru_list' in flags, remove dentry from list
     |        +---------------+                                                   
     |                                                                            
     |------> return 'removed'                                                    
     |                                                                            
     |    +-------------------+                                                   
     |--> | d_lru_shrink_move | set 'lru_list' in flags, add dentry to arg list   
     |    +-------------------+                                                   
     |                                                                            
     +--> return 'removed'                                                        
```

```
+--------------------+                                     
| shrink_dentry_list | : try to release each dentry in list
+----|---------------+                                     
     |                                                     
     |--> while list has something                         
     |                                                     
     |------> get dentry from list                         
     |                                                     
     |        +--------------+                             
     |------> | d_shrink_del | remove dentry from list     
     |        +--------------+                             
     |        +---------------+                            
     +------> | __dentry_kill | release dentry             
              +---------------+                            
```

```
+------------------------+                                     
| lru_deactivate_file_fn | : move page to inactive list        
+-----|------------------+                                     
      |    +------------------------+                          
      |--> | del_page_from_lru_list | remove page from lru list
      |    +------------------------+                          
      |    +-----------------+                                 
      |--> | ClearPageActive |                                 
      |    +-----------------+                                 
      |                                                        
      |--> if page has 'writeback' or 'dirty' label            
      |                                                        
      |        +----------------------+                        
      |------> | add_page_to_lru_list | add page to lru list   
      |        +----------------------+                        
      |        +----------------+                              
      |------> | SetPageReclaim |                              
      |        +----------------+                              
      |                                                        
      |--> else (writeback finishes)                           
      |                                                        
      |        +---------------------------+                   
      +------> | add_page_to_lru_list_tail |                   
               +---------------------------+                   
```

```
+----------------------+                                                                             
| deactivate_file_page | : deactivate a file page (add it to pvec, flush them if full)               
+-----|----------------+                                                                             
      |    +----------------------------+                                                            
      |--> | pagevec_add_and_need_flush | add page to pvec                                           
      |    +----------------------------+                                                            
      |                                                                                              
      |--> if pvec is full afterwards                                                                
      |                                                                                              
      |        +---------------------+                                                               
      +------> | pagevec_lru_move_fn | move pages in pvec to lru list of memory node and release them
               +---------------------+                                                               
```

```
+-----------------+                                                                                 
| prune_icache_sb | : isolate inodes to a local list, and try to release each of them               
+----|------------+                                                                                 
     |    +----------------------+                                                                  
     |--> | list_lru_shrink_walk | for each item in list, apply callback isolate(), return isolated#
     |    +----------------------+                                                                  
     |    +--------------+                                                                          
     +--> | dispose_list | evict each inode in list                                                 
          +--------------+                                                                          
```

```
+-------------------+                                                              
| inode_lru_isolate | : isolate inode from lru list for release preparation        
+---|---------------+                                                              
    |                                                                              
    |--> if inode is still in use                                                  
    |                                                                              
    |        +------------------+                                                  
    |------> | list_lru_isolate | remove inode from list                           
    |        +------------------+                                                  
    |                                                                              
    |------> return 'removed'                                                      
    |                                                                              
    |                                                                              
    |--> if inode has buffer or it has no mapping                                  
    |                                                                              
    |        +----------------------+                                              
    |------> | remove_inode_buffers | remove clean buffers from inode              
    |        +----------------------+                                              
    |                                                                              
    |------> if all buffers are removed                                            
    |                                                                              
    |            +--------------------------+                                      
    |----------> | invalidate_mapping_pages | invalidate all clean pages in mapping
    |            +--------------------------+                                      
    |                                                                              
    +------> return 'retry'                                                        
    |                                                                              
    +--> set 'freeing' in inode state                                              
    |                                                                              
    |    +-----------------------+                                                 
    |--> | list_lru_isolate_move | move inode to arg list                          
    |    +-----------------------+                                                 
    |                                                                              
    +--> return 'removed'                                                          
    
    
    
+---------------+                                                                                
| lru_cache_add | : add page to percpu pvec, if full, flush to node lru list                     
+---|-----------+                                                                                
    |                                                                                            
    |--> get per cpu pvec                                                                        
    |                                                                                            
    |    +----------------------------+                                                          
    |--> | pagevec_add_and_need_flush |                                                          
    |    +----------------------------+                                                          
    |                                                                                            
    |--> if pvec if full after addition                                                          
    |                                                                                            
    |        +-------------------+                                                               
    +------> | __pagevec_lru_add | move pages in pvec to lru list of memory node and release them
             +-------------------+                                                                   
```

## <a name="reference"></a> Reference

(TBD)
