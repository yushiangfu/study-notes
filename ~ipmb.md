```
drivers/char/ipmi/ipmb_dev_int.c                                                                     
+------------+                                                                                        
| ipmb_probe | : prepare ipmb_dev, register misc dev, register slave callback                         
+-|----------+                                                                                        
  |                                                                                                   
  |--> alloc ipmb_dev                                                                                 
  |                                                                                                   
  |--> setup misc_dev and install 'ipmb_fops'                                                         
  |    +-----------+                                                                                  
  |    | ipmb_read | get data from first element in req_queue, copy to user                           
  |    +-----------+                                                                                  
  |    +------------+                                                                                 
  |    | ipmb_write | get msg from user, send out i2c or smbus packet                                 
  |    +------------+                                                                                 
  |                                                                                                   
  |    +---------------+                                                                              
  |--> | misc_register | determine dev#, add arg 'misc' to list (misc_list)                           
  |    +---------------+                                                                              
  |                                                                                                   
  |--> check if property "i2c-protocol" is set in dt                                                  
  |                                                                                                   
  |    +--------------------+                                                                         
  +--> | i2c_slave_register | set up client (callback) and register to bus as slave, enable slave mode
       +--------------------+ +---------------+                                                       
                              | ipmb_slave_cb | assemble request, prepend to queue, wake up waiter(s) 
                              +---------------+                                                       
```

```
drivers/char/ipmi/ipmb_dev_int.c                                             
+-----------+                                                                 
| ipmb_read | : get data from first element in req_queue, copy to user        
+-|---------+                                                                 
  |                                                                           
  |--> while request_queue is empty                                           
  |    |                                                                      
  |    |    +--------------------------+                                      
  |    +--> | wait_event_interruptible | wait till request_queue has something
  |         +--------------------------+                                      
  |                                                                           
  |--> get first element from queue                                           
  |                                                                           
  |--> copy data from element to msg                                          
  |                                                                           
  |    +----------+                                                           
  |--> | list_del | remove element from queue                                 
  |    +----------+                                                           
  |    +--------------+                                                       
  +--> | copy_to_user |                                                       
       +--------------+                                                       
```

```
drivers/char/ipmi/ipmb_dev_int.c                                  
+------------+                                                     
| ipmb_write | : get msg from user, send out i2c or smbus packet   
+-|----------+                                                     
  |    +----------------+                                          
  |--> | copy_from_user | copy msg from user                       
  |    +----------------+                                          
  |                                                                
  |--> if ipmb_dev supports i2c protocol (not smbus)               
  |    |                                                           
  |    |    +----------------+                                     
  |    |--> | ipmb_i2c_write | setup i2c msg and trasfer i2c packet
  |    |    +----------------+                                     
  |    +--> return                                                 
  |                                                                
  |--> get slave_addr and net_lun from msg                         
  |                                                                
  |    +----------------------------+                              
  +--> | i2c_smbus_write_block_data |                              
       +----------------------------+                              
```

```
drivers/char/ipmi/ipmb_dev_int.c                                                                                            
+---------------+                                                                                                            
| ipmb_slave_cb | : assemble request, prepend to queue, wake up waiter(s)                                                    
+-|-------------+                                                                                                            
  |                                                                                                                          
  +--> switch event                                                                                                          
                                                                                                                             
       case write_requested                                                                                                  
       -                                                                                                                     
       +--> buf[1] = client addr                                                                                             
                                                                                                                             
       case write_received                                                                                                   
       -                                                                                                                     
       +--> buf[+idx] = val                                                                                                  
                                                                                                                             
       case stop                                                                                                             
       |                                                                                                                     
       |    +---------------------+                                                                                          
       +--> | ipmb_handle_request | alloc queue_elem, copy request from ipmb_dev, prepend to request_queue, wake up waiter(s)
            +---------------------+                                                                                          
```

```
drivers/char/ipmi/ipmb_dev_int.c                                                                                  
+---------------------+                                                                                            
| ipmb_handle_request | : alloc queue_elem, copy request from ipmb_dev, prepend to request_queue, wake up waiter(s)
+-|-------------------+                                                                                            
  |                                                                                                                
  |--> alloc queue_elem                                                                                            
  |                                                                                                                
  |--> copy request from ipmb_dev to queue_elem                                                                    
  |                                                                                                                
  |    +----------+                                                                                                
  |--> | list_add | prepend queue_elem to request_queue                                                            
  |    +----------+                                                                                                
  |    +-------------+                                                                                             
  +--> | wake_up_all | wake up whichever waiting on the queue                                                      
       +-------------+                                                                                             
```
