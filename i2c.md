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
