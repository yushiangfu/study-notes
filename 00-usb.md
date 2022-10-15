> The note is based on Linux version 5.15.0 in OpenBMC.

## Index

- [Introduction](#introduction)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

```
+--------------+                                                                                                  
| ast_vhub_irq | : handle vhub irq                                                                                
+---|----------+                                                                                                  
    |    +--------+                                                                                               
    |--> | writel | ack interrupt                                                                                 
    |    +--------+                                                                                               
    |                                                                                                             
    |--> if there's interrupt for endpoint                                                                        
    |                                                                                                             
    |------> for each involved endpoint                                                                           
    |                                                                                                             
    |            +----------------------+                                                                         
    |----------> | ast_vhub_epn_ack_irq | ack epn irq                                                             
    |            +----------------------+                                                                         
    |                                                                                                             
    |--> if there's interrupt for port                                                                            
    |                                                                                                             
    |------> for each port                                                                                        
    |                                                                                                             
    |            +------------------+                                                                             
    |----------> | ast_vhub_dev_irq | handle dev irq                                                              
    |            +------------------+                                                                             
    |                                                                                                             
    |--> if there's interrupt for top-level vhub ep0                                                              
    |                                                                                                             
    |        +-------------------------+                                                                          
    |------> | ast_vhub_ep0_handle_ack | given ep0 state, handle the 1st req on list accordingly (send or receive)
    |        +-------------------------+                                                                          
    |         or                                                                                                  
    |        +---------------------------+                                                                        
    |------> | ast_vhub_ep0_handle_setup | handle req propery, pass to gadget driver if needed                    
    |        +---------------------------+                                                                        
    |                                                                                                             
    |--> if there's interrupt for top-level bus events                                                            
    |                                                                                                             
    |        +---------------------+                                                                              
    |------> | ast_vhub_hub_resume | for each port, call its driver->resume()                                     
    |        +---------------------+                                                                              
    |         or                                                                                                  
    |        +----------------------+                                                                             
    |------> | ast_vhub_hub_suspend | for each port, call its driver->suspend()                                   
    |        +----------------------+                                                                             
    |         or                                                                                                  
    |        +--------------------+                                                                               
    +------> | ast_vhub_hub_reset | for each port, call its driver->suspend(), and write hw reg to reset          
             +--------------------+                                                                               
```

```
+------------------+                                                                                               
| ast_vhub_dev_irq | : handle dev irq                                                                              
+----|-------------+                                                                                               
     |    +-------+                                                                                                
     |--> | readl | read isr status                                                                                
     |    +-------+                                                                                                
     |    +--------+                                                                                               
     |--> | writel | ack interrupt                                                                                 
     |    +--------+                                                                                               
     |                                                                                                             
     |--> if 'epo in ack stall'                                                                                    
     |                                                                                                             
     |        +-------------------------+                                                                          
     |------> | ast_vhub_ep0_handle_ack | given ep0 state, handle the 1st req on list accordingly (send or receive)
     |        +-------------------------+                                                                          
     |                                                                                                             
     |--> if 'epo out ack stall'                                                                                   
     |                                                                                                             
     |        +-------------------------+                                                                          
     |------> | ast_vhub_ep0_handle_ack | given ep0 state, handle the 1st req on list accordingly (send or receive)
     |        +-------------------------+                                                                          
     |                                                                                                             
     |--> if 'ep0 setup'                                                                                           
     |                                                                                                             
     |        +---------------------------+                                                                        
     +------> | ast_vhub_ep0_handle_setup | handle req propery, pass to gadget driver if needed                    
              +---------------------------+                                                                        
```

```
+---------------------------+                                                      
| ast_vhub_ep0_handle_setup | : handle req propery, pass to gadget driver if needed
+------|--------------------+                                                      
       |                                                                           
       |--> copy setup packet from 'ep0 setup'                                     
       |                                                                           
       |--> set ep0 state and direction                                            
       |                                                                           
       |--> if ep has no device yet                                                
       |                                                                           
       |------> if req type is standard                                            
       |                                                                           
       |            +--------------------------+                                   
       |----------> | ast_vhub_std_hub_request | handle the request properly       
       |            +--------------------------+                                   
       |                                                                           
       |------> elif req type is class                                             
       |                                                                           
       |            +----------------------------+                                 
       |----------> | ast_vhub_class_hub_request | handle the request properly     
       |            +----------------------------+                                 
       |                                                                           
       |--> elif req type is statndard                                             
       |                                                                           
       |        +--------------------------+                                       
       |------> | ast_vhub_std_dev_request | handle the request properly           
       |        +--------------------------+                                       
       |                                                                           
       |--> return if rc == data                                                   
       |                                                                           
       |--> if rc == driver (pass req to drvier)                                   
       |                                                                           
       |------> call ->setup()                                                     
       |                                                                           
       |--> elif rc == stall                                                       
       |                                                                           
       |        +--------+                                                         
       |------> | writel | write 'stall'                                           
       |        +--------+                                                         
       |                                                                           
       |--> elif rc == complete                                                    
       |                                                                           
       |        +--------+                                                         
       +------> | writel | write 'tx buf ready'                                    
                +--------+                                                         
```

```
+--------------------------+                              
| ast_vhub_std_dev_request | : handle the request properly
+------|-------------------+                              
       |                                                  
       |--> get value and index from 'ctrl req'           
       |                                                  
       |--> switch req                                    
       |                                                  
       |--> case 'set address': blabla                    
       |                                                  
       |--> case 'get status': blabla                     
       |                                                  
       +--> case 'set/clear feature': blabla              
```

```
+----------------------------+                              
| ast_vhub_class_hub_request | : handle the request properly
+------|---------------------+                              
       |                                                    
       |--> get value/index/length from ctrl req            
       |                                                    
       |--> switch req                                      
       |                                                    
       |--> case 'get status': blabla                       
       |                                                    
       |--> case 'get/set hub feature': blabla              
       |                                                    
       |--> case 'get/set port feature': blabla             
       |                                                    
       +--> case 'tt?': blabla                              
```

```
+--------------------------+                              
| ast_vhub_std_hub_request | : handle the request properly
+------|-------------------+                              
       |                                                  
       |--> get value/index/length from ctrl req          
       |                                                  
       |--> determine vhub speed                          
       |                                                  
       |--> switch req                                    
       |                                                  
       |--> case 'set address': blabla                    
       |                                                  
       |--> case 'get status': blabla                     
       |                                                  
       |--> case 'set/clear feature': blabla              
       |                                                  
       |--> case 'get/set configuration': blabla          
       |                                                  
       |--> case 'get descriptor': blabla                 
       |                                                  
       +--> case 'get/set interface': blabla              
```

```
+-------------------------+                                                                                        
| ast_vhub_ep0_handle_ack | : given ep0 state, handle the 1st req on list accordingly (send or receive)            
+------|------------------+                                                                                        
       |    +-------+                                                                                              
       |--> | readl | read ep0 status                                                                              
       |    +-------+                                                                                              
       |    +--------------------------+                                                                           
       |--> | list_first_entry_or_null | get the first req                                                         
       |    +--------------------------+                                                                           
       |                                                                                                           
       |--> switch state                                                                                           
       |                                                                                                           
       |--> case token                                                                                             
       |------> if req (shouldn't happen)                                                                          
       |            +---------------+                                                                              
       |----------> | ast_vhub_nuke | finalize all req on ep                                                       
       |            +---------------+                                                                              
       +----------> stall = true                                                                                   
       |                                                                                                           
       |--> case data                                                                                              
       |------> if no req (should have)                                                                            
       |----------> stall = true                                                                                   
       |----------> break                                                                                          
       |------> if direction is 'in'                                                                               
       |            +----------------------+                                                                       
       |----------> | ast_vhub_ep0_do_send | copy from req to ep buffer, and trigger send                          
       |            +----------------------+                                                                       
       |------> else ('out')                                                                                       
       |            +-------------------------+                                                                    
       |----------> | ast_vhub_ep0_do_receive | copy from ep buffer to req, trigger 'receive' or 'send' accordingly
       |            +-------------------------+                                                                    
       |                                                                                                           
       |--> case status                                                                                            
       +------> if req (shouldn't happen)                                                                          
       |            +---------------+                                                                              
       |            | ast_vhub_nuke | finalize all req on ep                                                       
       |            +---------------+                                                                              
       |------> if direction and ack mismatch                                                                      
       |----------> stall = true                                                                                   
       |                                                                                                           
       |--> case stall                                                                                             
       |        +---------------+                                                                                  
       |------> | ast_vhub_nuke | finalize all req on ep                                                           
       |        +---------------+                                                                                  
       |                                                                                                           
       |--> if stall                                                                                               
       |                                                                                                           
       |        +--------+                                                                                         
       +------> | writel | stall the ep                                                                            
                +--------+                                                                                         
```

```
+-------------------------+                                                                      
| ast_vhub_ep0_do_receive | : copy from ep buffer to req, trigger 'receive' or 'send' accordingly
+------|------------------+                                                                      
       |                                                                                         
       |--> calculate remaining space                                                            
       |                                                                                         
       |--> copy data from ep buffer to request                                                  
       |                                                                                         
       |--> update 'actual' size                                                                 
       |                                                                                         
       |--> if we're done (no more to receive)                                                   
       |                                                                                         
       |        +--------+                                                                       
       |------> | writel | write 'tx buf ready'                                                  
       |        +--------+                                                                       
       |        +---------------+                                                                
       |------> | ast_vhub_done | finalize req                                                   
       |        +---------------+                                                                
       |                                                                                         
       |--> else                                                                                 
       |                                                                                         
       |        +-----------------------+                                                        
       +------> | ast_vhub_ep0_rx_prime | write 'rx buf ready'                                   
                +-----------------------+                                                        
```

```
+------------------------------+                                                                                                
| ast_vhub_epn_handle_ack_desc | : handle ack desc                                                                              
+-------|----------------------+                                                                                                
        |                                                                                                                       
        |--> get 'last' from hardware register                                                                                  
        |                                                                                                                       
        |    +--------------------------+                                                                                       
        |--> | list_first_entry_or_null | get the first req from queue of ep                                                    
        |    +--------------------------+                                                                                       
        |                                                                                                                       
        |--> while ep last != 'last'                                                                                            
        |                                                                req                                                    
        |------> get desc from ep last                                 +------+                                                 
        |                                                              |  +------+                                              
        |------> ep last++                                             |  | desc |                                              
        |                                                              |  +------+                                              
        |------> continue if it's not an active req                    |  | desc |                                              
        |                                                              |  +------+                                              
        |------> if current desc is the last specified in req          |  | desc |                                              
        |                                                              +--+------+                                              
        |            +---------------+                                                                                          
        |----------> | ast_vhub_done | finalize req                                                                             
        |            +---------------+                                                                                          
        |            +--------------------------+                                                                               
        |----------> | list_first_entry_or_null | get next req from list (in case somewhere adds it suddenly?)                  
        |            +--------------------------+                                                                               
        |                                                                                                                       
        +----------> break                                                                                                      
        |                                                                                                                       
        |--> if there's more work (req)                                                                                         
        |                                                                                                                       
        |        +------------------------+                                                                                     
        +------> | ast_vhub_epn_kick_desc | given req, for each desc it needs: fill addr and size info, kill hardware to process
                 +------------------------+                                                                                     
```

```
+---------------+                                                             
| ast_vhub_done | : finalize req                                              
+---|-----------+                                                             
    |    +---------------+                                                    
    |--> | list_del_init | remove req from list                               
    |    +---------------+                                                    
    |                                                                         
    |--> if req has used dma                                                  
    |                                                                         
    |        +---------------------------------+                              
    |------> | usb_gadget_unmap_request_by_dev |                              
    |        +---------------------------------+                              
    |                                                                         
    |--> if the req isn't internal                                            
    |                                                                         
    |        +-----------------------------+                                  
    +------> | usb_gadget_giveback_request | control led and call ->complete(), e.g.,
             +-----------------------------+ composite_setup_complete()
```

```
+------------------------+                                                                                       
| ast_vhub_epn_kick_desc | : given req, for each desc it needs: fill addr and size info, kill hardware to process
+-----|------------------+                                                                                       
      |                                                                                                          
      |--> lable 'active' on req                                                                                 
      |                                                                                                          
      |--> return if the last desc in req is handled                                                             
      |                                                                                                          
      |--> while we can still create desc and req needs one                                                      
      |                                                                                                          
      |------> get next free desc from ep                                                                        
      |                                                                                                          
      |------> ep next++                                                                                         
      |                                                                                                          
      |------> calculate chunk size                                                                              
      |                                                                                                          
      |------> populate desc (save buf addr in desc)                                                             
      |                                                                                                          
      |------> save chunk size in desc                                                                           
      |                                                                                                          
      |------> if it's the end of req or we don't have enough desc                                               
      |                                                                                                          
      |----------> set 'interrupt' bit in desc                                                                   
      |                                                                                                          
      |    +--------+                                                                                            
      +--> | writel | kick hardware to process descriptors                                                       
           +--------+                                                                                            
```

```
+----------------------+                                               
| ast_vhub_ep0_do_send | : copy from req to ep buffer, and trigger send
+-----|----------------+                                               
      |                                                                
      |--> label 'last_desc' if it's the status from gadget            
      |                                                                
      |--> if we're done (can receive)                                 
      |                                                                
      |        +--------+                                              
      |------> | writel | write 'rx buff ready' to hw reg              
      |        +--------+                                              
      |        +---------------+                                       
      |------> | ast_vhub_done | finalize req                          
      |        +---------------+                                       
      |                                                                
      +------> return                                                  
      |                                                                
      |--> decide chunk size                                           
      |                                                                
      |    +--------+                                                  
      |--> | memcpy | copy data from req to ep buffer                  
      |    +--------+                                                  
      |    +--------+                                                  
      +--> | writel | save chunck size and trigger send                
           +--------+                                                  
```

```
+---------------+                                
| ast_vhub_nuke | : finalize all req on ep       
+---|-----------+                                
    |                                            
    |--> while ep queue still has something      
    |                                            
    |        +------------------+                
    |------> | list_first_entry | get the 1st req
    |        +------------------+                
    |        +---------------+                   
    +------> | ast_vhub_done | finalize req      
             +---------------+                   
```

```
+----------------------+                                       
| ast_vhub_epn_ack_irq | : ack epn irq                         
+-----|----------------+                                       
      |                                                        
      |--> if the ep is in desc mode                           
      |                                                        
      |        +------------------------------+                
      |------> | ast_vhub_epn_handle_ack_desc | handle ack desc
      |        +------------------------------+                
      |                                                        
      |--> else                                                
      |                                                        
      |        +-------------------------+                     
      +------> | ast_vhub_epn_handle_ack | handle ack          
               +-------------------------+                     
```

```
 +-------------------------+                                                                               
 | ast_vhub_epn_handle_ack | : handle ack                                                                  
 +------|------------------+                                                                               
        |    +-------+                                                                                     
        |--> | readl | read ep status                                                                      
        |    +-------+                                                                                     
        |    +--------------------------+                                                                  
        |--> | list_first_entry_or_null | get the 1st request                                              
        |    +--------------------------+                                                                  
        |                                                                                                  
        |--> return if no req                                                                              
        |                                                                                                  
        |--> go to 'next chunk' if req isn't active                                                        
        |                                                                                                  
        |--> clear 'active' on req                                                                         
        |                                                                                                  
        |--> prepare data buffer if needed, and adjust size                                                
        |                                                                                                  
        |--> label 'last' on req if it's a short packet                                                    
        |                                                                                                  
        |--> if no other desc needs handling                                                               
        |                                                                                                  
        |        +---------------+                                                                         
        |------> | ast_vhub_done | finalize req                                                            
        |        +---------------+                                                                         
        |        +--------------------------+                                                              
        |------> | list_first_entry_or_null | pick up next req                                             
        |        +--------------------------+                                                              
        |                                                                                                  
        |------> return if nothing to do                                                                   
 next chunk                                                                                                
        |    +-------------------+                                                                         
        +--> | ast_vhub_epn_kick | prepare buf, save addr to hw reg, label 'active' on req, and kick and hw
             +-------------------+                                                                         
```

```
+-------------------+                                                                           
| ast_vhub_epn_kick | : prepare buf, save addr to hw reg, label 'active' on req, and kick and hw
+----|--------------+                                                                           
     |                                                                                          
     |--> calculate chunk size                                                                  
     |                                                                                          
     |--> prepare buffer and write addr to hw register                                          
     |                                                                                          
     |--> label 'active' on req (packet on the fly)                                             
     |                                                                                          
     |    +--------+                                                                            
     +--> | writel | kick the hw                                                                
          +--------+                                                                            
```
  
## <a name="reference"></a> Reference

- [T. Petazzoni, Using USB gadget drivers](https://bootlin.com/doc/legacy/usb-gadget/usb_gadget_drivers.pdf)
