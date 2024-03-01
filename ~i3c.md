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
