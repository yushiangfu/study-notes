> Study case: Linux version 5.15.0 on AST2500 emulation

## Index

- [Introduction](#introduction)
- [Files and DTS](#files-and-dts)
- [Control Flow](#control-flow)
- [Boot Up](#boot-up)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

(TBD)

## <a name="files-and-dts"></a> Files and DTS

SPI is a protocol like I2C that we use to communicate with devices within a computer system. 
The most significant differences from I2C are:

- One SPI controller can only access a few devices, and it's usually 2 to 3.
- The device type is always flash storage from my working experience, e.g., SPI NOR flash.

And the memory technology device (MTD) is a higher layer that abstracts the complicated SPI command sending stuff. 
Userspace applications can apply regular file system operations to the device file, like reading and writing. 
Here's the list of all the MTD device files, and we can see that there are 7 MTDs duplicated in three groups:

- character device
- character device (read-only)
- block device

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

Also, there's a folder containing link files with descriptive names, and each points to the corresponding character device file. 
Note that mtd1 to mtd5 represent the partition while mtd0 means the whole flash containing them. The mtd6 is another entire flash.

From the description in DTS, we can see that there are three SPI controllers in AST2500. 
The SPI protocol replaces the address concept in the I2C protocol with slave selection, which is a prerequisite action before accessing devices. 
Please refer to the below diagram to better know the relation between controller, chip, and mtd.

```
                                                                                +----  flash@0
                                                                                |  +------------+
                                                                                |  |            | ◄─── [mtd6]
                                                                                |  +------------+
                 +----  flash@0 ◄─── [chip]                    spi@1e630000 ----+
                 |  +------------+                                              |
                 |  |   u-boot   | ◄─── [mtd1] ─┐                               |
                 |  +------------+              │                               |
                 |  | u-boot-env | ◄─── [mtd2]  │                               +----  flash@1
                 |  +------------+              │                                     (disabled)
                 |  |   kernel   | ◄─── [mtd3]  │ ◄─── [mtd0]
[controller]     |  +------------+              │
      │          |  |    rofs    | ◄─── [mtd4]  │
      ▼          |  +------------+              │
spi@1e620000 ----+  |    rwfs    | ◄─── [mtd5] ─┘
                 |  +------------+
                 |
                 |
                 |----  flash@1                                                 +----  flash@0
                 |     (disabled)                                               |     (disabled)
                 |                                                              |
                 |                                                              |
                 +----  flash@2                                spi@1e631000 ----+
                       (disabled)                                (disabled)     |
                                                                                |
                                                                                |
                                                                                +----  flash@1
                                                                                      (disabled)
```

<details>
  <summary> DTS </summary>

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
    interrupts = <0x13>;               slave select
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

    flash@1 {
        reg = <0x01>;
        compatible = "jedec,spi-nor";
        spi-max-frequency = <0x2faf080>;
        status = "disabled";
    };

    flash@2 {
        reg = <0x02>;
        compatible = "jedec,spi-nor";
        spi-max-frequency = <0x2faf080>;
        status = "disabled";
    };
};

spi@1e630000 {
    reg = <0x1e630000 0xc4 0x30000000 0x8000000>;
    #address-cells = <0x01>;
    #size-cells = <0x00>;
    compatible = "aspeed,ast2500-spi";
    clocks = <0x02 0x19>;
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <0x03>;
    phandle = <0x1b>;

    flash@0 {
        reg = <0x00>;
        compatible = "jedec,spi-nor";
        spi-max-frequency = <0x5f5e100>;
        status = "okay";
        m25p,fast-read;
        label = "pnor";
    };

    flash@1 {
        reg = <0x01>;
        compatible = "jedec,spi-nor";
        spi-max-frequency = <0x2faf080>;
        status = "disabled";
    };
};

spi@1e631000 {
    reg = <0x1e631000 0xc4 0x38000000 0x8000000>;
    #address-cells = <0x01>;
    #size-cells = <0x00>;
    compatible = "aspeed,ast2500-spi";
    clocks = <0x02 0x19>;
    status = "disabled";

    flash@0 {
        reg = <0x00>;
        compatible = "jedec,spi-nor";
        spi-max-frequency = <0x2faf080>;
        status = "disabled";
    };

    flash@1 {
        reg = <0x01>;
        compatible = "jedec,spi-nor";
        spi-max-frequency = <0x2faf080>;
        status = "disabled";
    };
};
```
</details>
  
## <a name="control-flow"></a> Control Flow

The control flows between the character device and block device are distinct. 
But they eventually connect to the MTD layer through the different entry points and further deliver the data request to the SPI driver.
  
```
              +---------+                                
              | syscall |                                
              +---------+                                
                   |                                     
                   v                                     
              +---------+                                
              |   vfs   |                                
              |+---+---+|                                
 def_chr_fops ||chr|blk|| def_blk_fops                   
              ++---+---++                                
                 |   |                                   
                 |   +--+                                
                 |      v                                
                 |     +---------+                       
                 |     | address | def_blk_aops          
                 |     |  space  | (is block layer here?)
                 |     +---------+                       
                 |      |                                
                 |   +--+                                
                 v   v                                   
              ++---+---++                                
     mtd_fops ||chr|blk|| mtdblock_tr                    
              |+---+---+|                                
              |   mtd   |                                
              +---------+                                
                   |                                     
                   v                                     
                +-----+                                  
                | spi |                                  
                +-----+                                  
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
  <summary> Code trace of 'mtd_fops' </summary>
  
```
+--------------+                                                               
| init_mtdchar |                                                               
+---|----------+                                                               
    |    +-------------------+                                                 
    +--> | __register_chrdev | reserve devt# (major = 90, minor = 0 ~ 2^20 - 1)
         +-------------------+                                                 
```

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
static struct mtd_blktrans_ops mtdblock_tr = { 
    .open       = mtdblock_open,
    .release    = mtdblock_release,
    .readsect   = mtdblock_readsect,
    .writesect  = mtdblock_writesect,
...
};
```
  
<details>
  <summary> Code trace of 'mtdblock_tr' </summary>
  
```
 (defined by macro in drivers/mtd/mtdblock.c)
+------------------+
| mtdblock_tr_init | :
+----|-------------+
     |    +-----------------------+
     +--> | register_mtd_blktrans |
          +-----|-----------------+
                |
                |--> if the notifier isn't registered yet
                |
                |        +-------------------+
                |------> | register_mtd_user | register notifier and apply to existing mtd devices first
                |        +-------------------+
                |
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
                         | mtdblock_add_mtd | : prepare gendisk and queue
                         +----|-------------+
                              |    +----------------------+
                              +--> | add_mtd_blktrans_dev | : prepare gendisk and queue
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
                                         +--> | device_add_disk | add device, register bdi, and scan partitions
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

</details>
 
```
    mtd->_write = spi_nor_write;
    mtd->_erase = spi_nor_erase;
    mtd->_read = spi_nor_read;
...
```
  
<details>
  <summary> Code trace of SPI functions </summary>

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

</details>
  
## <a name="uboot"></a> U-Boot
  
```
env/sf.c                                                                                                                                    
+------------------------+                                                                                                                   
| spi_flash_probe_bus_cs |                                                                                                                   
+-|----------------------+                                                                                                                   
  |    +------------------------+                                                                                                            
  +--> | spi_flash_probe_bus_cs |                                                                                                            
       +-|----------------------+                                                                                                            
         |    +--------------------+                                                                                                         
         +--> | spi_get_bus_and_cs |                                                                                                         
              +-|------------------+                                                                                                         
                |    +--------------------------+                                                                                            
                +--> | uclass_get_device_by_seq | get bus dev (probe spi controller if not yet)                                              
                |    +--------------------------+                                                                                            
                |    +--------------+                                                                                                        
                +--> | device_probe | probe spi flash                                                                                        
                     +-|------------+                                                                                                        
                       |                                                                                                                     
                       |--> select pinctrl (another probe is triggered)                                                                      
                       |                                                                                                                     
                       +--> call ->probe(), e.g.,                                                                                            
                            +---------------------+                                                                                          
                            | spi_flash_std_probe |                                                                                          
                            +-|-------------------+                                                                                          
                              |    +-----------------------+                                                                                 
                              +--> | spi_flash_probe_slave |                                                                                 
                                   +-|---------------------+                                                                                 
                                     |    +---------------+   +----------------------+                                                       
                                     |--> | spi_claim_bus |...| aspeed_spi_claim_bus |                                                       
                                     |    +---------------+   +-|--------------------+                                                       
                                     |                          |    +-----------------------+                                               
                                     |                          +--> | aspeed_spi_flash_init | if flash hasn't been probed, return right away
                                     |                               +-----------------------+ (the flow changes after spi_nor_scan is run)  
                                     |    +--------------+                                                                                   
                                     +--> | spi_nor_scan |                                                                                   
                                          +-|------------+                                                                                   
                                            |                                                                                                
                                            |--> read flash id, init params, select cmd (, init flash if needed)                             
                                            |                                                                                                
                                            +--> SF: Detected $$ with page size $$, erase $$, total $$                                   
```
  
```
drivers/spi/spi-uclass.c                                                                                          
+---------------+   +----------------------+                                                                       
| spi_claim_bus |...| aspeed_spi_claim_bus |                                                                       
+---------------+   +-|--------------------+                                                                       
                      |    +-----------------------+                                                               
                      +--> | aspeed_spi_flash_init |                                                               
                           +-|---------------------+                                                               
                             |                                                                                     
                             |--> if flash has no name (not yet probed), return
                             |  
                             |--> if flash has been init, return (so each flash will be init at most once)         
                             |                                                                                     
                             |--> print CS0: init $ flags:$ size:$ page:$ sector:$ erase:$                   
                             |    print cmds [ erase:$ read=$ write:$ ] dummy:$, speed:$                
                             |                                                                                     
                             |    +----------------------------+                                                   
                             |--> | aspeed_g6_spi_hclk_divisor |
                             |    +-|--------------------------+                                                   
                             |      |                                                                              
                             |      +--> print hclk=$$ required=$$ h_div $$, divisor is $$ (mask $$) speed=$$
                             |                                                                                     
                             |--> init flash's iomode                                                              
                             |                                                                                     
                             |--> determine ce_ctrl_user and ce_ctrl_fread, set ce_ctrl_fread to hw reg 
                             |                                                                                     
                             |--> print CS0: USER mode $$ FREAD mode $$                                    
                             |                                                                                     
                             |    +-------------------------------+                                                
                             +--> | aspeed_spi_timing_calibration |
                                  +-|-----------------------------+                                                
                                    |    +----------------------------+                                            
                                    |--> | aspeed_g6_spi_hclk_divisor |
                                    |    +----------------------------+                                            
                                    |                                                                              
                                    +--> print cs: $$, freq: $$MHz                                         
```
  
```
drivers/core/root.c                                                                                      
+------------------+                                                                                      
| dm_init_and_scan | : save udevice 'root' in gd, bind each enabled dev of dt to driver                   
+-|----------------+                                                                                      
  |    +---------+                                                                                        
  |--> | dm_init | get udevice 'root' and save in gd                                                      
  |    +---------+                                                                                        
  |    +------------------+                                                                               
  |--> | dm_scan_platdata | (skip)                                                                        
  |    +------------------+                                                                               
  |    +----------------------+                                                                           
  |--> | dm_extended_scan_fdt | for each eanbled dev in dt: find target driver & prepare dev binding to it
  |    +----------------------+                                                                           
  |    +---------------+                                                                                  
  +--> | dm_scan_other | (skip)                                                                           
       +---------------+                                                                                  
```
  
```
drivers/core/root.c                                                     
+---------+                                                              
| dm_init | : get udevice 'root' and save in gd                          
+-|-------+            -                                                 
  |    +---------------------+                                           
  |--> | device_bind_by_name | given info (root info), get target udevice
  |    +---------------------+                                           
  |    +--------------+                                                  
  +--> | device_probe | (root driver has no ->probe() though)            
       +--------------+                                                  
```
  
```
drivers/core/root.c                                                                                 
+----------------------+                                                                             
| dm_extended_scan_fdt | : for each eanbled dev in dt: find target driver & prepare dev binding to it
+-|--------------------+                                                                             
  |    +-------------+                                                                               
  |--> | dm_scan_fdt | for each eanbled dev in dt: find target driver & prepare dev binding to it    
  |    +-------------+                                                                               
  |    +-------------------------+                                                                   
  |--> | dm_scan_fdt_ofnode_path | "/clocks"                                                         
  |    +-------------------------+                                                                   
  |    +-------------------------+                                                                   
  +--> | dm_scan_fdt_ofnode_path | "/firmware"                                                       
       +-------------------------+                                                                   
```
  
```
drivers/core/root.c                                                                                                      
+-------------+                                                                                                           
| dm_scan_fdt | : for each eanbled dev in dt: find target driver & prepare dev binding to it                              
+-|-----------+                                                                                                           
  |    +------------------+                                                                                               
  +--> | dm_scan_fdt_node | : for each eanbled dev in dt: find target driver & prepare dev binding to it                  
       +-|----------------+                                                                                               
         |                                                                                                                
         +--> while we can still get a valid node                                                                         
              |                                                                                                           
              |--> get dt node name                                                                                       
              |                                                                                                           
              |--> if it's 'chosen' or 'firmware', specially handle it and continue                                       
              |                                                                                                           
              |--> if it's not enabled, continue                                                                          
              |                                                                                                           
              |    +----------------+                                                                                     
              +--> | lists_bind_fdt | given dev compatible list, find target driver and prepare dev binding to that driver
                   +----------------+                                                                                     
```
  
```
drivers/core/lists.c                                                                                    
+----------------+                                                                                       
| lists_bind_fdt | : given dev compatible list, find target driver and prepare dev binding to that driver
+-|--------------+                                                                                       
  |                                                                                                      
  |--> get compatible list                                                                               
  |                                                                                                      
  +--> for each value in list                                                                            
       |                                                                                                 
       |--> for each compiled-in driver                                                                  
       |    |                                                                                            
       |    |    +-------------------------+                                                             
       |    |--> | driver_check_compatible | check if driver supports compatible value                   
       |    |    +-------------------------+                                                             
       |    |                                                                                            
       |    +--> if found, break                                                                         
       |                                                                                                 
       |--> if nothing found, continue                                                                   
       |                                                                                                 
       |    +------------------------------+                                                             
       |--> | device_bind_with_driver_data | prepare dev and bind to driver                              
       |    +------------------------------+                                                             
       |                                                                                                 
       +--> pass dev back through argument                                                               
```
  
```
cmd/sf.c                                                                           
+--------------------+                                                              
| do_spi_flash_probe | : given bus/cs, probe target dev, save flash struct in global
+-|------------------+                                                              
  |                                                                                 
  |--> prepare default bus/cs/speed/mode                                            
  |                                                                                 
  |--> parse bus/cs/speed/mode from argv                                            
  |                                                                                 
  |--> given bus/cs, remove existing one if it's found                              
  |                                                                                 
  |    +------------------------+                                                   
  |--> | spi_flash_probe_bus_cs | bind dev to driver, probe it if not yet active    
  |    +------------------------+                                                   
  |                                                                                 
  +--> get flash struct and save in global                                          
```
  
```
drivers/mtd/spi/sf-uclass.c                                                
+------------------------+                                                  
| spi_flash_probe_bus_cs | : bind dev to driver, probe it if not yet active 
+-|----------------------+                                                  
  |                                                                         
  |--> determine name                                                       
  |                                                                         
  |    +--------------------+                                               
  +--> | spi_get_bus_and_cs | bind dev to driver, probe it if not yet active
       +--------------------+                                               
```
  
```
drivers/spi/spi-uclass.c
+--------------------+
| spi_get_bus_and_cs | : bind dev to driver, probe it if not yet active
+-|------------------+
  |    +--------------------------+
  |--> | uclass_get_device_by_seq | given bus#, find bus dev
  |    +--------------------------+ (here it probes the spi controler, and hence it calls aspeed_spi_probe)
  |                                 (each spi controller is probed at most once)
  |    +----------------------+
  |--> | spi_find_chip_select | traverse all dev on bus to find the cs#-matched one
  |    +----------------------+
  |
  |--> if not found (expected)
  |    |
  |    |    +--------------------+
  |    |--> | device_bind_driver |
  |    |    +--------------------+
  |    |
  |    +--> set up plat-data (cs/max_hz/mode)
  |
  |--> if not yet active
  |    |
  |    |    +--------------+
  |    +--> | device_probe | label 'activated', call ->probe
  |         +--------------+ (here it probes the spi flash, and hence it calls spi_flash_std_probe)
  |                          (unlike spi controller, flash seems to be probed everytime)
  |    +--------------------+
  |--> | spi_set_speed_mode | call ->set_speed() & ->set_mode()
  |    +--------------------+
  |
  +--> return bus and dev through argumets
```
  
```
drivers/core/device.c                                                                                                     
+--------------+                                                                                                           
| device_probe | : label 'activated', call ->probe                                                                         
+-|------------+                                                                                                           
  |                                                                                                                        
  |--> if parent dev exists                                                                                                
  |    |                                                                                                                   
  |    |    +--------------+                                                                                               
  |    +--> | device_probe | (recursive)                                                                                   
  |         +--------------+                                                                                               
  |    +--------------------+                                                                                              
  |--> | uclass_resolve_seq | get a unique seq# for dev?                                                                   
  |    +--------------------+                                                                                              
  |                                                                                                                        
  |--> label 'activated'                                                                                                   
  |                                                                                                                        
  |--> select pinctrl state 'default'                                                                                      
  |                                                                                                                        
  |    +-------------------------+                                                                                         
  |--> | uclass_pre_probe_device | call uclass drvier's ->pre_probe() & ->child_pre_probe()                                
  |    +-------------------------+                                                                                         
  |                                                                                                                        
  |--> if parent dev has ->child_pre_probe(), call it                                                                      
  |                                                                                                                        
  |    +------------------+                                                                                                
  |--> | clk_set_defaults | (key of stability?)                                                                            
  |    +------------------+                                                                                                
  |                                                                                                                        
  |--> call ->probe(), e.g.,                                                                                               
  |    +---------------------+                                                                                             
  |    | spi_flash_std_probe | do calibration, identify id, determine params and init hw regs, select read/write/erase cmds
  |    +---------------------+                                                                                             
  |                                                                                                                        
  |    +--------------------------+                                                                                        
  |--> | uclass_post_probe_device | call uclass drvier's ->child_post_probe() & ->post_probe()                             
  |    +--------------------------+                                                                                        
  |                                                                                                                        
  +--> select pinctrl state 'default'                                                                                      
```
  
```
drivers/core/uclass.c                                                                
+-------------------------+                                                           
| uclass_pre_probe_device | : call uclass drvier's ->pre_probe() & ->child_pre_probe()
+-|-----------------------+                                                           
  |                                                                                   
  |--> if ->pre_probe() exists                                                        
  |    -                                                                              
  |    +--> call it, e.g.,                                                            
  |         +-----------------------+                                                 
  |         | wait for real example |                                                 
  |         +-----------------------+                                                 
  |                                                                                   
  +--> if ->child_pre_probe() exists                                                  
       -                                                                              
       +--> call it, e.g.,                                                            
            +---------------------+                                                   
            | spi_child_pre_probe | ste up 'slave' from plat-data (max_hz, mode, ...) 
            +---------------------+                                                   
```
  
```
drivers/mtd/spi/sf_probe.c                                                                                                    
+---------------------+                                                                                                        
| spi_flash_std_probe | : do calibration, identify id, determine params and init hw regs, select read/write/erase cmds         
+-|-------------------+                                                                                                        
  |    +-----------------------+                                                                                               
  +--> | spi_flash_probe_slave | : do calibration, identify id, determine params and init hw regs, select read/write/erase cmds
       +-|---------------------+                                                                                               
         |    +---------------+                                                                                                
         |--> | spi_claim_bus | given cs, determine read/write modes to init hw registers, do calibration                      
         |    +---------------+                                                                                                
         |    +--------------+                                                                                                 
         +--> | spi_nor_scan | identify id, determine params and init hw regs, select read/write/erase cmds                    
              +--------------+                                                                                                 
```
  
```
drivers/spi/spi-uclass.c                                                                                            
+---------------+                                                                                                    
| spi_claim_bus | : given cs, determine read/write modes to init hw registers, do calibration                        
+-|-------------+                                                                                                    
  |    +------------------+                                                                                          
  +--> | dm_spi_claim_bus | : given cs, determine read/write modes to init hw registers, do calibration              
       +-|----------------+                                                                                          
         |                                                                                                           
         |--> determine speed                                                                                        
         |                                                                                                           
         |--> if the target speed != slave speed                                                                     
         |    |                                                                                                      
         |    |    +--------------------+                                                                            
         |    +--> | spi_set_speed_mode | call ->set_speed() & ->set_mode()                                          
         |         +--------------------+                                                                            
         |                                                                                                           
         +--> if ->claim_bus() exists                                                                                
              -                                                                                                      
              +--> call it, e.g.,                                                                                    
                   +----------------------+                                                                          
                   | aspeed_spi_claim_bus | given cs, determine read/write modes to init hw registers, do calibration
                   +----------------------+                                                                          
```
  
```
drivers/spi/spi-uclass.c                                                                   
+--------------------+                                                                      
| spi_set_speed_mode | : call ->set_speed() & ->set_mode()                                  
+-|------------------+                                                                      
  |                                                                                         
  |--> if ->set_speed() exists                                                              
  |    -                                                                                    
  |    +--> call it, e.g.,                                                                  
  |         +----------------------+                                                        
  |         | aspeed_spi_set_speed | do nothing, ->claim_bus() will take care of it         
  |         +----------------------+                                                        
  |                                                                                         
  +--> if ->set_mode() exists                                                               
       -                                                                                    
       +--> call it, e.g.,                                                                  
            +---------------------+                                                         
            | aspeed_spi_set_mode | basically do nothing, ->claim_bus() will take care of it
            +---------------------+                                                         
```
  
```
drivers/spi/aspeed_spi.c                                                                           
+----------------------+                                                                            
| aspeed_spi_claim_bus | : given cs, determine read/write modes to init hw registers, do calibration
+-|--------------------+                                                                            
  |    +----------------------+                                                                     
  |--> | aspeed_spi_get_flash | given cs, get the corresponding flash struct                        
  |    +----------------------+                                                                     
  |    +-----------------------+                                                                    
  +--> | aspeed_spi_flash_init | determine read/write modes to init hw, do calibration, label 'init'
       +-----------------------+                                                                    
```
  
```
drivers/spi/aspeed_spi.c                                                                      
+-----------------------+                                                                      
| aspeed_spi_flash_init | : determine read/write modes to init hw, do calibration, label 'init'
+-|---------------------+                                                                      
  |                                                                                            
  |--> if labeld 'init', return                                                                
  |                                                                                            
  |--> given spi read opcode, determine flash read mode                                        
  |                                                                                            
  |--> given spi program opcode, determine flash write mode                                    
  |                                                                                            
  |--> determine value for control register                                                    
  |                                                                                            
  |--> write addr-width to hw reg                                                              
  |                                                                                            
  |--> write value to hw (control) reg                                                         
  |                                                                                            
  |    +------------------------------+                                                        
  |--> | aspeed_spi_flash_set_segment | set segment addr in hw reg                             
  |    +------------------------------+                                                        
  |    +-------------------------------+                                                       
  |--> | aspeed_spi_timing_calibration | (key of stability?)                                   
  |    +-------------------------------+                                                       
  |                                                                                            
  +--> label 'init'                                                                            
```
  
```
drivers/mtd/spi/spi-nor-core.c                                                                
+--------------+                                                                               
| spi_nor_scan | : identify id, determine params and init hw regs, select read/write/erase cmds
+-|------------+                                                                               
  |                                                                                            
  |--> prepare default protocols for read/write                                                
  |                                                                                            
  |    +-----------------+                                                                     
  |--> | spi_nor_read_id | read id from flash, return the corresponding info                   
  |    +-----------------+                                                                     
  |    +---------------------+                                                                 
  |--> | spi_nor_init_params | determine read & page-program parameters                        
  |    +---------------------+                                                                 
  |                                                                                            
  |--> set up mtd and install ops (spi_nor_erase/spi_nor_read/spi_nor_write)                   
  |                                                                                            
  |    +---------------+                                                                       
  |--> | spi_nor_setup | select commands for fast read, page program, and sector erase         
  |    +---------------+                                                                       
  |    +--------------+                                                                        
  +--> | spi_nor_init | enable quad if necessary, enable 4-byte-addr mode if necessary         
       +--------------+                                                                        
```
  
```
drivers/mtd/spi/spi-nor-core.c                                   
+---------------------+                                           
| spi_nor_init_params | : determine read & page-program parameters
+-|-------------------+                                           
  |                                                               
  |--> determine read & page-program settings                     
  |                                                               
  +--> if it has quad capability, install op if ncessary          
```
  
## <a name="boot-up"></a> Boot Up
  
During the kernel boot flow, the SPI driver sequentially probes SPI controllers and parses partitions from DTB. 
After creating the MTD representing the whole flash, it identifies the flash model and installs the SPI operation set.
The probing and parsing procedure generates the below kernel boot log.
    
```
                                      +--    [    0.815817] aspeed-smc 1e620000.spi: Using 50 MHz SPI frequency                    
                                      |      [    0.819803] aspeed-smc 1e620000.spi: n25q256a (32768 Kbytes)                       
                                      |      [    0.820383] aspeed-smc 1e620000.spi: CE0 window [ 0x20000000 - 0x22000000 ] 32MB   
+------------------+                  |      [    0.820860] aspeed-smc 1e620000.spi: CE1 window [ 0x22000000 - 0x2a000000 ] 128MB  
| aspeed_smc_probe |                  |      [    0.821194] aspeed-smc 1e620000.spi: read control register: 203b0045               
+----|-------------+                  |      [    1.579900] Freeing initrd memory: 1072K                                           
     |    +------------------------+  |      [    1.694391] 5 fixed-partitions partitions found on MTD device bmc                  
     +--> | aspeed_smc_setup_flash |  |      [    1.694692] Creating 5 MTD partitions on "bmc":                                    
          +------------------------+  |      [    1.695062] 0x000000000000-0x000000060000 : "u-boot"                               
                                      |      [    1.697822] 0x000000060000-0x000000080000 : "u-boot-env"                           
                                      |      [    1.699548] 0x000000080000-0x0000004c0000 : "kernel"                               
                                      |      [    1.701035] 0x0000004c0000-0x000001c00000 : "rofs"                                 
                                      +--    [    1.702559] 0x000001c00000-0x000002000000 : "rwfs"                                 
                                      +--    [    1.708837] aspeed-smc 1e630000.spi: Using 100 MHz SPI frequency                   
+------------------+                  |      [    1.710020] aspeed-smc 1e630000.spi: mx66l1g45g (131072 Kbytes)                    
| aspeed_smc_probe |                  |      [    1.710244] aspeed-smc 1e630000.spi: CE0 window resized to 120MB (AST2500 HW quirk)
+----|-------------+                  |      [    1.710686] aspeed-smc 1e630000.spi: CE0 window [ 0x30000000 - 0x37800000 ] 120MB  
     |    +------------------------+  |      [    1.711108] aspeed-smc 1e630000.spi: CE1 window [ 0x37800000 - 0x38000000 ] 8MB    
     +--> | aspeed_smc_setup_flash |  |      [    1.711402] aspeed-smc 1e630000.spi: CE0 window too small for chip 128MB           
          +------------------------+  |      [    1.711628] aspeed-smc 1e630000.spi: read control register: 203c0045               
                                      +--    [    1.713799] aspeed-smc 1e630000.spi: Calibration area too uniform, using low speed 
```
  
<details>
  <summary> Code trace of driver probe </summary>
  
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
    |--> | spi_nor_get_flash_info | get manufacturer info, which decides the sector size and num --> flash size
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
+----------------+                                                                     
| add_mtd_device | : register mtd dev, create its ro dev, and prepare gendisk and queue
+---|------------+                                                                     
    |    +-----------+                                                                 
    |--> | idr_alloc | get an unique id                                                
    |    +-----------+                                                                 
    |    +-----------------+                                                           
    |--> | device_register | registe the mtd device                                    
    |    +-----------------+                                                           
    |                                                                                  
    |--> set up mtd                                                                    
    |                                                                                  
    |    +---------------+                                                             
    |--> | mtd_nvmem_add | (skip)                                                      
    |    +---------------+                                                             
    |    +---------------+                                                             
    |--> | device_create | create a read-only device of the mtd                        
    |    +---------------+                                                             
    |                                                                                  
    |--> for each mtd notifier                                                         
    |                                                                                  
    +------> call ->add(), e.g.,                                                       
             +---------------------+                                                   
             | blktrans_notify_add | prepare gendisk and queue                         
             +-----|---------------+                                                   
                   |                                                                   
                   |--> for each blktrans_majors                                       
                   |                                                                   
                   +------> call ->add_mtd(), e.g.,                                    
                            +------------------+                                       
                            | mtdblock_add_mtd | prepare gendisk and queue             
                            +------------------+                                       
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
                                     | parse_fixed_partitions | allocate partition array,
                                     +-----|------------------+ and read info from each partition node
                                           |                                                                                   
                                           |--> allocate partition array                                                       
                                           |                                                                                   
                                           |--> for each partition node                                                        
                                           |                                                                                   
                                           |------> get addr & size info from node                                             
                                           |                                                                                   
                                           +------> get label from node and use it as partition name                           
```

Please note that by default the debug log won't display during boot time, and we need to utilize _dmesg_ to check it instead. 

- Related log
```
[    1.489785] aspeed-smc 1e620000.spi: Using 50 MHz SPI frequency
[    1.493323] aspeed-smc 1e620000.spi: n25q256a (32768 Kbytes)
[    1.493947] aspeed-smc 1e620000.spi: CE0 window [ 0x20000000 - 0x22000000 ] 32MB
[    1.494494] aspeed-smc 1e620000.spi: CE1 window [ 0x22000000 - 0x2a000000 ] 128MB
[    1.494842] aspeed-smc 1e620000.spi: read control register: 203b0045
[    1.728658] random: crng init done
[    1.923572] 5 fixed-partitions partitions found on MTD device bmc
[    1.923940] Creating 5 MTD partitions on "bmc":
[    1.924305] 0x000000000000-0x000000060000 : "u-boot"
[    1.927399] 0x000000060000-0x000000080000 : "u-boot-env"
[    1.930167] 0x000000080000-0x0000004c0000 : "kernel"
[    1.932337] 0x0000004c0000-0x000001c00000 : "rofs"
[    1.934627] 0x000001c00000-0x000002000000 : "rwfs"

▲ fmc: spi@1e620000/flash@0
------------------------------------------------------------------------------------------
▼ spi1: spi@1e630000/flash@0

[    1.941091] aspeed-smc 1e630000.spi: Using 100 MHz SPI frequency
[    1.942534] aspeed-smc 1e630000.spi: mx66l1g45g (131072 Kbytes)
[    1.942806] aspeed-smc 1e630000.spi: CE0 window resized to 120MB (AST2500 HW quirk)
[    1.943841] aspeed-smc 1e630000.spi: CE0 window [ 0x30000000 - 0x37800000 ] 120MB
[    1.944343] aspeed-smc 1e630000.spi: CE1 window [ 0x37800000 - 0x38000000 ] 8MB
[    1.944715] aspeed-smc 1e630000.spi: CE0 window too small for chip 128MB
[    1.945007] aspeed-smc 1e630000.spi: read control register: 203c0045
[    1.947181] aspeed-smc 1e630000.spi: Calibration area too uniform, using low speed

Code flow:
[aspeed_smc_probe]
    prepare controller
    get 1st MEM resource and map it as registers
    get 2nd MEM resource and map it as AHB base <- ?
    [aspeed_smc_setup_flash]
        for each flash connected to the controller (e.g. only 1)
            prepare chip
        
```
  
</details>

## <a name="reference"></a> Reference

(TBD)
