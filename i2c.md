```
+----------+                                      
| i2c_init | : register i2c bus and dummy driver  
+--|-------+                                      
   |    +--------------+                          
   |--> | bus_register | register 'i2c_bus_type'  
   |    +--------------+                          
   |    +----------------+                        
   +--> | i2c_add_driver | register a dummy driver
        +----------------+                        
```

```
+--------------+                                                                                 
| i2c_dev_init | : create i2c class and track addition/removal of adapters                       
+---|----------+                                                                                 
    |                                                                                            
    +--> print "i2c /dev entries driver"                                                         
    |                                                                                            
    |    +------------------------+                                                              
    |--> | register_chrdev_region | reserve the specified dev# range                             
    |    +------------------------+                                                              
    |    +--------------+                                                                        
    |--> | class_create | "i2c-dev"                                                              
    |    +--------------+                                                                        
    |    +-----------------------+                                                               
    |--> | bus_register_notifier | track the addition and removal of adapters                    
    |    +-----------------------+                                                               
    |                                                                                            
    |--> for each i2c dev                                                                        
    |                                                                                            
    |        +-----------------------+                                                           
    +------> | i2cdev_attach_adapter | bind to existing adapters, but there's none at the momemnt
             +-----------------------+                                                           
```

```
+-----------------+                           
| aspeed_i2c_init | : init i2c controller (hw)
+----|------------+                           
     |                                        
     |--> disable everything (hw)             
     |                                        
     |    +---------------------+             
     |--> | aspeed_i2c_init_clk |             
     |    +---------------------+             
     |                                        
     |--> enable master mode (hw)             
     |                                        
     +--> enable interrupt (hw)               
```

```
+---------------------+                                                    
| i2c_mux_add_adapter | : alloc priv, set up priv->adap and register it    
+-----|---------------+                                                    
      |                                                                    
      |--> alloc priv                                                      
      |                                                                    
      |--> set up algo operations                                          
      |                                                                    
      |--> set up priv->adap, e.g., parent, retry, timeout, lock ops       
      |                                                                    
      |--> if muxc has an of_node                                          
      |                                                                    
      |------> relate adaptor and child node                               
      |                                                                    
      |    +-----------------+                                             
      +--> | i2c_add_adapter | set adapter's bus# and name, and register it
           +-----------------+                                             
```

```
+-----------------+                                                                      
| i2c_add_adapter | : set adapter's bus# and name, and register it                       
+----|------------+                                                                      
     |                                                                                   
     |--> if dev has an of_node                                                          
     |                                                                                   
     |        +----------------------------+                                             
     |------> | __i2c_add_numbered_adapter | set adapter's bus# and name, and register it
     |        +----------------------------+                                             
     |                                                                                   
     |--> set adapter's bus#                                                             
     |                                                                                   
     |    +----------------------+                                                       
     +--> | i2c_register_adapter | set adapter's name, and register it                   
          +----------------------+                                                       
```
