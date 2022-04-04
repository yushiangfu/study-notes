> Study case: Linux version 5.15.0 on AST2500 emulation

## Index

- [Introduction](#introduction)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

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
+------------------+                                                              
| blkdev_readahead |                                                              
+----|-------------+                                                              
     |    +-----------------+                                                     
     +--> | mpage_readahead |                                                     
          +----|------------+                                                     
               |                                                                  
               |--> while we can get next page buffer from rac (readahead control)
               |                                                                  
               |        +-------------------+                                     
               |------> | do_mpage_readpage |                                     
               |        +----|--------------+                                     
               |             |                                                    
               |             |--> build 'blocks'                                  
               |             |                                                    
               |             |--> ensure we have a 'bio'                          
               |             |                                                    
               |             |    +--------------+                                
               |             |--> | bio_add_page | add page to the bio            
               |             |    +--------------+                                
               |             |    +------------------+                            
               |             +--> | mpage_bio_submit | submit bio                 
               |                  +------------------+                            
               |                                                                  
               |--> if it's not yet submitted for any reason                      
               |                                                                  
               |        +------------------+                                      
               +------> | mpage_bio_submit | submit bio                           
                        +------------------+                                      
```

```
+-----------------+                                         
| blkdev_readpage |                                         
+----|------------+                                         
     |    +----------------------+                          
     +--> | block_read_full_page |                          
          +-----|----------------+                          
                |    +---------------------+                
                |--> | create_page_buffers | prepare bh list
                |    +---------------------+                
                |                                           
                |--> build array with the bh list           
                |                                           
                |--> for each entity in array               
                |                                           
                |        +-----------+                      
                +------> | submit_bh |                      
                         +-----------+                      
```

```
+-----------------+                                                                                                         
| blk_finish_plug |                                                                                                         
+----|------------+                                                                                                         
     |    +---------------------+                                                                                           
     |--> | blk_flush_plug_list |                                                                                           
     |    +-----|---------------+                                                                                           
     |          |    +----------------------+                                                                               
     |          |--> | flush_plug_callbacks | if plug has callback list, execute each of them                               
     |          |    +----------------------+                                                                               
     |          |    +------------------------+                                                                             
     |          +--> | blk_mq_flush_plug_list |                                                                             
     |               +-----|------------------+                                                                             
     |                     |                                                                                                
     |                     |--> for each entity in mq list                                                                  
     |                     |                                                                                                
     |                     |        +------------------------------+                                                        
     |                     +------> | blk_mq_sched_insert_requests |                                                        
     |                              +-------|----------------------+                                                        
     |                                      |                                                                               
     |                                      |--> attempt to get elevator from queue                                         
     |                                      |                                                                               
     |                                      |--> if elevator exists                                                         
     |                                      |                                                                               
     |                                      |------> call ->insert_requests()                                               
     |                                      |                                                                               
     |                                 +-   |--> else                                                                       
     |                     (simplified |    |                                                                               
     |                        logic)   |    |        +--------------------------------+                                     
     |                                 +-   +------> | blk_mq_try_issue_list_directly | eventually trigger the mtd operation
     |                                               +--------------------------------+                                     
     |                                                                                                                      
     +--> unset current task plug                                                                                           
```

```
+--------------------------------+                                                          
| blk_mq_try_issue_list_directly |                                                          
+-------|------------------------+                                                          
        |                                                                                   
        |--> while list isn't empty yet                                                     
        |                                                                                   
        |------> remove first entry from list                                               
        |                                                                                   
        |        +-------------------------------+                                          
        +------> | blk_mq_request_issue_directly |                                          
                 +-------|-----------------------+                                          
                         |    +-----------------------------+                               
                         +--> | __blk_mq_try_issue_directly |                               
                              +-------|---------------------+                               
                                      |    +-------------------------+                      
                                      +--> | __blk_mq_issue_directly |                      
                                           +------|------------------+                      
                                                  |                                         
                                                  +--> call ->queue_rq(), e.g.,             
                                                       +--------------+                     
                                                       | mtd_queue_rq | (refer to the below)
                                                       +--------------+                     
```

```
+--------------------+                                                                                
| blkdev_write_begin |                                                                                
+----|---------------+                                                                                
     |    +-------------------+                                                                       
     +--> | block_write_begin |                                                                       
          +----|--------------+                                                                       
               |    +-----------------------------+                                                   
               |--> | grab_cache_page_write_begin | get a page from mapping, or allocate one otherwise
               |    +-----------------------------+                                                   
               |    +---------------------+                                                           
               +--> | __block_write_begin |                                                           
                    +-----|---------------+                                                           
                          |    +-------------------------+                                            
                          +--> | __block_write_begin_int |                                            
                               +------|------------------+                                            
                                      |                                                               
                                      |--> for each block head (bh) in list                           
                                      |                                                               
                                      +------> adjust bh status                                       
```

## <a name="reference"></a> Reference

(TBD)
