## Index

- [Introduction](#introduction)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

```
+------------------------+                                                       
| generic_file_read_iter | read data from page of mapping
+-----|------------------+                                                       
      |                                                                          
      |--> if 'DIRECT' flag is set                                               
      |                                                                          
      |------> (skip, not my case here)                                          
      |                                                                          
      |    +--------------+                                                      
      +--> | filemap_read |                                                      
           +---|----------+                                                      
               |                                                                 
               |--> while we haven't read enough                                 
               |                                                                 
               |        +-------------------+                                    
               |------> | filemap_get_pages | read data from page to pvec        
               |        +-------------------+ (refer to the below)               
               |                                                                 
               |------> for each page in pvec                                    
               |                                                                 
               |            +-------------------+                                
               +----------> | copy_page_to_iter | copy data from page to iterator
                            +-------------------+                                
```

```
+-------------------+                                                                           
| filemap_get_pages | ensure data is there and up-to-date                                       
+----|--------------+                                                                           
     |    +------------------------+                                                            
     |--> | filemap_get_read_batch | save a bunch of page addresses in pvec                     
     |    +------------------------+                                                            
     |                                                                                          
     |--> if get nothing                                                                        
     |                                                                                          
     |        +---------------------------+                                                     
     |------> | page_cache_sync_readahead |                                                     
     |        +---------------------------+                                                     
     |        +------------------------+                                                        
     |------> | filemap_get_read_batch | save a bunch of page addresses in pvec                 
     |        +------------------------+                                                        
     |                                                                                          
     |--> if still get nothing                                                                  
     |                                                                                          
     |        +---------------------+                                                           
     |------> | filemap_create_page | allocate a page, read data to it, and add the page to cache
     |        +---------------------+                                                           
     |                                                                                          
     |------> return                                                                            
     |                                                                                          
     |                                                                                          
     |--> get the last page in pvec                                                             
     |                                                                                          
     |--> if it's labeled READAHEAD                                                             
     |                                                                                          
     |        +-------------------+                                                             
     |------> | filemap_readahead |                                                             
     |        +----|--------------+                                                             
     |             |    +----------------------------+                                          
     |             +--> | page_cache_async_readahead |                                          
     |                  +----------------------------+                                          
     |                                                                                          
     +--> ensure the pages are up-to-date                                                       
```

```
+---------------------------+                                                                                               
| page_cache_sync_readahead |                                                                                               
+------|--------------------+                                                                                               
       |                                                                                                                    
       +--> prepare readahead parameters                                                                                    
       |                                                                                                                    
       |    +--------------------+                                                                                          
       +--> | page_cache_sync_ra |                                                                                          
            +----|---------------+                                                                                          
                 |    +--------------------+                                                                                
                 +--> | ondemand_readahead |                                                                                
                      +----|---------------+                                                                                
                           |                                                                                                
                           |--> adjust readahead parameters                                                                 
                           |                                                                                                
                           |    +------------------+                                                                        
                           +--> | do_page_cache_ra |                                                                        
                                +----|-------------+                                                                        
                                     |    +-------------------------+                                                       
                                     +--> | page_cache_ra_unbounded |                                                       
                                          +------|------------------+                                                       
                                                 |                                                                          
                                                 |--> for each index in the mapping range                                   
                                                 |                                                                          
                                                 |------> if page exist already                         -+                  
                                                 |                                                       |                  
                                                 |            +------------+                             |                  
                                                 |----------> | read_pages | (traced separately)         |                  
                                                 |            +------------+                             |                  
                                                 |                                                       |                  
                                                 |------> else                                           |  simplified
                                                 |                                                       |  logic
                                                 |            +--------------------+                     |                  
                                                 |----------> | __page_cache_alloc |                     |                  
                                                 |            +--------------------+                     |                  
                                                 |            +----------+                               |                  
                                                 |----------> | list_add | add the page to a local pool  |                  
                                                 |            +----------+                              -+                  
                                                 |    +------------+                                                        
                                                 +--> | read_pages | (traced separately)                                    
                                                      +------------+                                                        
```

```
+------------+                                                
| read_pages |                                                
+--|---------+                                                
   |    +----------------+                                    
   |--> | blk_start_plug | prepare a plug for the current task
   |    +----------------+                                    
   |                                                          
   |--> if ->readahead() exists                               
   |                                                          
   |------> call ->readahead(), e.g.,                         
   |        +------------------+                              
   |        | blkdev_readahead | (refer to the below)         
   |        +------------------+                              
   |                                                          
   |--> else if ->readpages() exists                          
   |                                                          
   |------> call ->readpages(), e.g.,                         
   |        +---------------+                                 
   |        | nfs_readpages |                                 
   |        +---------------+                                 
   |                                                          
   |--> else                                                  
   |                                                          
   |------> loop                                              
   |                                                          
   |----------> call ->readpage(), e.g.,                      
   |            +-----------------+                           
   |            | blkdev_readpage | (refer to the below)      
   |            +-----------------+                           
   |    +-----------------+                                   
   +--> | blk_finish_plug | (refer to the below)              
        +-----------------+                                   
```

```
+---------------------+                                                         
| filemap_create_page |                                                         
+-----|---------------+                                                         
      |    +------------------+                                                 
      |--> | page_cache_alloc | get a free page                                 
      |    +------------------+                                                 
      |    +-----------------------+                                            
      |--> | add_to_page_cache_lru | add the page to mapping and percpu lru list
      |    +-----------------------+                                            
      |    +-------------------+                                                
      |--> | filemap_read_page |                                                
      |    +----|--------------+                                                
      |         |                                                               
      |         +--> call ->readpage(), e.g.,                                   
      |              +-----------------+                                        
      |              | blkdev_readpage |                                        
      |              +-----------------+                                        
      |    +-------------+                                                      
      +--> | pagevec_add | add the page to pvec                                 
           +-------------+                                                      
```

```
+---------------+                                                                                                                   
| sync_blockdev | sync bdev mapping to back storage                                                                                 
+---|-----------+                                                                                                                   
    |    +-----------------+                                                                                                        
    +--> | __sync_blockdev |                                                                                                        
         +----|------------+                                                                                                        
              |                                                                                                                     
              |--> if caller doesn't wait                                                                                           
              |                                                                                                                     
              |        +---------------+                                                                                            
              |------> | filemap_flush |                                                                                            
              |        +---------------+                                                                                            
              |                                                                                                                     
              |--> else                                                                                                             
              |                                                                                                                     
              |        +------------------------+                                                                                   
              +------> | filemap_write_and_wait |                                                                                   
                       +-----|------------------+                                                                                   
                             |    +------------------------------+                                                                  
                             +--> | filemap_write_and_wait_range |                                                                  
                                  +-------|----------------------+                                                                  
                                          |    +----------------------------+                                                       
                                          |--> | __filemap_fdatawrite_range |                                                       
                                          |    +------|---------------------+                                                       
                                          |           |    +------------------------+                                               
                                          |           +--> | filemap_fdatawrite_wbc |                                               
                                          |                +-----|------------------+                                               
                                          |                      |    +---------------+                                             
                                          |                      +--> | do_writepages | with specified range, write dirty pages back
                                          |                           +---------------+                                             
                                          |    +-------------------------+                                                          
                                          +--> | filemap_fdatawait_range | wait till WRITEBACK pages finish                         
                                               +-------------------------+                                                          
```

```
+---------------+                                                                                                         
| do_writepages | with specified range, write dirty pages back                                                            
+---|-----------+                                                                                                         
    |                                                                                                                     
    |--> if ->writepages() exists                                                                                         
    |                                                                                                                     
    +------> call ->writepages(), e.g.,                                                                                   
    |        +-------------------+      +--------------------+                                                            
    |        | blkdev_writepages | ---> | generic_writepages |                                                            
    |        +-------------------+      +--------------------+                                                            
    |                                                                                                                     
    |--> else                                                                                                             
    |                                                                                                                     
    |        +--------------------+                                                                                       
    +------> | generic_writepages |                                                                                       
             +----|---------------+                                                                                       
                  |    +----------------+                                                                                 
                  |--> | blk_start_plug | prepare plug for current task                                                   
                  |    +----------------+                                                                                 
                  |    +-------------------+                                                                              
                  |--> | write_cache_pages | within range, call mapping->writepage() for each dirty page                  
                  |    +-------------------+                                                                              
                  |    +-----------------+                                                                                
                  +--> | blk_finish_plug | for each entity in list, add to a queue (e.g., io scheduler queue or mtd queue)
                       +-----------------+                                                                                
```

```
+---------------------------+                                                      
| __generic_file_write_iter | copy data to pages of mapping, mark them dirty       
+------|--------------------+                                                      
       |                                                                           
       |--> if 'DIRECT' flag is set                                                
       |                                                                           
       |------> (skip, not my case here)                                           
       |                                                                           
       |--> else                                                                   
       |                                                                           
       |        +-----------------------+                                          
       +------> | generic_perform_write |                                          
                +-----|-----------------+                                          
                      |                                                            
                      |--> while we haven't written enough                         
                      |                                                            
                      |------> call ->write_begin(), e.g.,                         
                      |        +--------------------+                              
                      |        | blkdev_write_begin | get a page from mapping      
                      |        +--------------------+                              
                      |        +----------------------------+                      
                      |------> | copy_page_from_iter_atomic | copy data to the page
                      |        +----------------------------+                      
                      |                                                            
                      +------> call ->write_end(), e.g.,                           
                               +------------------+                                
                               | blkdev_write_end | mark buffer dirty              
                               +------------------+                                
```

```
+---------------------+                                          
| unmap_mapping_range |                                          
+-----|---------------+                                          
      |                                                          
      |--> ensure the start and len is page-size aligned         
      |                                                          
      |    +---------------------+                               
      +--> | unmap_mapping_pages | unmap all the vma within range
           +---------------------+                               
```

```
+---------------+                                                                        
| release_pages | drop ref for each page in array, free the pages whose ref reaches 0
+---|-----------+                                                                        
    |                                                                                    
    |--> for each page in array                                                          
    |                                                                                    
    |------> page ref--                                                                  
    |                                                                                    
    |------> if ref != 0 (it's still used), continue                                     
    |                                                                                    
    |------> if page is set 'lru'                                                        
    |                                                                                    
    |            +------------------------+                                              
    |----------> | del_page_from_lru_list | remove page from list (lruvec of memory node)
    |            +------------------------+                                              
    |            +------------------------+                                              
    |----------> | __clear_page_lru_flags | clear 'lru', 'active', 'unevictable' on page 
    |            +------------------------+                                              
    |                                                                                    
    |------> add page to a local list                                                    
    |                                                                                    
    |    +----------------------+                                                        
    +--> | free_unref_page_list | free each page in the local list                       
         +----------------------+                                                        
```

```
+-------------------+                                                                                           
| __pagevec_lru_add | move pages in pvec to lru list of memory node and release them                            
+----|--------------+                                                                                           
     |                                                                                                          
     |--> for each page in pvec                                                                                 
     |        +----------------------------+                                                                    
     |        | relock_page_lruvec_irqsave | lock and get lruvec from memory node                               
     |        +----------------------------+                                                                    
     |        +----------------------+                                                                          
     |        | __pagevec_lru_add_fn |                                                                          
     |        +-----|----------------+                                                                          
     |              |                                                                                           
     |              |--> label 'lru' on page                                                                    
     |              |                                                                                           
     |              |    +----------------------+                                                               
     |              +--> | add_page_to_lru_list | add page to the proper list in lruvec based on page attributes
     |                   +----------------------+                                                               
     |    +-------------------------------+                                                                     
     |--> | unlock_page_lruvec_irqrestore | unlock                                                              
     |    +-------------------------------+                                                                     
     |    +---------------+                                                                                     
     +--> | release_pages | drop ref for each page in array, free the pages whose ref reaches 0                 
          +---------------+                                                                                     
```

## <a name="reference"></a> Reference

(TBD)



