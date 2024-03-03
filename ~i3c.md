```
drivers/i3c/master/ast2600-i3c-master.c                                                                                                                  
+------------------+                                                                                                                                      
| aspeed_i3c_probe | : iomap regs, request isr, init work (handle newly found devs), register master/i2c_adapter, register clients                        
+-|----------------+                                                                                                                                      
  |                                                                                                                                                       
  |--> alloc master                                                                                                                                       
  |                                                                                                                                                       
  |    +---------------------------------+                                                                                                                
  |--> | syscon_regmap_lookup_by_phandle | get i3c-global regmap                                                                                          
  |    +---------------------------------+                                                                                                                
  |    +--------------------------------+                                                                                                                 
  |--> | devm_platform_ioremap_resource | iomap registers                                                                                                 
  |    +--------------------------------+                                                                                                                 
  |    +------------------+                                                                                                                               
  |--> | platform_get_irq |                                                                                                                               
  |    +------------------+                                                                                                                               
  |    +------------------+                                                                                                                               
  |--> | devm_request_irq | register isr                                                                                                                  
  |    +------------------+ +-------------------------------+                                                                                             
  |                         | aspeed_i3c_master_irq_handler | handle interrupt as a master or slave, clear reg                                            
  |                         +-------------------------------+                                                                                             
  |    +---------------------------------+                                                                                                                
  |--> | aspeed_i3c_master_timing_config | config timing                                                                                                  
  |    +---------------------------------+                                                                                                                
  |    +----------------------------------+                                                                                                               
  |--> | aspeed_i3c_master_init_group_dat | init group dat and write to reg                                                                               
  |    +----------------------------------+                                                                                                               
  |    +-----------+----------------------+                                                                                                               
  |--> | INIT_WORK | aspeed_i3c_master_hj | communiate with dev if required, dynamically assign addrs, register each found dev                            
  |    +-----------+----------------------+                                                                                                               
  |    +---------------------+                                                                                                                            
  +--> | i3c_master_register | setup master dev, collect board info, attach i2c/i3c devs, re-do daa, register master/i2c_adapter, register i2c/i3c clients
       +---------------------+                                                                                                                            
```

```
drivers/i3c/master.c                                                                                                                                
+---------------------+                                                                                                                              
| i3c_master_register | : setup master dev, collect board info, attach i2c/i3c devs, re-do daa, register master/i2c_adapter, register i2c/i3c clients
+-|-------------------+                                                                                                                              
  |                                                                                                                                                  
  |--> setup master dev                                                                                                                              
  |                                                                                                                                                  
  |    +--------------+                                                                                                                              
  |--> | i3c_bus_init | init sw struct                                                                                                               
  |    +--------------+                                                                                                                              
  |    +---------------------+                                                                                                                       
  |--> | of_populate_i3c_bus | get prop 'jdec-spd', for each child dt node: prepare board info & append to list in master                            
  |    +---------------------+                                                                                                                       
  |    +------------------+                                                                                                                          
  |--> | i3c_bus_set_mode | determine i2c/i3c rate                                                                                                   
  |    +------------------+                                                                                                                          
  |    +-----------------+                                                                                                                           
  |--> | alloc_workqueue |                                                                                                                           
  |    +-----------------+                                                                                                                           
  |    +---------------------+                                                                                                                       
  |--> | i3c_master_bus_init | attach i2c devices, init hw, reset all dynamic addrs, attach i3c devices, do dynamic addr assignment                  
  |    +---------------------+                                                                                                                       
  |    +------------+                                                                                                                                
  |--> | device_add | register dev to framework                                                                                                      
  |    +------------+                                                                                                                                
  |    +-----------------------------+                                                                                                               
  |--> | i3c_master_i2c_adapter_init | register i2c adapter, for each i2c_board info: register i2c_client                                            
  |    +-----------------------------+                                                                                                               
  |    +----------------------------------+                                                                                                          
  +--> | i3c_master_register_new_i3c_devs | for each desc in master bus, register dev                                                                
       +----------------------------------+                                                                                                          
```

```
drivers/i3c/master.c                                                                                                   
+---------------------+                                                                                                 
| of_populate_i3c_bus | : get prop 'jdec-spd', for each child dt node: prepare board info & append to list in master    
+-|-------------------+                                                                                                 
  |                                                                                                                     
  |--> get property "jdec-spd" and save in master                                                                       
  |                                                                                                                     
  +--> for each child node                                                                                              
       |                                                                                                                
       |    +-----------------------+                                                                                   
       +--> | of_i3c_master_add_dev | parse dt node, prepare i2c or i3c board info accordingly, append to list in master
            +-----------------------+                                                                                   
```

```
drivers/i3c/master.c                                                                                         
+-----------------------+                                                                                     
| of_i3c_master_add_dev | : parse dt node, prepare i2c or i3c board info accordingly, append to list in master
+-|---------------------+                                                                                     
  |                                                                                                           
  +--> read prop 'reg'                                                                                        
       |                                                                                                      
       |--> if reg[1] == 0 (i2c dev)                                                                          
       |    |                                                                                                 
       |    |    +---------------------------------+                                                          
       |    +--> | of_i3c_master_add_i2c_boardinfo | prepare i2c board info and append to list in master      
       |         +---------------------------------+                                                          
       |                                                                                                      
       +--> else (i3c dev)                                                                                    
            |                                                                                                 
            |    +---------------------------------+                                                          
            +--> | of_i3c_master_add_i3c_boardinfo | prepare i3c board info and append to list in master      
                 +---------------------------------+                                                          
```

```
drivers/i3c/master.c                                                                    
+---------------------------------+                                                      
| of_i3c_master_add_i2c_boardinfo | : prepare i2c board info and append to list in master
+-|-------------------------------+                                                      
  |                                                                                      
  |--> alloc board_info                                                                  
  |                                                                                      
  |    +-----------------------+                                                         
  |--> | of_i2c_get_board_info | parse dt node to setup arg info                         
  |    +-----------------------+                                                         
  |    +---------------+                                                                 
  +--> | list_add_tail | append info to i2c_info_list of master                          
       +---------------+                                                                 
```

```
drivers/i2c/i2c-core-of.c                                 
+-----------------------+                                  
| of_i2c_get_board_info | : parse dt node to setup arg info
+-|---------------------+                                  
  |                                                        
  |--> read prop 'reg' (addr)                              
  |                                                        
  +--> read other props and setup arg info                 
```

```
drivers/i3c/master.c                                                                        
+---------------------------------+                                                          
| of_i3c_master_add_i3c_boardinfo | : prepare i3c board info and append to list in master    
+-|-------------------------------+                                                          
  |                                                                                          
  |--> alloc board_info                                                                      
  |                                                                                          
  |--> if reg[0] (static addr)                                                               
  |    |                                                                                     
  |    |    +------------------------------+                                                 
  |    +--> | i3c_bus_get_addr_slot_status | given addr, get slot status ('free' is expected)
  |         +------------------------------+                                                 
  |                                                                                          
  |--> get prop "assigned-address" (dynamic addr)                                            
  |                                                                                          
  |--> if got                                                                                
  |    |                                                                                     
  |    |    +------------------------------+                                                 
  |    +--> | i3c_bus_get_addr_slot_status | given addr, get slot status ('free' is expected)
  |         +------------------------------+                                                 
  |                                                                                          
  |--> interpret peripheral id from reg[1] & reg[2]                                          
  |                                                                                          
  |--> get prop 'dcr' and 'bcr' and save                                                     
  |                                                                                          
  |    +---------------+                                                                     
  +--> | list_add_tail | append info to i3c_info_list of master                              
       +---------------+                                                                     
```

```
drivers/i3c/master.c                                                                                                                    
+---------------------+                                                                                                                  
| i3c_master_bus_init | : attach i2c devices, init hw, reset all dynamic addrs, attach i3c devices, do dynamic addr assignment           
+-|-------------------+                                                                                                                  
  |                                                                                                                                      
  |--> for each i2c board info in master                                                                                                 
  |    |                                                                                                                                 
  |    |    +------------------------------+                                                                                             
  |    |--> | i3c_bus_get_addr_slot_status | given addr, get slot status                                                                 
  |    |    +------------------------------+                                                                                             
  |    |    ('free' is expected)                                                                                                         
  |    |                                                                                                                                 
  |    |    +------------------------------+                                                                                             
  |    |--> | i3c_bus_set_addr_slot_status | given addr, set slot status = arg (i2c_dev)                                                 
  |    |    +------------------------------+                                                                                             
  |    |    +--------------------------+                                                                                                 
  |    |--> | i3c_master_alloc_i2c_dev | alloc i2c_dev                                                                                   
  |    |    +--------------------------+                                                                                                 
  |    |    +---------------------------+                                                                                                
  |    +--> | i3c_master_attach_i2c_dev | attach i2c dev to i3c master                                                                   
  |         +---------------------------+                                                                                                
  |                                                                                                                                      
  |--> call ->bus_init, e.g.,                                                                                                            
  |    +----------------------------+                                                                                                    
  |    | aspeed_i3c_master_bus_init | write regs to config role/clock/threshld/interrupt/addr, prep self i3c_dev & attatch, enable master
  |    +----------------------------+                                                                                                    
  |                                                                                                                                      
  |    +--------------------------+                                                                                                      
  |--> | i3c_master_rstdaa_locked | send cmd 'reset dynamic addr assignment' to dst (everyone)                                           
  |    +--------------------------+                                                                                                      
  |    +-------------------------+                                                                                                       
  |--> | i3c_master_disec_locked | send cmd 'disable event c?' to dst (everyone)                                                         
  |    +-------------------------+                                                                                                       
  |                                                                                                                                      
  |--> for each i3c board info in master                                                                                                 
  |    |                                                                                                                                 
  |    |    +------------------------------+                                                                                             
  |    |--> | i3c_bus_get_addr_slot_status | given addr, get its slot status (should be 'free')                                          
  |    |    +------------------------------+                                                                                             
  |    |    +------------------------------+                                                                                             
  |    |--> | i3c_bus_set_addr_slot_status | set status[addr] = i3c_dev                                                                  
  |    |    +------------------------------+                                                                                             
  |    |                                                                                                                                 
  |    +--> if curr board_info has specified static addr                                                                                 
  |         |                                                                                                                            
  |         |    +------------------------------+                                                                                        
  |         +--> | i3c_master_early_i3c_dev_add | alloc i3c_dev, attach to master, set static addr, retrieve dev info                    
  |              +------------------------------+                                                                                        
  |    +-------------------+                                                                                                             
  +--> | i3c_master_do_daa | communiate with dev if required, dynamically assign addrs, register each found dev                          
       +-------------------+                                                                                                             
```

```
drivers/i3c/master.c                                                                                      
+---------------------------+                                                                              
| i3c_master_attach_i2c_dev | : attach i2c dev to i3c master                                               
+-|-------------------------+                                                                              
  |                                                                                                        
  |--> if ->attach_i2c_dev exists                                                                          
  |    -                                                                                                   
  |    +--> call it, e.g.,                                                                                 
  |         +----------------------------------+                                                           
  |         | aspeed_i3c_master_attach_i2c_dev | set group dat, save dev_addr in master, save hw_idx in dev
  |         +----------------------------------+                                                           
  |                                                                                                        
  +--> append dev to i2c_list in master                                                                    
```

```
drivers/i3c/master/ast2600-i3c-master.c                                                                                            
+----------------------------+                                                                                                      
| aspeed_i3c_master_bus_init | : write regs to config role/clock/threshld/interrupt/addr, prep self i3c_dev & attatch, enable master
+-|--------------------------+                                                                                                      
  |    +----------------------------+                                                                                               
  |--> | aspeed_i3c_master_set_role | determine master/slave, write to reg                                                          
  |    +----------------------------+                                                                                               
  |                                                                                                                                 
  |--> given mode, write reg to configure i2c or i3c clock                                                                          
  |                                                                                                                                 
  |--> write reg to set threshold                                                                                                   
  |                                                                                                                                 
  |--> clear intrrupt regs                                                                                                          
  |                                                                                                                                 
  |--> get a free addr for master, write to reg                                                                                     
  |                                                                                                                                 
  |    +---------------------+                                                                                                      
  |--> | i3c_master_set_info | prepare i3c_dev with info, attach to master                                                          
  |    +---------------------+                                                                                                      
  |    +--------------------------+                                                                                                 
  +--> | aspeed_i3c_master_enable | write reg to enable master                                                                      
       +--------------------------+                                                                                                 
```

```
drivers/i3c/master.c                                                                      
+---------------------+                                                                    
| i3c_master_set_info | : prepare i3c_dev with info, attach to master                      
+-|-------------------+                                                                    
  |    +--------------------------+                                                        
  |--> | i3c_master_alloc_i3c_dev | alloc and setup i3c_dev (info, master, ...)            
  |    +--------------------------+                                                        
  |    +---------------------------+                                                       
  +--> | i3c_master_attach_i3c_dev | save dev addr in master, append dev to master i3c list
       +---------------------------+                                                       
```

```
drivers/i3c/master.c                                                         
+--------------------------+                                                  
| i3c_master_rstdaa_locked | : send cmd 'reset dynamic addr assignment' to dst
+-|------------------------+                                                  
  |    +------------------------------+                                       
  |--> | i3c_bus_get_addr_slot_status | given addr, get its slot status       
  |    +------------------------------+                                       
  |                                                                           
  +--> send cmd 'reset dynamic addr assignment' to dst                        
```

```
drivers/i3c/master.c                                                                                 
+------------------------------+                                                                      
| i3c_master_early_i3c_dev_add | : alloc i3c_dev, attach to master, set static addr, retrieve dev info
+-|----------------------------+                                                                      
  |    +--------------------------+                                                                   
  |--> | i3c_master_alloc_i3c_dev | alloc i3c_dev                                                     
  |    +--------------------------+                                                                   
  |    +---------------------------+                                                                  
  |--> | i3c_master_attach_i3c_dev | save dev addr in master, append dev to master i3c list           
  |    +---------------------------+                                                                  
  |    +---------------------------+                                                                  
  |--> | i3c_master_setdasa_locked | send cmd 'dynamic address static address' to dst                 
  |    +---------------------------+                                                                  
  |    +-----------------------------+                                                                
  |--> | i3c_master_reattach_i3c_dev | update slot status, set group dat, save addr in master         
  |    +-----------------------------+                                                                
  |    +------------------------------+                                                               
  +--> | i3c_master_retrieve_dev_info | send cmds to collect device config                            
       +------------------------------+                                                               
```

```
drivers/i3c/master.c                                                                               
+-----------------------------+                                                                     
| i3c_master_i2c_adapter_init | : register i2c adapter, for each i2c_board info: register i2c_client
+-|---------------------------+                                                                     
  |                                                                                                 
  |--> setup i2c_adapter (parent, algo, ...)                                                        
  |                                                                                                 
  |    +-----------------+                                                                          
  |--> | i2c_add_adapter | determine adapter id and register it                                     
  |    +-----------------+                                                                          
  |                                                                                                 
  +--> for each i2c_boardinfo in master                                                             
       |                                                                                            
       |    +---------------------------------+                                                     
       |--> | i3c_master_find_i2c_dev_by_addr | given addr, find i2c_dev from master                
       |    +---------------------------------+                                                     
       |    +-----------------------+                                                               
       +--> | i2c_new_client_device | prepare 'i2c client' and register device                      
            +-----------------------+ (which potentially triggers other i2c driver probe)           
```

```
drivers/i3c/master.c                                                                       
+---------------------------+                                                               
| i3c_master_attach_i3c_dev | : save dev addr in master, append dev to master i3c list      
+-|-------------------------+                                                               
  |    +--------------------------+                                                         
  |--> | i3c_master_get_i3c_addrs | save status in slot of addr                             
  |    +--------------------------+                                                         
  |                                                                                         
  |--> if ->attach_i3c_dev exists                                                           
  |    -                                                                                    
  |    +--> call it, e.g.,                                                                  
  |         +----------------------------------+                                            
  |         | aspeed_i3c_master_attach_i3c_dev | alloc dat/priv for dev, save addr in master
  |         +----------------------------------+                                            
  |    +---------------+                                                                    
  +--> | list_add_tail | append dev to master i3c list                                      
       +---------------+                                                                    
```

```
drivers/i3c/master/ast2600-i3c-master.c                                          
+----------------------------------+                                              
| aspeed_i3c_master_attach_i3c_dev | : alloc dat/priv for dev, save addr in master
+-|--------------------------------+                                              
  |    +---------------------------------+                                        
  |--> | aspeed_i3c_master_set_group_dat | alloc or release group dat resource    
  |    +---------------------------------+                                        
  |                                                                               
  |--> alloc dev_data                                                             
  |                                                                               
  |    +--------------------------------------+                                   
  |--> | aspeed_i3c_master_get_group_hw_index | get group hw index                
  |    +--------------------------------------+                                   
  |                                                                               
  |--> save addr in master                                                        
  |                                                                               
  +--> dev.priv = dev_data                                                        
```

```
drivers/i3c/master.c                                                               
+--------------------------+                                                        
| i3c_master_get_i3c_addrs | : save status in slot of addr                          
+-|------------------------+                                                        
  |                                                                                 
  |--> if dev has static addr                                                       
  |    |                                                                            
  |    |    +------------------------------+                                        
  |    |--> | i3c_bus_get_addr_slot_status | given addr, get slot status from master
  |    |    +------------------------------+                                        
  |    |                                                                            
  |    |--> (status is expected to be 'free')                                       
  |    |                                                                            
  |    |    +------------------------------+                                        
  |    +--> | i3c_bus_set_addr_slot_status | set status to slot of addr             
  |         +------------------------------+                                        
  |                                                                                 
  +--> if dev has dynamic addr                                                      
       |                                                                            
       |    +------------------------------+                                        
       |--> | i3c_bus_get_addr_slot_status | given addr, get slot status from master
       |    +------------------------------+                                        
       |                                                                            
       |--> (status is expected to be 'free')                                       
       |                                                                            
       |    +------------------------------+                                        
       +--> | i3c_bus_set_addr_slot_status | set status to slot of addr             
            +------------------------------+                                        
```

```
drivers/i3c/master/ast2600-i3c-master.c                                 
+---------------------------------+                                      
| aspeed_i3c_master_set_group_dat | : alloc or release group dat resource
+-|-------------------------------+                                      
  |                                                                      
  |--> get dev_grp from master                                           
  |                                                                      
  |--> given arg addr, calculate val                                     
  |                                                                      
  |--> if val                                                            
  |    |                                                                 
  |    |--> if dev_grp has no hw_idx yet                                 
  |    |                                                                 
  |    |             +--------------------------------+                  
  |    |--> hw_idx = | aspeed_i3c_master_get_free_pos |                  
  |    |             +--------------------------------+                  
  |    |                                                                 
  |    +--> write hw_idx to reg                                          
  |                                                                      
  +--> else                                                              
       -                                                                 
       +--> write reg to release dat resource                            
```

```
drivers/i3c/master.c                                 
+--------------------------+                          
| i3c_master_sethid_locked | : broadcast 'set hid' cmd
+-|------------------------+                          
  |    +-----------------------+                      
  |--> | i3c_ccc_cmd_dest_init | init cmd_dest        
  |    +-----------------------+                      
  |    +------------------+                           
  |--> | i3c_ccc_cmd_init | init cmd_cmd              
  |    +------------------+                           
  |    +--------------------------------+             
  |--> | i3c_master_send_ccc_cmd_locked | send ccc cmd
  |    +--------------------------------+             
  |    +--------------------------+                   
  +--> | i3c_ccc_cmd_dest_cleanup | free payload      
       +--------------------------+                   
```

```
drivers/i3c/master.c                                                       
+--------------------------------+                                          
| i3c_master_send_ccc_cmd_locked | : send ccc cmd                           
+-|------------------------------+                                          
  |                                                                         
  |--> if ->supports_ccc_cmd exists                                         
  |    -                                                                    
  |    +--> call it, e.g.,                                                  
  |         +------------------------------------+                          
  |         | aspeed_i3c_master_supports_ccc_cmd | check if cmd is supported
  |         +------------------------------------+                          
  |                                                                         
  +--> call ->send_ccc_cmd, e.g.,                                           
       +--------------------------------+                                   
       | aspeed_i3c_master_send_ccc_cmd | send ccc cmd                      
       +--------------------------------+                                   
```

```
drivers/i3c/master/ast2600-i3c-master.c                       
+--------------------------------+                             
| aspeed_i3c_master_send_ccc_cmd | : send ccc cmd              
+-|------------------------------+                             
  |                                                            
  |--> if read_no_write                                        
  |    |                                                       
  |    |    +--------------------+                             
  |    +--> | aspeed_i3c_ccc_get | sync hardware and perform tx
  |         +--------------------+                             
  |                                                            
  +--> else                                                    
       |                                                       
       |    +--------------------+                             
       +--> | aspeed_i3c_ccc_set | sync hardware and perform tx
            +--------------------+ (rx buf is filled elsewhere)
```

```
drivers/i3c/master/ast2600-i3c-master.c                                     
+--------------------+                                                       
| aspeed_i3c_ccc_get | : sync hardware and perform tx                        
+-|------------------+                                                       
  |    +-------------------------------+                                     
  |--> | aspeed_i3c_master_sync_hw_dat | write reg to sync hardware data     
  |    +-------------------------------+                                     
  |    +------------------------------+                                      
  |--> | aspeed_i3c_master_alloc_xfer | alloc xfer                           
  |    +------------------------------+                                      
  |                                                                          
  |--> setup cmd                                                             
  |                                                                          
  |    +--------------------------------+                                    
  |--> | aspeed_i3c_master_enqueue_xfer | queue transfer and send out thru hw
  |    +--------------------------------+                                    
  |    +-----------------------------+                                       
  |--> | wait_for_completion_timeout | wait for completion, or timeout       
  |    +-----------------------------+                                       
  |    +-----------------------------+                                       
  +--> | aspeed_i3c_master_free_xfer | free xfer                             
       +-----------------------------+                                       
```

```
drivers/i3c/master/ast2600-i3c-master.c                                                   
+--------------------------------+                                                         
| aspeed_i3c_master_enqueue_xfer | : queue transfer and send out thru hw                   
+-|------------------------------+                                                         
  |                                                                                        
  |--> init completion                                                                     
  |                                                                                        
  +--> if there's cur xfer                                                                 
  |    |                                                                                   
  |    |    +---------------+                                                              
  |    +--> | list_add_tail | append arg xfer to queue                                     
  |         +---------------+                                                              
  |                                                                                        
  +--> else                                                                                
       |                                                                                   
       |                                                                                   
       |--> cur xfer = arg xfer                                                            
       |                                                                                   
       |    +-------------------------------------+                                        
       +--> | aspeed_i3c_master_start_xfer_locked | for each cmd in xfer, write tx_data/cmd
            +-------------------------------------+                                        
```

```
drivers/i3c/master/ast2600-i3c-master.c                                         
+-------------------------------------+                                          
| aspeed_i3c_master_start_xfer_locked | : for each cmd in xfer, write tx_data/cmd
+-|-----------------------------------+                                          
  |                                                                              
  |--> for each cmd in xfer                                                      
  |    |                                                                         
  |    |    +------------------------------+                                     
  |    +--> | aspeed_i3c_master_wr_tx_fifo | write tx data to reg                
  |         +------------------------------+                                     
  |                                                                              
  +--> for each cmd in xfer                                                      
       |                                                                         
       |                                                                         
       +--> write cmd to reg                                                     
```

```
drivers/i3c/master/ast2600-i3c-master.c                                                                     
+-------------------------------+                                                                            
| aspeed_i3c_master_sir_handler | : get a free slot, copy ibi status/data to it, queue work to handle payload
+-|-----------------------------+                                                                            
  |                                                                                                          
  |--> get ibi addr/len from arg status                                                                      
  |                                                                                                          
  |--> given addr, get its dev                                                                               
  |                                                                                                          
  |    +-------------------------------+                                                                     
  |--> | i3c_generic_ibi_get_free_slot | get a free slot from master priv                                    
  |    +-------------------------------+                                                                     
  |                                                                                                          
  |--> copy ibi status to slot buf                                                                           
  |                                                                                                          
  |    +---------------------------------+                                                                   
  |--> | aspeed_i3c_master_read_ibi_fifo | append ibi data to slot buf                                       
  |    +---------------------------------+                                                                   
  |    +----------------------+                                                                              
  +--> | i3c_master_queue_ibi | queue the slot to work queue                                                 
       +----------------------+ +-----------------------+                                                    
                                | i3c_master_handle_ibi | run callback on payload, return slot to pool       
                                +-----------------------+                                                    
```

```
drivers/i3c/master.c                                                   
+-----------------------+                                               
| i3c_master_handle_ibi | : run callback on payload, return slot to pool
+-|---------------------+                                               
  |                                                                     
  |--> given work, get outer slot                                       
  |                                                                     
  |--> get payload buf/len from slot                                    
  |                                                                     
  |--> if ibi source is a i3c device                                    
  |    |                                                                
  |    |    +-------------------------+                                 
  |    +--> | i3c_ibi_mqueue_callback |                                 
  |         +-------------------------+                                 
  |                                                                     
  |--> call ->recycle_ibi_slot(), e.g.,                                 
  |    +------------------------------------+                           
  |    | aspeed_i3c_master_recycle_ibi_slot | add slot back to pool     
  |    +------------------------------------+                           
  |                                                                     
  +--> if ibi has no pending bits                                       
       |                                                                
       |    +----------+                                                
       +--> | complete |                                                
            +----------+                                                
```

```
drivers/i3c/master/ast2600-i3c-master.c                                              
+-------------------------------+                                                     
| aspeed_i3c_slave_resp_handler | : for each response, call slave callback or complete
+-|-----------------------------+                                                     
  |                                                                                   
  |--> read response info from register                                               
  |                                                                                   
  +--> for each response                                                              
       |                                                                              
       |--> read response info from another register                                  
       |                                                                              
       |--> if no error & there's something to read                                   
       |    |                                                                         
       |    |    +--------------------------------+                                   
       |    |--> | aspeed_i3c_master_read_rx_fifo | memcpy data from register         
       |    |    +--------------------------------+                                   
       |    |                                                                         
       |    +--> if slave has registered callback                                     
       |         -                                                                    
       |         +--> call it, e.g.,                                                  
       |              +---------------------------+                                   
       |              | i3c_slave_eeprom_callback |                                   
       |              +---------------------------+                                   
       |                                                                              
       +--> if no error & nothing to read                                             
            |                                                                         
            |    +----------+                                                         
            +--> | complete |                                                         
                 +----------+                                                         
```

```
drivers/i3c/master/ast2600-i3c-master.c                                                                                     
+-------------------------------+                                                                                            
| aspeed_i3c_master_irq_handler | : handle interrupt as a master or slave, clear reg                                         
+-|-----------------------------+                                                                                            
  |                                                                                                                          
  |--> read status from register                                                                                             
  |                                                                                                                          
  |--> if we are secondary master                                                                                            
  |    |                                                                                                                     
  |    |--> if response ready                                                                                                
  |    |    |                                                                                                                
  |    |    |    +-------------------------------+                                                                           
  |    |    +--> | aspeed_i3c_slave_resp_handler | for each response, call slave callback or complete                        
  |    |         +-------------------------------+                                                                           
  |    |                                                                                                                     
  |    +--> if ccc updated                                                                                                   
  |         |                                                                                                                
  |         |    +--------------------------------+                                                                          
  |         +--> | aspeed_i3c_slave_event_handler | read event from reg, write to another reg                                
  |              +--------------------------------+                                                                          
  |                                                                                                                          
  |--> else                                                                                                                  
  |    |                                                                                                                     
  |    |--> if ibi threshold                                                                                                 
  |    |    |                                                                                                                
  |    |    |    +------------------------------+                                                                            
  |    |    +--> | aspeed_i3c_master_demux_ibis | handle each ibi (e.g., save statue in slott, or assign addr & register dev)
  |    |         +------------------------------+                                                                            
  |    |                                                                                                                     
  |    +--> if the master bus is in halt state                                                                               
  |         |                                                                                                                
  |         |    +------------------------------+                                                                            
  |         |--> | aspeed_i3c_master_reset_ctrl |                                                                            
  |         |    +------------------------------+                                                                            
  |         |    +--------------------------+                                                                                
  |         +--> | aspeed_i3c_master_resume |                                                                                
  |              +--------------------------+                                                                                
  |                                                                                                                          
  +--> write status to reg to clear                                                                                          
```

```
drivers/i3c/master/ast2600-i3c-master.c                                                                                                   
+------------------------------+                                                                                                           
| aspeed_i3c_master_demux_ibis | : handle each ibi (e.g., save statue in slott, or assign addr & register dev)                             
+-|----------------------------+                                                                                                           
  |                                                                                                                                        
  |--> read ibi info from register                                                                                                         
  |                                                                                                                                        
  +--> for each ibi                                                                                                                        
       |                                                                                                                                   
       |--> read queue status, and get ibi addr from it                                                                                    
       |                                                                                                                                   
       |--> if 'sir' type                                                                                                                  
       |    |                                                                                                                              
       |    |    +-------------------------------+                                                                                         
       |    +--> | aspeed_i3c_master_sir_handler | get a free slot, copy ibi status/data to it, queue work to handle payload               
       |         +-------------------------------+                                                                                         
       |                                                                                                                                   
       +--> if 'high-speed job' type                                                                                                       
            |                                                                                                                              
            |    +------------+                                                                                                            
            +--> | queue_work | add work to queue                                                                                          
                 +------------+ +----------------------+                                                                                   
                                | aspeed_i3c_master_hj | communiate with dev if required, dynamically assign addrs, register each found dev
                                +----------------------+                                                                                   
```

```
drivers/i3c/master.c                                                                                                        
+----------------------+                                                                                                     
| aspeed_i3c_master_hj | : communiate with dev if required, dynamically assign addrs, register each found dev                
+-------------------+--+                                                                                                     
| i3c_master_do_daa | : communiate with dev if required, dynamically assign addrs, register each found dev                   
+-------------------+                                                                                                        
  |                                                                                                                          
  |--> if master has jdec_spd                                                                                                
  |    |                                                                                                                     
  |    |    +--------------------------+                                                                                     
  |    |--> | i3c_master_sethid_locked | broadcast 'set hid' cmd                                                             
  |    |    +--------------------------+                                                                                     
  |    |    +---------------------------+                                                                                    
  |    +--> | i3c_master_setaasa_locked | broadcast 'set aasa' cmd                                                           
  |         +---------------------------+                                                                                    
  |                                                                                                                          
  |--> else                                                                                                                  
  |    -                                                                                                                     
  |    +--> call ->do_daa(), e.g.,                                                                                           
  |         +-----------------------+                                                                                        
  |         | aspeed_i3c_master_daa | send cmd 'addr assign' & get response, set group dat and alloc i3c_dev for each set bit
  |         +-----------------------+                                                                                        
  |    +----------------------------------+                                                                                  
  +--> | i3c_master_register_new_i3c_devs | for each desc in master bus, register dev                                        
       +----------------------------------+                                                                                  
```

```
drivers/i3c/master.c                                                           
+----------------------------------+                                            
| i3c_master_register_new_i3c_devs | : for each desc in master bus, register dev
+-|--------------------------------+                                            
  |                                                                             
  +--> for each desc in master bus                                              
       |                                                                        
       |--> alloc i3c_dev                                                       
       |                                                                        
       |--> setup dev (type, bus, ...)                                          
       |                                                                        
       |    +-----------------+                                                 
       +--> | device_register |                                                 
            +-----------------+                                                 
```

```
drivers/i3c/master/ast2600-i3c-master.c                                                                                     
+-----------------------+                                                                                                    
| aspeed_i3c_master_daa | send cmd 'addr assign' & get response, set group dat and alloc i3c_dev for each set bit
+-|---------------------+                                                                                                    
  |                                                                                                                          
  |--> for each pos in master                                                                                                
  |    |                                                                                                                     
  |    |    +--------------------------+                                                                                     
  |    |--> | i3c_master_get_free_addr |                                                                                     
  |    |    +--------------------------+                                                                                     
  |    |                                                                                                                     
  |    +--> save addr in master, and write to reg                                                                            
  |                                                                                                                          
  |    +------------------------------+                                                                                      
  |--> | aspeed_i3c_master_alloc_xfer | alloc xfer                                                                           
  |    +------------------------------+                                                                                      
  |    +--------------------------------+                                                                                    
  |--> | aspeed_i3c_master_get_free_pos | get first free pos in master                                                       
  |    +--------------------------------+                                                                                    
  |                                                                                                                          
  |--> setup cmd[0] in transfer                                                                                              
  |                                                                                                                          
  |    +--------------------------------+                                                                                    
  |--> | aspeed_i3c_master_enqueue_xfer | queue transfer and send out thru hw                                                
  |    +--------------------------------+                                                                                    
  |                                                                                                                          
  |--> wait for completion                                                                                                   
  |                                                                                                                          
  |--> calculate newdev (?)                                                                                                  
  |                                                                                                                          
  |--> for each pos in master                                                                                                
  |    |                                                                                                                     
  |    |--> if it's for newdev                                                                                               
  |    |    |                                                                                                                
  |    |    |    +---------------------------------+                                                                         
  |    |    |--> | aspeed_i3c_master_set_group_dat | alloc or release group dat resource                                     
  |    |    |    +---------------------------------+                                                                         
  |    |    |    +-------------------------------+                                                                           
  |    |    +--> | i3c_master_add_i3c_dev_locked | alloc i3c_dev, attach to master, send cmds to collect dev info, handle ibi
  |    |         +-------------------------------+                                                                           
  |    |                                                                                                                     
  |    +--> write reg to clean up free hw dat (becomes occupied?)                                                            
  |                                                                                                                          
  +--> free xfer                                                                                                             
```

```
drivers/i3c/master/ast2600-i3c-master.c                                                                      
+-------------------------------+                                                                             
| i3c_master_add_i3c_dev_locked | : alloc i3c_dev, attach to master, send cmds to collect dev info, handle ibi
+-|-----------------------------+                                                                             
  |    +--------------------------+                                                                           
  |--> | i3c_master_alloc_i3c_dev | alloc newdev                                                              
  |    +--------------------------+                                                                           
  |    +---------------------------+                                                                          
  |--> | i3c_master_attach_i3c_dev | save dev addr in master, append dev to master i3c list                   
  |    +---------------------------+                                                                          
  |    +------------------------------+                                                                       
  |--> | i3c_master_retrieve_dev_info | send cmds to collect device config                                    
  |    +------------------------------+                                                                       
  |    +-----------------------------+                                                                        
  |--> | i3c_master_attach_boardinfo | find matched board info from master, save in arg                       
  |    +-----------------------------+                                                                        
  |    +-------------------------------------+                                                                
  |--> | i3c_master_search_i3c_dev_duplicate | search if there's old dev sharing the same pid with new one    
  |    +-------------------------------------+                                                                
  |                                                                                                           
  |--> if found                                                                                               
  |    |                                                                                                      
  |    |--> copy config from old to new                                                                       
  |    |                                                                                                      
  |    |--> send cmd to set mrl/mwl if necessary                                                              
  |    |                                                                                                      
  |    |                                                                                                      
  |    +--> detach i3c_dev from master, release resources, free i3c_dev                                       
  |                                                                                                           
  |--> determine expected addr                                                                                
  |                                                                                                           
  |--> if old addr != new addr                                                                                
  |    |                                                                                                      
  |    |    +----------------------------+                                                                    
  |    |--> | i3c_master_setnewda_locked | send cmd ('set dada' or 'set newda') to dst                        
  |    |    +----------------------------+                                                                    
  |    |    +-----------------------------+                                                                   
  |    +--> | i3c_master_reattach_i3c_dev | update slot status, set group dat, save addr in master            
  |         +-----------------------------+                                                                   
  |                                                                                                           
  +--> if ibi handler exists                                                                                  
       |                                                                                                      
       |    +----------------------------+                                                                    
       |--> | i3c_dev_request_ibi_locked | prepare ibi, alloc pool/slots, save dev in master                  
       |    +----------------------------+                                                                    
       |                                                                                                      
       +--> if enable_ibi is set                                                                              
            |                                                                                                 
            |    +---------------------------+                                                                
            +--> | i3c_dev_enable_ibi_locked | send cmd 'enec' to dst, enable ibi interrupt                   
                 +---------------------------+                                                                
                                                                                                              
```

```
drivers/i3c/master.c                                                               
+---------------------------+                                                       
| i3c_dev_enable_ibi_locked | : send cmd 'enec' to dst, enable ibi interrupt        
+-|-------------------------+                                                       
  |                                                                                 
  +--> call ->enable_ibi, e.g.,                                                     
       +------------------------------+                                             
       | aspeed_i3c_master_enable_ibi | send cmd 'enec' to dst, enable ibi interrupt
       +------------------------------+                                             
```

```
drivers/i3c/master/ast2600-i3c-master.c                                            
+------------------------------+                                                    
| aspeed_i3c_master_enable_ibi | : send cmd 'enec' to dst, enable ibi interrupt     
+-|----------------------------+                                                    
  |    +------------------------+                                                   
  |--> | i3c_master_enec_locked | send cmd 'extended native error correction' to dst
  |    +------------------------+                                                   
  |                                                                                 
  +--> enable/disable the ibi interrupt accordingly                                 
```

```
drivers/i3c/master.c                                                             
+----------------------------+                                                    
| i3c_dev_request_ibi_locked | : prepare ibi, alloc pool/slots, save dev in master
+-|--------------------------+                                                    
  |                                                                               
  |--> alloc ibi                                                                  
  |                                                                               
  |--> given arg req, setup ibi                                                   
  |                                                                               
  +--> call ->request_ibi, e.g.,                                                  
       +-------------------------------+                                          
       | aspeed_i3c_master_request_ibi | alloc pool/slots, save dev in master     
       +-------------------------------+                                          
```

```
drivers/i3c/master/ast2600-i3c-master.c                                
+-------------------------------+                                       
| aspeed_i3c_master_request_ibi | : alloc pool/slots, save dev in master
+-|-----------------------------+                                       
  |    +----------------------------+                                   
  |--> | i3c_generic_ibi_alloc_pool | alloc pool and slots              
  |    +----------------------------+                                   
  |                                                                     
  +--> save dev in master                                               
```

```
drivers/i3c/master.c                                                                                          
+----------------------------+                                                                                 
| i3c_generic_ibi_alloc_pool | : alloc pool and slots                                                          
+-|--------------------------+                                                                                 
  |                                                                                                            
  |--> alloc pool                                                                                              
  |                                                                                                            
  |--> alloc that many slots                                                                                   
  |                                                                                                            
  |--> if payload len is given                                                                                 
  |    -                                                                                                       
  |    +--> alloc payload buf for pool                                                                         
  |                                                                                                            
  +--> for each slot in pool                                                                                   
       |                                                                                                       
       |    +--------------------------+                                                                       
       |--> | i3c_master_init_ibi_slot | init work with handler                                                
       |    +--------------------------+ +-----------------------+                                             
       |                                 | i3c_master_handle_ibi | run callback on payload, return slot to pool
       |                                 +-----------------------+                                             
       |                                                                                                       
       |--> point to somewhere in the allocated buf                                                            
       |                                                                                                       
       |    +---------------+                                                                                  
       +--> | list_add_tail | append to free_slots list of pool                                                
            +---------------+                                                                                  
```

```
drivers/i3c/master.c                                                                   
+-----------------------------+                                                         
| i3c_master_reattach_i3c_dev | : update slot status, set group dat, save addr in master
+-|---------------------------+                                                         
  |                                                                                     
  |--> if addrs mismatch between old/new                                                
  |    |                                                                                
  |    |    +------------------------------+                                            
  |    |--> | i3c_bus_get_addr_slot_status | given new addr, get slot status from master
  |    |    +------------------------------+                                            
  |    |    +------------------------------+                                            
  |    |--> | i3c_bus_set_addr_slot_status | set status (i3c dev) to slot of addr       
  |    |    +------------------------------+                                            
  |    |                                                                                
  |    +--> if old addr is given                                                        
  |         |                                                                           
  |         |    +------------------------------+                                       
  |         +--> | i3c_bus_set_addr_slot_status | set status (free) to slot of addr     
  |              +------------------------------+                                       
  |                                                                                     
  +--> if ->reattach_i3c_dev exists                                                     
       |                                                                                
       +--> call it, e.g.,                                                              
            +------------------------------------+                                      
            | aspeed_i3c_master_reattach_i3c_dev | set group dat, save addr in master   
            +------------------------------------+                                      
```

```
drivers/i3c/master.c                                                                  
+---------------------------+                                                          
| i3c_master_detach_i3c_dev | : detach i3c dev from master, release resources          
+-|-------------------------+                                                          
  |                                                                                    
  |--> if ->detach_i3c_dev exists                                                      
  |    -                                                                               
  |    +--> call it, e.g.,                                                             
  |         +----------------------------------+                                       
  |         | aspeed_i3c_master_detach_i3c_dev | detach dev from master                
  |         +----------------------------------+                                       
  |    +--------------------------+                                                    
  |--> | i3c_master_put_i3c_addrs | for both static/dynamic, set 'free' to slot of addr
  |    +--------------------------+                                                    
  |    +----------+                                                                    
  +--> | list_del | remove dev from list (master?)                                     
       +----------+                                                                    
```

```
drivers/i3c/master.c                                                             
+-----------------------------+                                                   
| i3c_master_attach_boardinfo | : find matched board info from master, save in arg
+-|---------------------------+                                                   
  |                                                                               
  +--> for each board_info in master                                              
       |                                                                          
       |--> if it doesn't match arg dev id, continue                              
       |                                                                          
       +--> save board_info and static_addr in arg                                
```

```
drivers/i3c/master.c                                                                      
+------------------------------+                                                           
| i3c_master_retrieve_dev_info | : send cmds to collect device config                      
+-|----------------------------+                                                           
  |    +------------------------------+                                                    
  |--> | i3c_bus_get_addr_slot_status | given addr, get slot status from master            
  |    +------------------------------+                                                    
  |    +--------------------------+                                                        
  |--> | i3c_master_getpid_locked | send 'get peripheral id' cmd to dst                    
  |    +--------------------------+                                                        
  |    +--------------------------+                                                        
  |--> | i3c_master_getbcr_locked | send 'get bus configuration register' cmd to dst       
  |    +--------------------------+                                                        
  |    +--------------------------+                                                        
  |--> | i3c_master_getdcr_locked | send 'get device configuration register' cmd to dst    
  |    +--------------------------+                                                        
  |                                                                                        
  |--> if read-back bcr shows there's speed limit                                          
  |    |                                                                                   
  |    |    +---------------------------+                                                  
  |    +--> | i3c_master_getmxds_locked | send 'get maximum extended data size' cmd to dst 
  |         +---------------------------+                                                  
  |    +--------------------------+                                                        
  |--> | i3c_master_getmrl_locked | send 'master read limit' cmd to dst                    
  |    +--------------------------+                                                        
  |    +--------------------------+                                                        
  |--> | i3c_master_getmwl_locked | send 'master write limit' cmd to dst                   
  |    +--------------------------+                                                        
  |                                                                                        
  +--> if read-back bcr shows there's high-data-rate capability                            
       |                                                                                   
       |    +-----------------------------+                                                
       +--> | i3c_master_gethdrcap_locked | send 'get high-data-rate capability' cmd to dst
            +-----------------------------+                                                
```
