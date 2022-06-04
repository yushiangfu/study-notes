> Study case: Linux version 5.15.0 on AST2500 emulation

## Index

- [Introduction](#introduction)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

Block device is one of the seven file types supported by the virtual file system (VFS). 
The device file has its special file operations prepared, and they work as the portal to block related functions. 
The blkdev_read_iter() and blkdev_write_iter() interact with address space, which is the abstract of mapping. 
Address space is a radix tree structure containing many pages that represent data pieces of a file or block storage. 
The block device mapping goes through the block layer, and each read-write operation is wrapped as a request and cached in a queue appropriately. 
When the time is right, the stream of requests bursts to the next layer, e.g., MTD.

```
         +--                                                                                                       
         |      +----------+            +----------+              +-----------+         +-----------+              
 syscall |      | sys_open |            | sys_read |              | sys_write |         | sys_close |              
         |      +----------+            +----------+              +-----------+         +-----------+              
         +--          |                       |                         |                     |                    
         +--          v                       v                         v                     v       worker       
         |      +----------+            +----------+              +-----------+           +------+ +----------+    
         |      | vfs_open |            | vfs_read |              | vfs_write |           | fput | | ____fput |    
         |      +----------+            +----------+              +-----------+           +------+ +----------+    
     vfs |            |                       |                         |                                |         
         |            v                       v                         v                                v         
         |     +-------------+      +------------------+      +-------------------+              +--------------+  
         |     | blkdev_open |      | blkdev_read_iter |      | blkdev_write_iter |              | blkdev_close |  
         |     +-------------+      +------------------+      +-------------------+              +--------------+  
         +--          |                       |                         |                                |         
         +--          |                       v                         v                                |         
         |            |             +------------------+      +------------------+                       |         
 address |            |             | blkdev_readahead |      | blkdev_writepage |                       | implicit
   space |            |             ++-----------------+      ++-----------------+                       | write   
         |            |              | blk_finish_plug |       | blk_finish_plug |                       |         
         |            |              +-----------------+       +-----------------+                       |         
         +--          |                       |                         |                                |         
         +--          v                       v                         v                                v         
         |    +---------------+     +-------------------+     +-------------------+            +------------------+
     mtd |    | mtdblock_open |     | mtdblock_readsect |     | mtdblock_writesect|            | mtdblock_release |
         |    +---------------+     +-------------------+     +-------------------+            +------------------+
         +--    
```

```
const struct file_operations def_blk_fops = {           --+
    .open       = blkdev_open,                            |
    .release    = blkdev_close,                           |
    .read_iter  = blkdev_read_iter,                       | vfs
    .write_iter = blkdev_write_iter,                      |
...                                                       |
}                                                       --+

const struct address_space_operations def_blk_aops = {  --+
    .readpage   = blkdev_readpage,                        |
    .readahead  = blkdev_readahead,                       |
    .writepage  = blkdev_writepage,                       | address
    .write_begin    = blkdev_write_begin,                 | space
    .write_end  = blkdev_write_end,                       |
    .writepages = blkdev_writepages,                      |
...                                                       |
};                                                      --+

static struct mtd_blktrans_ops mtdblock_tr = {          --+
    .open       = mtdblock_open,                          |
    .release    = mtdblock_release,                       |
    .readsect   = mtdblock_readsect,                      | mtd
    .writesect  = mtdblock_writesect,                     |
...                                                       |
};                                                      --+
```

<details>
  <summary> Code trace of VFS layer </summary>

```
+-------------+
| blkdev_open |
+---|---------+
    |    +-------------------+
    |--> | blkdev_get_by_dev |
    |    +----|--------------+
    |         |    +--------------------+
    |         +--> | blkdev_get_no_open | get bdev by dev#
    |         |    +--------------------+
    |         |    +------------------+
    |         +--> | blkdev_get_whole |
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
| blkdev_read_iter |                                                            
+----|-------------+                                                            
     |    +------------------------+                                            
     +--> | generic_file_read_iter | read data from page of mapping
          +------------------------+                                            
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

</details>

<details>
  <summary> Code trace of address space layer </summary>

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
| blkdev_readpage | : 
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

```
+-----------------+                                                                                                         
| blk_finish_plug | for each entity in list, add to a queue (e.g., io scheduler queue or mtd queue)
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
     |                                 +-   +------> | blk_mq_try_issue_list_directly | (refer to the below)
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
+------------+                                                                                             
| submit_bio | : append bio to task or wrap as request and add to a queue
+--|---------+                                                                                             
   |    +-------------------+                                                                              
   +--> | submit_bio_noacct |                                                                              
        +----|--------------+                                                                              
             |                                                                                             
             |--> if bio_list of current task is in use                                                    
             |                                                                                             
             |        +--------------+                                                                     
             |------> | bio_list_add | append the bio to the bio_list                                      
             |        +--------------+                                                                     
             |                                                                                             
             +------> return                                                                               
             |                                                                                             
             |--> if gendisk doesn't have ->submit_bio()                                                   
             |                                                                                             
             |        +------------------------+      +--------------+                                     
             +------> | __submit_bio_noacct_mq | ---> | __submit_bio | prepare 'request' and add to a queue
             |        +------------------------+      +--------------+                                     
             |                                                                                             
             |------> return                                                                               
             |                                                                                             
             |    +---------------------+      +--------------+                                            
             +--> | __submit_bio_noacct | ---> | __submit_bio | prepare 'request' and add to a queue       
                  +---------------------+      +--------------+                                            
```

```
+--------------+
| __submit_bio | : prepare 'request' and add to a queue
+---|----------+
    |
    |--> if disk has ->submit_bio()
    |
    |        call ->submit_bio()
    |
    |        return
    |
    |    +-------------------+
    +--> | blk_mq_submit_bio |
         +----|--------------+
              |    +------------------------+
              |--> | __blk_mq_alloc_request | prepare 'request'
              |    +------------------------+
              |    +-------------+
              |--> | blk_mq_plug | get plug from the current task
              |    +-------------+
              |
              +--> add the 'request' to plug list or io scheduler queue
```

</details>
         
## <a name="reference"></a> Reference

(TBD)
