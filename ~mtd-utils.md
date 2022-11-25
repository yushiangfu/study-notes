
### flashcp
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

### flash_erase

```
misc-utils/flash_erase.c                                                  
+------+                                                                   
| main |                                                                   
+-|----+                                                                   
  |                                                                        
  |--> handle arguments                                                    
  |                                                                        
  |    +-------------+                                                     
  |--> | libmtd_open | prepare 'lib' based on info under /sys/class/mtd    
  |    +-------------+                                                     
  |    +------+                                                            
  |--> | open | open mtd device file                                       
  |    +------+                                                            
  |    +------------------+                                                
  |--> | mtd_get_dev_info | given mtd device file, read info into arg 'mtd'
  |    +------------------+                                                
  |                                                                        
  |--> determine if we erase all within one mtd                            
  |                                                                        
  |--> if erase all                                                        
  |    |                                                                   
  |    |    +-----------------+                                            
  |    +--> | mtd_erase_multi | set area info and ioctl (mem_erase)        
  |         +-----------------+                                            
  |                                                                        
  +--> else (specific region)                                              
       -                                                                   
       +--> loop                                                           
            |                                                              
            |    +-----------+                                             
            +--> | mtd_erase |                                             
                 +-----------+                                             
```

```
lib/libmtd.c                                                                       
+------------------+                                                                 
| mtd_get_dev_info | : given mtd device file, read info into arg 'mtd'               
+-|----------------+                                                                 
  |    +--------------+                                                              
  |--> | dev_node2num | given mtd device file, return the mtd#                       
  |    +--------------+                                                              
  |    +-------------------+                                                         
  +--> | mtd_get_dev_info1 | read mtd info from different files and save to arg 'mtd'
       +-------------------+                                                         
```

```
lib/libmtd.c                                                   
+--------------+                                                
| dev_node2num | : given mtd device file, return the mtd#       
+-|------------+                                                
  |                                                             
  |--> get major/minor of given device file                     
  |                                                             
  |    +--------------+                                         
  |--> | mtd_get_info |  scan /sys/class/mtd/ to update mtd_info
  |    +--------------+                                         
  |                                                             
  +--> for each mtd                                             
       |                                                        
       |    +--------------+                                    
       |--> | mtd_get_info | get its major/minor                
       |    +--------------+                                    
       |                                                        
       +--> return if the match if found                        
```

```
lib/libmtd.c                                            
+--------------+                                          
| mtd_get_info | : scan /sys/class/mtd/ to update mtd_info
+-|------------+                                          
  |    +---------+                                        
  |--> | opendir | open /sys/class/mtd/ (?)               
  |    +---------+                                        
  |                                                       
  |--> endless loop                                       
  |    |                                                  
  |    |    +---------+                                   
  |    |--> | readdir | read dir_ent                      
  |    |    +---------+                                   
  |    |                                                  
  |    |--> break if read nothing                         
  |    |                                                  
  |    +--> read mtd# from file name                      
  |                                                       
  |    +----------+                                       
  +--> | closedir |                                       
       +----------+                                       
```

```
lib/libmtd.c                                                                 
+-------------------+                                                           
| mtd_get_dev_info1 | : read mtd info from different files and save to arg 'mtd'
+-|-----------------+                                                           
  |    +---------------+                                                        
  |--> | dev_get_major | get major/minor                                        
  |    +---------------+                                                        
  |    +---------------+                                                        
  |--> | dev_read_data | read mtd name                                          
  |    +---------------+                                                        
  |    +---------------+                                                        
  |--> | dev_read_data | read mtd type                                          
  |    +---------------+                                                        
  |                                                                             
  +--> read other info, e.g., eb_size, mtd_size, io_size, ...                   
```

```
https://git.infradead.org/mtd-utils.git
```
