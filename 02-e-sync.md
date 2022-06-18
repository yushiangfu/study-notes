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

## <a name="reference"></a> Reference

(TBD)
