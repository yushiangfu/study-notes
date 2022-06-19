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

## <a name="reference"></a> Reference

(TBD)
