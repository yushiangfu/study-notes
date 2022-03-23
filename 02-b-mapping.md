## Index

- [Introduction](#introduction)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

```
+------------------------+                                                       
| generic_file_read_iter |                                                       
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
                                                 |------> else                                           |  simplified logic
                                                 |                                                       |                  
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
     |------> | filemap_create_page | alocate a page, read data to it, and add the page to cache
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

## <a name="reference"></a> Reference

(TBD)



