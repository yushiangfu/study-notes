> Study case: Linux version 5.15.0 on AST2500 emulation

## Index

- [Introduction](#introduction)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

```
                   +--------  flash@0                                                                    
                   |      +------------+                                                                 
                   |      |   u-boot   |                                                                 
                   |      +------------+                                                                 
                   |      | u-boot-env |                                                                 
                   |      +------------+                                                                 
                   |      |   kernel   |                         +--------  flash@0                      
                   |      +------------+                         |      +------------+                   
                   |      |    rofs    |                         |      |            |                   
                   |      +------------+                         |      +------------+                   
 spi@1e620000 -----+      |    rwfs    |       spi@1e630000 -----+                           spi@1e631000
                   |      +------------+                         |                             (disabled)
                   |                                             |                                       
                   |                                             |                                       
                   |--------  flash@1                            +--------  flash@1                      
                   |         (disabled)                                    (disabled)                    
                   |                                                                                     
                   |                                                                                     
                   +--------  flash@2                                                                    
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

## <a name="reference"></a> Reference

(TBD)
