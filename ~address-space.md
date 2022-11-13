## Index

- [Introduction](#introduction)
- [Reference](#reference)


## <a name="introduction"></a> Introduction

Address space is the concept of either file or device content, and it plays the role of page cache between user applications and content on the device. 
Usually, backing stores such as drives work relatively slowly compared to the operation in memory, and the page cache is designed to solve the problem. 
Applications can operate data smoothly in memory while the kernel handles the read, write, and sync action toward the storage.

The management data structure is **xarray**, renamed from the **radix tree** because it acts like a resizable array and has an index-based lookup. 
Each node contains a fixed number of pointers that point to either:

- nowhere
- next level node
- end leaf, usually **page**

```
                                   xarray                     
                               (or radix tree)                
                                                              
                               +-------------+                
                               | | | | | | | |                
                               +-------------+                
                                    |   |                     
                    +---------------+   +-------+             
                    v                           v             
                   +-------------+             +-------------+
                   | | | | | | | |             | | | | | | | |
                   +-------------+             +-------------+
                      |   |                       |       |   
                      v   v                       v       v   
                     +-+ +-+                     +-+    +-+   
                     |p| |p|                     |p|    |p|   
                     |a| |a|                     |a|    |a|   
                     |g| |g|                     |g|    |g|   
   file              |e| |e|                     |e|    |e|   
    in               +-+ +-+                     +-+    +-+   
  concept                                                     
  +-----+             |   |                       |      |    
  |     |             |   |                       |      |    
  |||||||  <----------+   |                       |      |    
  |     |                 |                       |      |    
  |     |                 |                       |      |    
  |||||||  <--------------+                       |      |    
  |     |                                         |      |    
  |     |                                         |      |    
  |||||||  <--------------------------------------+      |    
  |||||||  <---------------------------------------------+    
  |     |                                                     
  +-----+                                                     
```

Assuming the address space represents the content of a file, only the pieces of interest are within the tree while others remain untouched in storage. 
The address operation set (aops) is installed initially and triggered when reading and writing operations happen. 
Here are the examples of **aops** for block devices and files separately.

```
                  +--------------+                      |                  +----------------+                  
                  | block device |                      |                  | file＠shmem_fs  |                  
                  +--------------+                      |                  +----------------+                  
                                                        |                                                      
                                                        |                                                      
 const struct address_space_operations def_blk_aops = { | const struct address_space_operations shmem_aops = { 
     .set_page_dirty = __set_page_dirty_buffers,        |     .writepage         = shmem_writepage,           
     .readpage           = blkdev_readpage,             |     .set_page_dirty 	 = __set_page_dirty_no_writeback,
     .readahead          = blkdev_readahead,            |     .write_begin       = shmem_write_begin,         
     .writepage          = blkdev_writepage,            |     .write_end         = shmem_write_end,           
     .write_begin        = blkdev_write_begin,          |     .migratepage    	 = migrate_page,                 
     .write_end          = blkdev_write_end,            |     .error_remove_page = generic_error_remove_page, 
     .writepages         = blkdev_writepages,           | };                                                   
     .direct_IO          = blkdev_direct_IO,            |                                                      
     .migratepage        = buffer_migrate_page_norefs,  |                                                      
     .is_dirty_writeback = buffer_check_dirty_writeback,|                                                      
 };                                                     |                                                      
```

When a user reads a range within a file, the VFS layer reads data from the xarray, the page cache between applications, and the backing store. 
For those absent slots, the address space layer submits bio requests queued in the task plug list. 
The kernel also attempts to perform some asynchronous read, which reduces the waiting time if the following content is accessed soon.
Then the VFS layer flushes all the bio requests to the block layer for actual IO processing and installs pages to the xarray.

As for the write operation, the kernel first writes data to the xarray and marks involved papes dirty. 
Later it syncs all the dirtied pages to the backing store in a way specified by address space ops.

```
    vfs
                                       │
                                       │
        ───────────────────────────────┼───────────────────────────────────
                                       │
                                       ▼
                                     xarray
                                 (or radix tree)

                                 +-------------+
                                 | | | | | | | |
                                 +-------------+
                                      |   |
                      +---------------+   +-------+
                      v                           v
address              +-------------+             +-------------+
  space              | | | | | | | |             | | | | | | | |
                     +-------------+             +-------------+
                        |   |     |                 |   |   |
                        v   v     v                 v   v   v
                       +-+ +-+                     +-+    +-+
                       |p| |p|    ?                |p|  ? |p|
                       |a| |a|                     |a|    |a|
                       |g| |g|                     |g|    |g|
                       |e| |e|                     |e|    |e|
                       +-+ +-+    │                +-+  │ +-+
                                  │                     │
        ──────────────────────────┼─────────────────────┼──────────────────
                                  │                     │
                                  ▼                     ▼
  block                         ┌───┐                 ┌───┐
  layer                         │bio│                 │bio│
                                └───┘                 └───┘
```

<details>
  <summary> Code trace </summary>

```
+------------------+                                                            
| blkdev_read_iter |                                                            
+----|-------------+                                                            
     |    +------------------------+                                            
     +--> | generic_file_read_iter | read data from page of mapping
          +------------------------+                                            
```

```
+------------------------+                                                       
| generic_file_read_iter | : read data from page of mapping
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
| filemap_get_pages | : ensure data is there and up-to-date                                       
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
| page_cache_sync_readahead | : ensure pages exist, and read data to them
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
                                     +--> | page_cache_ra_unbounded | : ensure pages exist, and read data to them
                                          +------|------------------+                                                       
                                                 |                                                                          
                                                 |--> for each index in the mapping range                                   
                                                 |                                                                          
                                                 |------> if page exist already                         -+                  
                                                 |                                                       |                  
                                                 |            +------------+                             |                  
                                                 |----------> | read_pages | plug, submit read-type bio, unplug
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
| read_pages | : plug, submit read-type bio, unplug
+--|---------+                                                
   |    +----------------+                                    
   |--> | blk_start_plug | prepare a plug for the current task
   |    +----------------+                                    
   |                                                          
   |--> if ->readahead() exists                               
   |                                                          
   |------> call ->readahead(), e.g.,                         
   |        +------------------+                              
   |        | blkdev_readahead | prepare a bio, add page buffer to it, and submit that bio
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
+------------------+                                                              
| blkdev_readahead | : prepare a bio, add page buffer to it, and submit that bio
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
| blkdev_write_iter |                                                                                        
+----|--------------+                                                                                        
     |    +----------------+                                                                                 
     |--> | blk_start_plug |                                                                                 
     |    +----------------+                                                                                 
     |    +---------------------------+                                                                      
     |--> | __generic_file_write_iter | copy data to pages of mapping, mark them dirty                       
     |    +---------------------------+                                                                      
     |    +--------------------+                                                                             
     |--> | generic_write_sync | sync back to storage                                                        
     |    +--------------------+                                                                             
     |    +-----------------+                                                                                
     +--> | blk_finish_plug | for each entity in list, add to a queue (e.g., io scheduler queue or mtd queue)
          +-----------------+                                                                                
```

```
+---------------------------+                                                      
| __generic_file_write_iter | : copy data to pages of mapping, mark them dirty       
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

```
+------------------+                                          
| blkdev_write_end |                                          
+----|-------------+                                          
     |    +-----------------+                                 
     +--> | block_write_end |                                 
          +----|------------+                                 
               |    +----------------------+                  
               +--> | __block_commit_write | mark buffer dirty
                    +----------------------+                  
```

```
+---------------+                                                                                                                   
| sync_blockdev | : sync bdev mapping to back storage                                                                                 
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
                             +--> | filemap_write_and_wait_range | : writeback dirty pages and wait till 'writeback' cleared
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
| do_writepages | : with specified range, write dirty pages back                                                            
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
+------------------+                                    
| blkdev_writepage |                                    
+----|-------------+                                    
     |    +-----------------------+                     
     +--> | block_write_full_page |                     
          +-----|-----------------+                     
                |    +-------------------------+        
                +--> | __block_write_full_page |        
                     +------|------------------+        
                            |    +---------------------+
                            +--> | create_page_buffers |
                                 +---------------------+
                                 +---------------+      
                                 | submit_bh_wbc |      
                                 +---|-----------+      
                                     |    +------------+
                                     +--> | submit_bio |
                                          +------------+
```  
  
</details>
  

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

```
+---------------+                                                                                         
| lru_add_drain | : collect pages from other lru lists to memory node, and release them                     
+---|-----------+                                                                                         
    |                                                                                                     
    |--> get pvec from 'lru_pvecs.lru_add'                                                                
    |                                                                                                     
    |    +-------------------+                                                                            
    |--> | __pagevec_lru_add | move pages in pvec to lru list of memory node and release them             
    |    +-------------------+                                                                            
    |                                                                                                     
    |--> get pvec from 'lru_rotate.pvec'                                                                  
    |                                                                                                     
    |    +---------------------+                                                                          
    |--> | pagevec_lru_move_fn | move pages in pvec to lru list of memory node and release them           
    |    +---------------------+                                                                          
    |                                                                                                     
    |--> get pvec from 'lru_pvecs.lru_deactivate_file'                                                    
    |                                                                                                     
    |    +---------------------+                                                                          
    +--> | pagevec_lru_move_fn | move pages in pvec to lru list of memory node and release them           
    |    +---------------------+                                                                          
    |                                                                                                     
    |--> get pvec from 'lru_pvecs.lru_deactivate'                                                         
    |                                                                                                     
    |    +---------------------+                                                                          
    +--> | pagevec_lru_move_fn | move pages in pvec to lru list of memory node and release them           
    |    +---------------------+                                                                          
    |                                                                                                     
    |--> get pvec from 'lru_pvecs.lru_lazyfree'                                                           
    |                                                                                                     
    |    +---------------------+                                                                          
    |--> | pagevec_lru_move_fn | move pages in pvec to lru list of memory node and release them           
    |    +---------------------+                                                                          
    |    +---------------------+                                                                          
    +--> | activate_page_drain |                                                                          
         +-----|---------------+                                                                          
               |                                                                                          
               |--> get pvec from 'lru_pvecs.activate_page'                                               
               |                                                                                          
               |    +---------------------+                                                               
               +--> | pagevec_lru_move_fn | move pages in pvec to lru list of memory node and release them
                    +---------------------+                                                               
```

```
+--------------------------+                                                                                                
| unmap_mapping_range_tree | for each vma within range, clear page table entries within vma                                 
+------|-------------------+                                                                                                
       |    +---------------------------+                                                                                   
       |--> | vma_interval_tree_foreach |                                                                                   
       |    +---------------------------+                                                                                   
       |                                                                                                                    
       |------> adjust the start and end to ensure they are in the vma ragne                                                
       |                                                                                                                    
       |        +-------------------------+                                                                                 
       +------> | unmap_mapping_range_vma |                                                                                 
                +------|------------------+                                                                                 
                       |    +-----------------------+                                                                       
                       +--> | zap_page_range_single |                                                                       
                            +-----|-----------------+                                                                       
                                  |    +---------------+                                                                    
                                  +--> | lru_add_drain | collect pages from other lru lists to memory node, and release them
                                  |    +---------------+                                                                    
                                  |    +------------------+                                                                 
                                  +--> | unmap_single_vma | clear page table entries within range                           
                                       +------------------+                                                                 
```

```
+---------------------+                                                                           
| try_to_release_page | detatch bh list from page and free them                                   
+-----|---------------+                                                                           
      |                                                                                           
      |--> if page has the 'writeback' label, return                                              
      |                                                                                           
      |--> if mapping has ->releasepage()                                                         
      |                                                                                           
      |------> call ->releasepage()                                                               
      |                                                                                           
      |--> else                                                                                   
      |                                                                                           
      |        +---------------------+                                                            
      +------> | try_to_free_buffers | : detatch bh list from page and free those bh
               +-----|---------------+                                                            
                     |    +--------------+                                                        
                     |--> | drop_buffers | move the bh list to a local head, and detatch from page
                     |    +--------------+                                                        
                     |    +-------------------+                                                   
                     |--> | cancel_dirty_page | clear 'dirty' bit of page                         
                     |    +-------------------+                                                   
                     |                                                                            
                     |--> for each bh in local list                                               
                     |                                                                            
                     |        +------------------+                                                
                     +------> | free_buffer_head | return bh to kmem cache                        
                              +------------------+                                                
```

```
+----------------------+                                                      
| block_invalidatepage | clear some flags for those truncated bh              
+-----|----------------+                                                      
      |                                                                       
      |--> get buffer head from page private                                  
      |                                                                       
      |--> for each buffer in the linked list                                 
      |                                                                       
      |------> if the buffer is above the arg 'offset'                        
      |                                                                       
      |            +----------------+                                         
      |----------> | discard_buffer | clear some flags of the bh              
      |            +----------------+                                         
      |                                                                       
      |--> if arg 'length' is page size                                       
      |                                                                       
      |        +---------------------+                                        
      +------> | try_to_release_page | detatch bh list from page and free them
               +---------------------+                                        
```

```
+---------------------+                                                                                           
| truncate_inode_page | unmap and invalidate page, detatch it from mapping, and has its ref--                     
+-----|---------------+                                                                                           
      |    +-----------------------+                                                                              
      |--> | truncate_cleanup_page | unmpa and invalidate page                                                    
      |    +-----|-----------------+                                                                              
      |          |                                                                                                
      |          |--> if page is set 'mapped'                                                                     
      |          |                                                                                                
      |          |        +--------------------+                                                                  
      |          |------> | unmap_mapping_page | for each vma within range, clear page table entries within vma   
      |          |        +--------------------+                                                                  
      |          |                                                                                                
      |          |--> if page has private (private what?)                                                         
      |          |                                                                                                
      |          |        +-------------------+                                                                   
      |          +------> | do_invalidatepage | e.g., clear some flags for those truncated bh                     
      |                   +-------------------+                                                                   
      |    +------------------------+                                                                             
      +--> | delete_from_page_cache | detatch page from mapping and page ref--                                    
           +-----|------------------+                                                                             
                 |                                                                                                
                 |--> get page mapping                                                                            
                 |                                                                                                
                 |    +--------------------------+                                                                
                 |--> | __delete_from_page_cache | detatch page from mapping                                      
                 |    +--------------------------+                                                                
                 |    +----------------------+                                                                    
                 +--> | page_cache_free_page |                                                                    
                      +-----|----------------+                                                                    
                            |                                                                                     
                            |--> if mapping has ->freepage()                                                      
                            |                                                                                     
                            |------> call ->freepage()                                                            
                            |                                                                                     
                            |    +----------+                                                                     
                            +--> | put_page |                                                                     
                                 +----------+                                                                     
```

```
+--------------+                                                                                          
| sys_mprotect |                                                                                          
+---|----------+                                                                                          
    |    +------------------+                                                                             
    +--> | do_mprotect_pkey |                                                                             
         +----|-------------+                                                                             
              |    +----------+                                                                           
              |--> | find_vma | return  the first vma that meets 'addr < vm_end'                          
              |    +----------+                                                                           
              |                                                                                           
              |--> return if vma doesn't cover the start                                                  
              |                                                                                           
              |--> endless loop                                                                           
              |                                                                                           
              |------> if ->mprotect exists (only on x86?)                                                
              |                                                                                           
              +----------> call ->mprotect()                                                              
              |                                                                                           
              |        +----------------+                                                                 
              |------> | mprotect_fixup |                                                                 
              |        +---|------------+                                                                 
              |            |    +-----------+                                                             
              |            |--> | vma_merge | check if we can merge with prev/next vma with our new flags 
              |            |    +-----------+                                                             
              |            |                                                                              
              |            |--> if the specified range doens't match the whole vma                        
              |            |                                                                              
              |            |        +-----------+                                                         
              |            |------> | split_vma |                                                         
              |            |        +-----------+                                                         
              |            |                                                                              
              |            +--> change attributes in both sw and hw                                       
              |                                                                                           
              +--> break loop if we reach the end of range                                                
```

```
+-----------------------+                                                    
| flush_all_zero_pkmaps | release unused pages from pkmap                    
+-----|-----------------+                                                    
      |                                                                      
      |--> for each pkmap (512)                                              
      |                                                                      
      |------> if nothing we can do, or it's in use                          
      |                                                                      
      |----------> continue                                                  
      |                                                                      
      |------> pkmap_count[i] = 0                                            
      |                                                                      
      |        +----------+                                                  
      |------> | pte_page | get struct page from pte value (physical address)
      |        +----------+                                                  
      |        +-----------+                                                 
      |------> | pte_clear | clear pte                                       
      |        +-----------+                                                 
      |                                                                      
      +------> remove page from kmap list                                    
```

```
+----------+                                                                                                             
| ovl_mmap |                                                                                                             
+--|-------+                                                                                                             
   |                                                                                                                     
   |--> get read file from private data                                                                                  
   |                                                                                                                     
   |    +--------------+                                                                                                 
   +--> | vma_set_file | vma->vm_file = read_file                                                                        
        +--------------+                                                                                                 
        +-----------+                                                                                                    
        | call_mmap |                                                                                                    
        +--|--------+                                                                                                    
           |                                                                                                             
           +--> call ->mmap(), e.g.,                                                                                     
                +----------------------------+                                                                           
                | generic_file_readonly_mmap |                                                                           
                +------|---------------------+                                                                           
                       |    +-------------------+                                                                        
                       +--> | generic_file_mmap | vma->vm_ops = generic_file_vm_ops                                      
                            +-------------------+                                                                        
                                                                const struct vm_operations_struct generic_file_vm_ops = {
                                                                    .fault      = filemap_fault,                         
                                                                    .map_pages  = filemap_map_pages,                     
                                                                    .page_mkwrite   = filemap_page_mkwrite,              
                                                                };                                                       
```

```
+-------------------+                                                        
| filemap_map_pages |                                                        
+----|--------------+                                                        
     |    +----------------+                                                 
     |--> | first_map_page | get first page of mapping                       
     |    +----------------+                                                 
     |    +-----------------+                                                
     |--> | filemap_map_pmd |                                                
     |    +----|------------+                                                
     |         |                                                             
     |         |--> if pmd isn't set                                         
     |         |                                                             
     |         |        +--------------+                                     
     |         +------> | pmd_populate | have pmd point to the 2nd page table
     |                  +--------------+                                     
     |    +---------------------+                                            
     |--> | pte_offset_map_lock | get pte ptr based on addr                  
     |    +---------------------+                                            
     |                                                                       
     |--> for each page within range                                         
     |                                                                       
     |        +--------------+                                               
     |------> | find_subpage | ???                                           
     |        +--------------+                                               
     |                                                                       
     |------> update addr and offset                                         
     |                                                                       
     |        +------------+                                                 
     |------> | do_set_pte | handle rmap and set pte                         
     |        +------------+                                                 
     |    +---------------+                                                  
     +--> | next_map_page | get next page                                    
          +---------------+                                                  
```

```
+--------------------+                                                                   
| ondemand_readahead | : set up ra, ensure pages exist, and read data to them            
+----|---------------+                                                                   
     |                                                                                   
     |--> set up ra attributes (1: expected, 2: hit ra marker)                           
     |                                                                                   
     |    +------------------+                                                           
     +--> | do_page_cache_ra |                                                           
          +----|-------------+                                                           
               |    +-------------------------+                                          
               +--> | page_cache_ra_unbounded | ensure pages exist, and read data to them
                    +-------------------------+                                          
```

```
+-----------------------+                                                                       
| invalidate_inode_page | : detatch page from mapping                                           
+-----|-----------------+                                                                       
      |    +--------------------------+                                                         
      +--> | invalidate_complete_page |                                                         
           +------|-------------------+                                                         
                  |    +----------------+                                                       
                  +--> | remove_mapping |                                                       
                       +---|------------+                                                       
                           |    +------------------+                                            
                           +--> | __remove_mapping |                                            
                                +----|-------------+                                            
                                     |    +--------------------------+                          
                                     |--> | __delete_from_page_cache | detatch page from mapping
                                     |    +--------------------------+                          
                                     |                                                          
                                     |--> if mapping has ->freepage() (usually not)             
                                     |                                                          
                                     +------> call ->freepage()                                 
```

```
+--------------------------+                                                                                            
| invalidate_mapping_pages | : invalidate all clean pages in mapping                                                    
+------|-------------------+                                                                                            
       |    +----------------------------+                                                                              
       +--> | __invalidate_mapping_pages |                                                                              
            +------|---------------------+                                                                              
                   |                                                                                                    
                   |--> while we can still fill pvec                                                                    
                   |                                                                                                    
                   |------> for each page in pvec                                                                       
                   |                                                                                                    
                   |            +-----------------------+                                                               
                   |----------> | invalidate_inode_page | detatch page from mapping                                     
                   |            +-----------------------+                                                               
                   |                                                                                                    
                   |----------> if we invalidate nothing                                                                
                   |                                                                                                    
                   |                +----------------------+                                                            
                   |--------------> | deactivate_file_page | deactivate a file page (add it to pvec, flush them if full)
                   |                +----------------------+                                                            
                   |        +-----------------+                                                                         
                   +------> | pagevec_release | releases pages in pvec                                                  
                            +-----------------+                                                                         
```

```
+----------------------------+                                                                                                      
| truncate_inode_pages_final | : remove all pages from mapping and release them                                                     
+------|---------------------+                                                                                                      
       |                                                                                                                            
       |--> set 'exiting' in mapping flags                                                                                          
       |                                                                                                                            
       |    +----------------------+                                                                                                
       +--> | truncate_inode_pages |                                                                                                
            +-----|----------------+                                                                                                
                  |    +----------------------------+                                                                               
                  +--> | truncate_inode_pages_range | : for each page within range, remove it mapping and release it                
                       +------|---------------------+                                                                               
                              |                                                                                                     
                              |--> while we can still fill pvec                                                                     
                              |                                                                                                     
                              |------> for each page in pvec                                                                        
                              |                                                                                                     
                              |            +-----------------------+                                                                
                              |----------> | truncate_cleanup_page | unmap and invalidate page                                      
                              |            +-----------------------+                                                                
                              |        +------------------------------+                                                             
                              |------> | delete_from_page_cache_batch | for each page in pvec, remove it from mapping and release it
                              |        +------------------------------+                                                             
                              |        +-----------------+                                                                          
                              |------> | pagevec_release | release pages in pvec                                                    
                              |        +-----------------+                                                                          
                              |                                                                                                     
                              |--> if necessary, wait till all pages are gone                                                       
                              |                                                                                                     
                              |    +-----------------------------+                                                                  
                              +--> | cleancache_invalidate_inode | (disabled config)                                                
                                   +-----------------------------+                                                                  
```

```
+------------------------------+                                                               
| delete_from_page_cache_batch | : for each page in pvec, remove it from mapping and release it
+-------|----------------------+                                                               
        |    +-------------------------+                                                       
        |--> | page_cache_delete_batch | delete a bunch of pages from page cache               
        |    +-------------------------+                                                       
        |                                                                                      
        |--> for each page in pvec                                                             
        |                                                                                      
        |        +----------------------+                                                      
        +------> | page_cache_free_page |                                                      
                 +----------------------+                                                      
```

## <a name="reference"></a> Reference

(TBD)



