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
        |    +----------+                 +----------+                      +-----------+      +-----------+
syscall |    | sys_open |                 | sys_read |                      | sys_write |      | sys_close |
        |    +----------+                 +----------+                      +-----------+      +-----------+
        +--        |                            |                                 |                  |
        +--        v                            v                                 v                  v       worker
        |    +----------+                 +----------+                      +-----------+        +------+ +----------+
        |    | vfs_open |                 | vfs_read |                      | vfs_write |        | fput | | ____fput |
        |    +----------+                 +----------+                      +-----------+        +------+ +----------+
    vfs |          |                            |                                 |                             |
        |          v                            v                                 v                             v
        |   +-------------+           +------------------+              +-------------------+           +--------------+
        |   | blkdev_open |           | blkdev_read_iter |              | blkdev_write_iter |           | blkdev_close |
        |   +-------------+           +--+------------+--+              +-------------------+           +--------------+
        |          |                     | read_pages |                 | generic_writepages|               |
        |          |                     +------------+                 +-------------------+               |
        +--        |                           | |                               | |                        |
        +--        |                           v |                               v |                        |
address |          |        +------------------+ |            +------------------+ |                        | implicit
  space |          |        | blkdev_readahead | |            | blkdev_writepage | |                        | write
        |          |        +------------------+ |            +------------------+ |                        |
        +--        |              |              |                   |             |                        |
        +--        |              v              v                   v             v                        |
   block|          |        +------------++-----------------+ +------------++-----------------+             |
   layer|          |        | submit_bio || blk_finish_plug | | submit_bio || blk_finish_plug |             |
        |          |        +------------++-----------------+ +------------++-----------------+             |
        +--        |                               |                                 |                      |
        +--        v                               v                                 v                      v
        |  +---------------+            +-------------------+             +-------------------+    +------------------+
    mtd |  | mtdblock_open |            | mtdblock_readsect |             | mtdblock_writesect|    | mtdblock_release |
        |  +---------------+            +-------------------+             +-------------------+    +------------------+
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
    .readpage    = blkdev_readpage,                       |
    .readahead   = blkdev_readahead,                      |
    .writepage   = blkdev_writepage,                      | address
    .write_begin = blkdev_write_begin,                    | space
    .write_end   = blkdev_write_end,                      |
    .writepages  = blkdev_writepages,                     |
...                                                       |
};                                                      --+

blk_start_plug                                          --+ block
submit_bio                                                | layer
blk_finish_plug                                         --+

static struct mtd_blktrans_ops mtdblock_tr = {          --+
    .open      = mtdblock_open,                           |
    .release   = mtdblock_release,                        |
    .readsect  = mtdblock_readsect,                       | mtd
    .writesect = mtdblock_writesect,                      |
...                                                       |
};                                                      --+
```

IO operation is apparently much slower compared to CPU accessing data in memory. 
The Block layer assumes the role of caching IO requests altogether, and it delivers them in batch to the driver layer.

- blk_start_plug(): temporarily install a plug to the task, so the requests pile up instead of going through directly.
- submit_bio(): set up a corresponding **request** for **bio** and append it to the end of the plug list.
- blk_finish_plug(): splice the list into a local one and let the driver do the actual IO operation.

```
                           task                                                                          
                         +------+                                                                        
 blk_start_plug()        | plug------->  blk_plug                                                        
                         +------+       +---------+                                                      
                                        | mq_list |                                                      
                                        +---------+                                                      
                                                                                                         
                         --------------------------------------------------------------------------------
                                                                                                         
                           task                                                                          
                         +------+                                                                        
 submit_bio()            | plug------->  blk_plug          request           request           request   
                         +------+       +---------+     +-----------+     +-----------+     +-----------+
                                        | mq_list<------->queuelist<------->queuelist<------->queuelist |
                                        +---------+     +-----------+     +-----------+     +-----------+
                                                                                                         
                         --------------------------------------------------------------------------------
                                                                                                         
                           task                                                                          
                         +------+                                                                        
 blk_finish_plug()       | plug |        blk_plug          request           request           request   
                         +------+       +---------+     +-----------+     +-----------+     +-----------+
                                        | mq_list<------->queuelist<------->queuelist<------->queuelist |
                                        +---------+     +-----------+     +-----------+     +-----------+
```

<details>
  <summary> Code trace </summary>

```
+-----------------+                                                                                                         
| blk_finish_plug | : for each entity in list, add to a queue (e.g., io scheduler queue or mtd queue)
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
     |                     +------> | blk_mq_sched_insert_requests | : deliver rq to elevator or directly to the driver
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

</details>
         
```
 loop 	 8  
 nbd     16
 mtd      1
 spi-mtd  7
```

```
+-------------------+                                                                                                  
| blk_mq_alloc_disk | : preapre gendisk, and corresponding queue, bdi, and inode                                       
+----|--------------+                                                                                                  
     |    +---------------------+                                                                                      
     +--> | __blk_mq_alloc_disk |                                                                                      
          +-----|---------------+                                                                                      
                |    +------------------------+                                                                        
                |--> | blk_mq_init_queue_data |                                                                        
                |    +-----|------------------+                                                                        
                |          |    +-----------------+                                                                    
                |          |--> | blk_alloc_queue | allocate a queue and set up                                        
                |          |    +-----------------+                                                                    
                |          |    +-----------------------------+                                                        
                |          +--> | blk_mq_init_allocated_queue | install ops (e.g., mtd_mq_ops) and further set up queue
                |               +-----------------------------+                                                        
                |    +-------------------+                                                                             
                +--> | __alloc_disk_node | prepare gendisk, bdi, and inode                                             
                     +----|--------------+                                                                             
                          |                                                                                            
                          |--> allocate a gendisk                                                                      
                          |                                                                                            
                          |    +-----------+                                                                           
                          |--> | bdi_alloc | allocate bdi, set up wb and its dwork                                     
                          |    +-----------+                                                                           
                          |    +------------+                                                                          
                          |--> | bdev_alloc | allocate inode for the bdev                                              
                          |    +------------+                                                                          
                          |                                                                                            
                          +--> set up gendisk                                                                          
```

```
+-----------+                                                                                         
| bdi_alloc | ： allocate bdi, set up wb and its dwork                                                 
+--|--------+                                                                                         
   |                                                                                                  
   |--> allocate a bdi                                                                                
   |                                                                                                  
   |    +----------+                                                                                  
   +--> | bdi_init |                                                                                  
        +--|-------+                                                                                  
           |    +---------------+                                                                     
           +--> | cgwb_bdi_init |                                                                     
                +---|-----------+                                                                     
                    |    +---------+                                                                  
                    +--> | wb_init |                                                                  
                         +--|------+                                                                  
                            |                                                                         
                            |--> set up wb                                                            
                            |                         +-----------+                                   
                            +--> init dwork with fn = | wb_workfn |                                   
                                                      +-----------+                                   
                                                      rename worker, and wake it up to handle the work
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
    |    +-------------------+
    +--> | submit_bio_checks | add offset to sector of partion, so it becomes disk-wise sector
    |    +-------------------+
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

```
+-------------------+                                                                                           
| submit_bio_checks | : add offset to sector of partion, so it becomes disk-wise sector                         
+----|--------------+                                                                                           
     |    +-------------+                                                                                       
     |--> | blk_mq_plug | get current plug                                                                      
     |    +-------------+                                                                                       
     |                                                                                                          
     |--> if bio isn't set 'remapped'                                                                           
     |                                                                                                          
     |------> if bdev is a partition                                                                            
     |                                                                                                          
     |            +---------------------+                                                                       
     |----------> | blk_partition_remap | adjust the partition-wise offset to disk-wise offset, label 'remapped'
     |            +---------------------+                                                                       
     |                                                                                                          
     +--> perform a few other checks                                                                            
```


```
+-------------+
| blkdev_open | :
+---|---------+
    |    +-------------------+
    |--> | blkdev_get_by_dev | :
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

```c
struct bdev_inode {
    struct block_device bdev;   // all inodes that represent the block device are kept here
    struct inode vfs_inode;
};
```

```c
struct block_device {
    dev_t           bd_dev;                 // dev#
    int         bd_openers;                 // acount for how often the bdev is opened
    struct inode *      bd_inode;           // points to inode that represents the bdev
    struct gendisk *    bd_disk;            // points to the partition related layer
} __randomize_layout;
```

```c
truct gendisk {
    int major;                                  // self explained
    int first_minor;                            // self explained
    int minors;                                 // length of consecutive minor#
    char disk_name[DISK_NAME_LEN];              // disk name in /proc and /sys
    struct block_device *part0;                 // points to the first partition?
    const struct block_device_operations *fops; // low-level device operations
    struct request_queue *queue;                // request queue
    void *private_data;                         // private data of driver
};
```

```c
struct request_queue {
    struct elevator_queue   *elevator;  // points to io scheduler
    unsigned long       queue_flags;
    unsigned long       nr_requests;
    struct queue_limits limits;         // all kinds of limitations
};
```

```c
struct queue_limits {
    unsigned int        max_sectors;            // max sectors the device can handle in one request
    unsigned int        max_segment_size;       // max segment size in one request
};
```

```c
struct request {
    struct request_queue *q;            // points to the queue
    struct bio *bio;                    // points to the transferring bio
    struct bio *biotail;                // points to the last bio in list
    struct list_head queuelist;         // list node, but rarely used
    unsigned short nr_phys_segments;    // number of segments in a request
};
```

```c
struct bio {
    struct bio      *bi_next;       // node in singly linked list
    struct block_device *bi_bdev;   // points to the related block dev
    bio_end_io_t        *bi_end_io; // called when transfer completes, so block layer can, e.g., wake up sleeping task
    void            *bi_private;    // for driver-specific data
    unsigned short      bi_vcnt;    // entry# in io vector
    struct bio_vec      *bi_io_vec; // points to io vector
};
```

```c
struct bio_vec {
    struct page *bv_page;       // points to the page used for data transfer
    unsigned int    bv_len;     // specifies how many bytes in page are used if bv_offset is set
    unsigned int    bv_offset;  // offset within the page, usually 0
};
```

```c
struct task_struct {
    struct bio_list         *bio_list;
};
```

```c
struct elevator_mq_ops {
    int (*init_sched)(struct request_queue *, struct elevator_type *); 
    void (*exit_sched)(struct elevator_queue *); 
    int (*init_hctx)(struct blk_mq_hw_ctx *, unsigned int);
    void (*exit_hctx)(struct blk_mq_hw_ctx *, unsigned int);
    void (*depth_updated)(struct blk_mq_hw_ctx *); 

    bool (*allow_merge)(struct request_queue *, struct request *, struct bio *); 
    bool (*bio_merge)(struct request_queue *, struct bio *, unsigned int);
    int (*request_merge)(struct request_queue *q, struct request **, struct bio *); 
    void (*request_merged)(struct request_queue *, struct request *, enum elv_merge);
    void (*requests_merged)(struct request_queue *, struct request *, struct request *); 
    void (*limit_depth)(unsigned int, struct blk_mq_alloc_data *); 
    void (*prepare_request)(struct request *); 
    void (*finish_request)(struct request *); 
    void (*insert_requests)(struct blk_mq_hw_ctx *, struct list_head *, bool);
    struct request *(*dispatch_request)(struct blk_mq_hw_ctx *); 
    bool (*has_work)(struct blk_mq_hw_ctx *); 
    void (*completed_request)(struct request *, u64);
    void (*requeue_request)(struct request *); 
    struct request *(*former_request)(struct request_queue *, struct request *); 
    struct request *(*next_request)(struct request_queue *, struct request *); 
    void (*init_icq)(struct io_cq *); 
    void (*exit_icq)(struct io_cq *); 
};
```

```c
struct elevator_type
{
    struct kmem_cache *icq_cache;
    struct elevator_mq_ops ops;
    size_t icq_size;
    size_t icq_align;
    struct elv_fs_entry *elevator_attrs;    // attributes shown in sysfs
    const char *elevator_name;              // identifer for users to select
    const char *elevator_alias;
    const unsigned int elevator_features;
    struct module *elevator_owner;
    char icq_cache_name[ELV_NAME_MAX + 6];
    struct list_head list;                  // doubly linked list node
};
```

```
+----------+                                                                               
| add_disk | :
+--|-------+                                                                               
   |    +-----------------+                                                                
   +--> | device_add_disk | : add device, register bdi, and scan partitions
        +----|------------+                                                                
             |    +------------------+                                                     
             |--> | elevator_init_mq | (skip, there's no any elevator)                     
             |    +------------------+                                                     
             |    +------------+                                                           
             |--> | device_add |                                                           
             |    +------------+                                                           
             |    +--------------------+                                                   
             |--> | blk_register_queue | register the queue to sysfs, label it 'registered'
             |    +--------------------+                                                   
             |    +--------------+                                                         
             |--> | bdi_register | register bdi to bdi_tree and bdi_list                   
             |    +--------------+                                                         
             |    +----------------------+                                                 
             +--> | disk_scan_partitions |                                                 
                  +----------------------+                                                 
```

```
+---------------+                                                            
| add_partition | : prepare bdev_inode, decide dev#, set up and register dev,
+---|-----------+   insert partno to gendisk, add inode to table             
    |    +---------+                                                         
    |--> | xa_load | check if part_no is already in gendisk part_tbl         
    |    +---------+                                                         
    |                                                                        
    |--> return 'busy' if it's the case                                      
    |                                                                        
    |    +------------+                                                      
    |--> | bdev_alloc | prepare a bdev_inode                                 
    |    +------------+                                                      
    |                                                                        
    |--> save start/size info in bdev and inode separately                   
    |                                                                        
    |    +--------------+                                                    
    |--> | dev_set_name | set dev name                                       
    |    +--------------+                                                    
    |                                                                        
    |--> set up class/type/parent of dev                                     
    |                                                                        
    |    +-------+                                                           
    |--> | MKDEV | prepare dev#                                              
    |    +-------+                                                           
    |                                                                        
    |--> if arg 'info' is provided                                           
    |                                                                        
    |        +---------+                                                     
    |------> | kmemdup | duplicate one for for block dev                     
    |        +---------+                                                     
    |    +------------+                                                      
    |--> | device_add |                                                      
    |    +------------+                                                      
    |    +-----------+                                                       
    |--> | xa_insert | insert partno to gendisk part_tbl                     
    |    +-----------+                                                       
    |    +----------+                                                        
    +--> | bdev_add | add to inode table (able to be looked up)              
         +----------+                                                        
```

```
cat /proc/partitions
```

## <a name="reference"></a> Reference

(TBD)
