> The note is based on Linux version 5.15.0 in OpenBMC.

## Index

- [Introduction](#introduction)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

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
    +------> | usb_gadget_giveback_request | control led and call ->complete()
             +-----------------------------+                                  
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
  
## <a name="reference"></a> Reference

- [T. Petazzoni, Using USB gadget drivers](https://bootlin.com/doc/legacy/usb-gadget/usb_gadget_drivers.pdf)
