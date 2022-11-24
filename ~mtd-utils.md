```
misc-utils/flashcp.c                              
+------+                                           
| main |                                           
+-|----+                                           
  |                                                
  |--> handle options from arguments               
  |                                                
  |    +-----------+                               
  |--> | safe_open | open device file (dst)        
  |    +-----------+                               
  |    +-------+                                   
  |--> | ioctl | opt = mem_get_info                
  |    +-------+                                   
  |    +-----------+                               
  |--> | safe_open | open image file (src)         
  |    +-----------+                               
  |                                                
  |--> determine erase size                        
  |                                                
  |    +-------+                                   
  |--> | ioctl | mem_erase                         
  |    +-------+                                   
  |                                                
  |--> while we haven't finished the write         
  |    |                                           
  |    |    +-----------+                          
  |    |--> | safe_read | read data from image file
  |    |    +-----------+                          
  |    |    +-------+                              
  |    +--> | write | write to device filew        
  |         +-------+                              
  |    +-------------+                             
  |--> | safe_rewind | rest image file offset to 0 
  |    +-------------+                             
  |    +-------------+                             
  |--> | safe_rewind | rest device file offset to 0
  |    +-------------+                             
  |                                                
  +--> while we haven't finished the verification  
       |                                           
       |--> read from image and device files       
       |                                           
       |    +--------+                             
       +--> | memcmp | compare data                
            +--------+                             
```

```
https://git.infradead.org/mtd-utils.git
```
