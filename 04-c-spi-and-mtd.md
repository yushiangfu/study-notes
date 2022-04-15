> Study case: Linux version 5.15.0 on AST2500 emulation

## Index

- [Introduction](#introduction)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

We can access the NAND or NOR flash through the SPI interface of the chip, e.g., AST2500. 
And the memory technology device (MTD) abstracts the complicated command sending stuff into a layer that users can apply regular read and write.
Here's the list of all the MTD device files, and we can see that there are 7 MTD partitions duplicated in three groups:

- character device
- character device (read-only)
- block device

Also, there's a folder containing link files with descriptive names, and each points to the corresponding character device file. 
Note that mtd1 to mtd6 represent the partition while mtd0 means the whole flash.

```
root@romulus:~# ls -l /dev/mtd*
crw-------    1 root     root       90,   0 Mar 12 08:17 /dev/mtd0
crw-------    1 root     root       90,   1 Mar 12 08:17 /dev/mtd0ro
crw-------    1 root     root       90,   2 Mar 12 08:17 /dev/mtd1
crw-------    1 root     root       90,   3 Mar 12 08:17 /dev/mtd1ro
crw-------    1 root     root       90,   4 Mar 12 08:17 /dev/mtd2
crw-------    1 root     root       90,   5 Mar 12 08:17 /dev/mtd2ro
crw-------    1 root     root       90,   6 Mar 12 08:17 /dev/mtd3
crw-------    1 root     root       90,   7 Mar 12 08:17 /dev/mtd3ro
crw-------    1 root     root       90,   8 Mar 12 08:17 /dev/mtd4
crw-------    1 root     root       90,   9 Mar 12 08:17 /dev/mtd4ro
crw-------    1 root     root       90,  10 Mar 12 08:17 /dev/mtd5
crw-------    1 root     root       90,  11 Mar 12 08:17 /dev/mtd5ro
crw-------    1 root     root       90,  12 Mar 12 08:17 /dev/mtd6
crw-------    1 root     root       90,  13 Mar 12 08:17 /dev/mtd6ro
brw-rw----    1 root     disk       31,   0 Mar 12 08:17 /dev/mtdblock0
brw-rw----    1 root     disk       31,   1 Mar 12 08:17 /dev/mtdblock1
brw-rw----    1 root     disk       31,   2 Mar 12 08:17 /dev/mtdblock2
brw-rw----    1 root     disk       31,   3 Mar 12 08:17 /dev/mtdblock3
brw-rw----    1 root     disk       31,   4 Mar 12 08:17 /dev/mtdblock4
brw-rw----    1 root     disk       31,   5 Mar 12 08:17 /dev/mtdblock5
brw-rw----    1 root     disk       31,   6 Mar 12 08:17 /dev/mtdblock6

/dev/mtd:
lrwxrwxrwx    1 root     root             7 Mar 12 08:17 bmc -> ../mtd0
lrwxrwxrwx    1 root     root             7 Mar 12 08:17 kernel -> ../mtd3
lrwxrwxrwx    1 root     root             7 Mar 12 08:17 pnor -> ../mtd6
lrwxrwxrwx    1 root     root             7 Mar 12 08:17 rofs -> ../mtd4
lrwxrwxrwx    1 root     root             7 Mar 12 08:17 rwfs -> ../mtd5
lrwxrwxrwx    1 root     root             7 Mar 12 08:17 u-boot -> ../mtd1
lrwxrwxrwx    1 root     root             7 Mar 12 08:17 u-boot-env -> ../mtd2
```

```
                 +----  flash@0                        +----  flash@0                        +----  flash@0
                 |  +------------+                     |  +------------+                     |     (disabled)
                 |  |   u-boot   |                     |  |            |                     |
                 |  +------------+                     |  +------------+                     |
                 |  | u-boot-env |                     |                                     |
                 |  +------------+                     |                                     |
                 |  |   kernel   |                     |                                     |
                 |  +------------+                     |                                     |
                 |  |    rofs    |                     |                                     |
                 |  +------------+                     |                                     |
spi@1e620000 ----+  |    rwfs    |    spi@1e630000 ----+                    spi@1e631000 ----+
                 |  +------------+                     |                      (disabled)     |
                 |                                     |                                     |
                 |                                     |                                     |
                 |----  flash@1                        +----  flash@1                        +----  flash@1
                 |     (disabled)                            (disabled)                            (disabled)
                 |
                 |
                 +----  flash@2
                       (disabled)
```

```
            register            ahb
                |                |
                |                |
                +-------+        +-------+
                |       |        |       |
spi@1e620000 {  v       v        v       v
    reg = <0x1e620000 0xc4 0x20000000 0x10000000>;
    #address-cells = <0x01>;
    #size-cells = <0x00>;
    compatible = "aspeed,ast2500-fmc";
    clocks = <0x02 0x19>;
    status = "okay";
    interrupts = <0x13>;               chip select
                                            |
    flash@0 {                               |
        reg = <0x00>; <---------------------+
        compatible = "jedec,spi-nor";
        spi-max-frequency = <0x2faf080>;
        status = "okay";
        m25p,fast-read;
        label = "bmc";

        partitions {
            compatible = "fixed-partitions";
            #address-cells = <0x01>;
            #size-cells = <0x01>;


                    u-boot@0 {
                        reg = <0x00 0x60000>;
                        label = "u-boot";
                    };

                    u-boot-env@60000 {
                        reg = <0x60000 0x20000>;
                        label = "u-boot-env";
                    };

                    kernel@80000 {
                        reg = <0x80000 0x440000>;
                        label = "kernel";
                    };

                    rofs@c0000 {
                        reg = <0x4c0000 0x1740000>;
                        label = "rofs";
                    };

                    rwfs@1c00000 {
                        reg = <0x1c00000 0x400000>;
                        label = "rwfs";
                    };
                };
            };
```

```
                 +----  flash@0 ◄─── [chip]
                 |  +------------+
                 |  |   u-boot   | ◄─── [mtd] ─┐
                 |  +------------+             │
                 |  | u-boot-env | ◄─── [mtd]  │
                 |  +------------+             │
                 |  |   kernel   | ◄─── [mtd]  │ ◄─── [mtd]
[controller]     |  +------------+             │
      │          |  |    rofs    | ◄─── [mtd]  │
      ▼          |  +------------+             │
spi@1e620000 ----+  |    rwfs    | ◄─── [mtd] ─┘
                 |  +------------+
                 |
                 |
                 |----  flash@1
                 |     (disabled)
                 |
                 |
                 +----  flash@2
                       (disabled)
```

```
+------------------+
| aspeed_smc_probe |
+----|-------------+
     |    +-----------------+
     |--> | of_match_device | check if the driver and arg device match
     |    +-----------------+
     |
     |--> set up controller
     |
     |--> map register base & ahb base
     |
     |    +------------------------+
     +--> | aspeed_smc_setup_flash |
          +-----|------------------+
                |
                |--> for each flash descriptor
                |
                |------> install 'aspeed_smc_controller_ops'
                |
                |------> print log, e.g., "aspeed-smc 1e620000.spi: Using 50 MHz SPI frequency"
                |
                |------> prepare 'chip' struct
                |
                |        +----------------------------+
                |------> | aspeed_smc_chip_setup_init | initialize
                |        +----------------------------+
                |        +--------------+
                |------> | spi_nor_scan | (refer to the below)
                |        +--------------+
                |        +------------------------------+
                |------> | aspeed_smc_chip_setup_finish | determine write and read control value
                |        +------------------------------+
                |        +---------------------+
                +------> | mtd_device_register | (refer to the below)
                         +---------------------+                                            
```

```
+--------------+
| spi_nor_scan |
+---|----------+
    |
    |--> prepare page-sized bounce buffer
    |
    |    +------------------------+
    |--> | spi_nor_get_flash_info | get manufacturer info, which determines the sector size and sector num --> flash size
    |    +------------------------+
    |    +---------------------+
    |--> | spi_nor_init_params | initialize flash parameters
    |    +---------------------+
    |
    |--> set up 'mtd' struct and install ops
    |
    |    +---------------+
    |--> | spi_nor_setup | e.g., select commands for fast read, page program, and sector erase
    |    +---------------+
    |    +--------------+
    |--> | spi_nor_init | send commands to init the flash
    |    +--------------+
    |
    +--> print log, e.g., "aspeed-smc 1e620000.spi: n25q256a (32768 Kbytes)"
```

```
+---------------------+
| mtd_device_register |
+-----|---------------+
      |    +---------------------------+
      +--> | mtd_device_parse_register |
           +------|--------------------+
                  |    +----------------+
                  |--> | add_mtd_device | register the mtd as mtd device and nvmem device
                  |    +----------------+
                  |    +----------------------+
                  +--> | parse_mtd_partitions |
                       +-----|----------------+
                             |
                             |--> for each type ('cmdline' and 'of')
                             |
                             |------> if 'of' (our case)
                             |
                             |            +-------------------+
                             |----------> | mtd_part_of_parse | (refer to the below)
                             |            +-------------------+
                             |
                             |        +--------------------+
                             +------> | add_mtd_partitions |
                                      +----|---------------+
                                           |
                                           |--> for each partition
                                           |
                                           |        +--------------------+
                                           |------> | allocate_partition | allocate 'mtd_info' and copy info to it
                                           |        +--------------------+
                                           |        +----------------+
                                           +------> | add_mtd_device | register the mtd as mtd device and nvmem device
                                                    +----------------+
```

```
+-------------------+                                                                                                          
| mtd_part_of_parse | allocate partition array and read info from each partition node                                          
+----|--------------+                                                                                                          
     |                                                                                                                         
     |--> for each string in 'partition' property (only 'fixed-partitions' in our case)                                        
     |                                                                                                                         
     |        +--------------------------------+                                                                               
     +------> | mtd_part_get_compatible_parser | get parser based on string, e.g., 'fixed-partitions'                          
              +-------|------------------------+                                                                               
                      |    +-------------------+                                                                               
                      +--> | mtd_part_do_parse |                                                                               
                           +----|--------------+                                                                               
                                |                                                                                              
                                +--> call parser_fn(), e.g.,                                                                   
                                     +------------------------+                                                                
                                     | parse_fixed_partitions | allocate partition array and read info from each partition node
                                     +-----|------------------+                                                                
                                           |                                                                                   
                                           |--> allocate partition array                                                       
                                           |                                                                                   
                                           |--> for each partition node                                                        
                                           |                                                                                   
                                           |------> get addr & size info from node                                             
                                           |                                                                                   
                                           +------> get label from node and use it as partition name                           
```

```
+--------------+                                                               
| init_mtdchar |                                                               
+---|----------+                                                               
    |    +-------------------+                                                 
    +--> | __register_chrdev | reserve devt# (major = 90, minor = 0 ~ 2^20 - 1)
         +-------------------+                                                 
```

```
static const struct file_operations mtd_fops = {
    .read       = mtdchar_read,
    .write      = mtdchar_write,
    .open       = mtdchar_open,
    .release    = mtdchar_close,
...
};
```

<details>
  <summary> Code trace of VFS layer </summary>

```
+--------------+                                           
| mtdchar_open |                                           
+---|----------+                                           
    |    +----------------+                                
    |--> | get_mtd_device | get master mtd                 
    |    +----------------+                                
    |                                                      
    +--> prepare 'mtd_file_info' and assign to file private
```

```
+--------------+                                                                                
| mtdchar_read |                                                                                
+---|----------+                                                                                
    |    +-------------------+                                                                  
    |--> | mtd_kmalloc_up_to | allocate buffer                                                  
    |    +-------------------+                                                                  
    |                                                                                           
    +--> while we haven't read enough to meet user's requirement                                
    |                                                                                           
    |        +----------+                                                                       
    |------> | mtd_read |                                                                       
    |        +--|-------+                                                                       
    |           |    +--------------+                                                           
    |           +--> | mtd_read_oob |                                                           
    |                +---|----------+                                                           
    |                    |    +------------------+                                              
    |                    +--> | mtd_read_oob_std |                                              
    |                         +----|-------------+                                              
    |                              |    +--------------------+                                  
    |                              |--> | mtd_get_master_ofs | adjust local offset to global one
    |                              |    +--------------------+                                  
    |                              |                                                            
    |                              +--> call ->_read(), e.g.,                                   
    |                                   +--------------+                                        
    |                                   | spi_nor_read | (refer to the below)                   
    |                                   +--------------+                                        
    |        +--------------+                                                                   
    +------> | copy_to_user |                                                                   
    |        +--------------+                                                                   
    |                                                                                           
    +--> free the buffer                                                                        
```

```
+---------------+                                             
| mtdchar_write |                                             
+---|-----------+                                             
    |    +-------------------+                                
    |--> | mtd_kmalloc_up_to | allocate buffer                
    |    +-------------------+                                
    |                                                         
    |    while we hanv't read enough                          
    |                                                         
    |        +----------------+                               
    |------> | copy_from_user |                               
    |        +----------------+                               
    |        +-----------+                                    
    +------> | mtd_write |                                    
    |        +--|--------+                                    
    |           |    +---------------+                        
    |           +--> | mtd_write_oob |                        
    |                +---|-----------+                        
    |                    |    +-------------------+           
    |                    +--> | mtd_write_oob_std |           
    |                         +----|--------------+           
    |                              |                          
    |                              +--> call ->_write(), e.g.,
    |                                   +---------------+     
    |                                   | spi_nor_write | (refer to the below)
    |                                   +---------------+     
    |                                                         
    +--> free the buffer                                      
```
  
```
+---------------+
| mtdchar_close |
+---|-----------+
    |
    |--> if the file is opened with 'write' flag
    |
    |        +----------+
    |------> | mtd_sync | call ->_sync() if it exists (not our case)
    |        +----------+
    |
    +--> free mtd_file_info
```
  
</details>

```
    mtd->_write = spi_nor_write;
    mtd->_erase = spi_nor_erase;
    mtd->_read = spi_nor_read;
...
```

```
+--------------+                                                  
| spi_nor_read |                                                  
+---|----------+                                                  
    |                                                             
    |--> while we haven't read enough                             
    |                                                             
    |        +----------------------+                             
    |------> | spi_nor_convert_addr | do nothing in our case      
    |        +----------------------+                             
    |        +-------------------+                                
    |------> | spi_nor_read_data |                                
    |        +-------------------+                                
    |                                                             
    +------> call ->read(), e.g.,                                 
             +-----------------+                                  
             | aspeed_smc_read | (driver of aspeed spi controller)
             +-----------------+ read data and copy to buffer     
```

```
+---------------+                                                                 
| spi_nor_write |                                                                 
+---|-----------+                                                                 
    |                                                                             
    |--> while we haven't write enough                                            
    |                                                                             
    |        +----------------------+                                             
    +------> | spi_nor_convert_addr | do nothing in our case                      
             +----------------------+                                             
             +----------------------+                                             
             | spi_nor_write_enable |                                             
             +----------------------+                                             
             +--------------------+                                               
             | spi_nor_write_data |                                               
             +----|---------------+                                               
                  |                                                               
                  +--> call ->write(), e.g.,                                      
                       +-----------------------+                                  
                       | aspeed_smc_write_user | (driver of aspeed spi controller)
                       +-----------------------+ write buffer data                
```

```
+------------------+                                                                                      
| mtdblock_tr_init |                                                                                      
+----|-------------+                                                                                      
     |    +-----------------------+                                                                       
     +--> | register_mtd_blktrans |                                                                       
          +-----|-----------------+                                                                       
                |    +-----------------+                                                                  
                |--> | register_blkdev | register name to major_names[major]                              
                |    +-----------------+                                                                  
                |                                                                                         
                |--> add arg tr to 'blktrans_majors' list                                                 
                |                                                                                         
                |--> for each mtd                                                                         
                |                                                                                         
                +------> call ->add_mtd(), e.g.,                                                          
                         +------------------+                                                             
                         | mtdblock_add_mtd |                                                             
                         +----|-------------+                                                             
                              |    +----------------------+                                               
                              +--> | add_mtd_blktrans_dev |                                               
                                   +-----|----------------+                                               
                                         |                                                                
                                         |--> add mtd to tr                                               
                                         |                                                                
                                         |    +-------------------+                                       
                                         |--> | blk_mq_alloc_disk | allocate gendisk and queue            
                                         |    +-------------------+                                       
                                         |                                                                
                                         |--> set up gendisk (major, minor, fops = 'mtd_block_ops')         
                                         |                                                                
                                         |    +-----------------+                                         
                                         +--> | device_add_disk | add disk information to kernel list     
                                              +-----------------+                                         
```

```
 +-------------+                                             
 | blkdev_open |                                             
 +-------------+                                             
        -                                                    
        -                                                    
        -                                                    
+---------------+                                            
| blktrans_open |                                            
+---|-----------+                                            
    |                                                        
    |--> call tr->open(), e.g.,                              
    |    +---------------+                                   
    |    | mtdblock_open | (nothing important)               
    |    +---------------+                                   
    |                                                        
    |    +------------------+                                
    +--> | __get_mtd_device |                                
         +----|-------------+                                
              |                                              
              +--> call master->_get_device(), e.g.,         
                   +--------------------+                    
                   | spi_nor_get_device | (nothing important)
                   +--------------------+                    
```

```
+--------------+                                           
| mtd_queue_rq |                                           
+---|----------+                                           
    |    +-------------------+                             
    +--> | mtd_blktrans_work |                             
         +----|--------------+                             
              |    +---------------------+                 
              +--> | do_blktrans_request |                 
                   +-----|---------------+                 
                         |                                 
                         |--> switch option                
                         |                                 
                         |--> case READ                    
                         |                                 
                         |------> call ->readsect(), e.g., 
                         |        +-------------------+    
                         |        | mtdblock_readsect |    
                         |        +-------------------+    
                         |                                 
                         |    case WRITE                   
                         |                                 
                         +------> call ->writesect(), e.g.,
                                  +--------------------+   
                                  | mtdblock_writesect |   
                                  +--------------------+   
```

## <a name="reference"></a> Reference

(TBD)
