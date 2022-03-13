> Study case: Linux version 5.15.0 on AST2500 emulation

## Index

- [Introduction](#introduction)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

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
                |------> | spi_nor_scan | (trace it separately)
                |        +--------------+
                |        +------------------------------+
                |------> | aspeed_smc_chip_setup_finish | determine write and read control value
                |        +------------------------------+
                |        +---------------------+
                +------> | mtd_device_register | (trace it separately)
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
                             |----------> | mtd_part_of_parse | allocate partition array and read info from each partition node
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

## <a name="reference"></a> Reference

(TBD)