```
peci.c                                              
+-----------------+                                  
| peci_SetDevName | : setup peci_device_list[]       
+-|---------------+                                  
  |                                                  
  |--> if arg 'peci_dev' is given                    
  |    -                                             
  |    +--> save it in peci_device_list[0]           
  |                                                  
  +--> else                                          
       |                                             
       |--> peci_device_list[0] = "/dev/peci-default"
       |                                             
       +--> peci_device_list[1] = "/dev/peci-0"      
```

```
peci.c                                                                    
+-----------+                                                              
| peci_Lock | : open peci dev                                              
+-|---------+                                                              
  |    +------+                                                            
  |--> | open | open peci_device_list[0] (peci_device_list[1] for fallback)
  |    +------+                                                            
  |                                                                        
  +--> (timeout is used for sleep before re-open)                          
```

```
peci.c                                          
+-----------+                                    
| peci_Ping | : ping (thru ioctl) target peci dev
+-|---------+                                    
  |                                              
  |--> check if addr is valid, in [0x30 ~ 0x37]  
  |                                              
  |    +-----------+                             
  |--> | peci_Open | open peci dev               
  |    +-----------+                             
  |    +---------------+                         
  |--> | peci_Ping_seq | ioctl (ping)            
  |    +---------------+                         
  |    +------------+                            
  +--> | peci_Close | close peci dev             
       +------------+                            
```

```
peci.c                                                                               
+------------------+                                                                  
| peci_RdPkgConfig | : ioctl to read pkg config, copy to arg                          
+----------------------+                                                              
| peci_RdPkgConfig_dom | : ioctl to read pkg config, copy to arg                      
+-|--------------------+                                                              
  |    +-----------+                                                                  
  |--> | peci_Open | open peci dev                                                    
  |    +-----------+                                                                  
  |    +--------------------------+                                                   
  |--> | peci_RdPkgConfig_seq_dom | setup param, ioctl to read pkg config, copy to arg
  |    +--------------------------+                                                   
  |    +------------+                                                                 
  +--> | peci_Close | close peci dev                                                  
       +------------+                                                                 
```

```
peci.c                                                                          
+--------------------------+                                                     
| peci_RdPkgConfig_seq_dom | : setup param, ioctl to read pkg config, copy to arg
+-|------------------------+                                                     
  |                                                                              
  |--> setup cmd param                                                           
  |                                                                              
  |    +-------------------+                                                     
  |--> | HW_peci_issue_cmd | ioctl (read pkg config)                             
  |    +-------------------+                                                     
  |                                                                              
  +--> copy pkg_config info to arg                                               
```
