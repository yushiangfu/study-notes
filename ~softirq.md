```
kernel/softirq.c                                                                 
+--------------+                                                                  
| softirq_init | : for tasklet and tasklet_hi: init list head and register handler
+-|------------+                                                                  
  |                                                                               
  |--> init percpu tasklet list heads for tasklet and tasklet_hi                  
  |                                                                               
  |    +--------------+                                                           
  |--> | open_softirq | register handlers for tasklet                             
  |    +--------------+ +----------------+                                        
  |                     | tasklet_action | handle each tasklet in list            
  |                     +----------------+                                        
  |    +--------------+                                                           
  +--> | open_softirq | register handlers for tasklet_hi                          
       +--------------+ +-------------------+                                     
                        | tasklet_hi_action | handle each tasklet in list         
                        +-------------------+                                     
```

```
kernel/softirq.c                                                                                                             
+----------------+                                                                                                            
| tasklet_action | : handle each tasklet in list                                                                              
+-|--------------+                                                                                                            
  |    +-----------------------+                                                                                              
  +--> | tasklet_action_common | : handle each tasklet in list                                                                
       +-|---------------------+                                                                                              
         |                                                                                                                    
         |--> save list locally and reset the global one                                                                      
         |                                                                                                                    
         +--> for each node in local list                                                                                     
              |                                                                                                               
              |--> try lock, if successful                                                                                    
              |    -                                                                                                          
              |    +--> if not count (enabled)                                                                                
              |         |                                                                                                     
              |         |    +---------------------+                                                                          
              |         |--> | tasklet_clear_sched | clear 'sched'                                                            
              |         |    +---------------------+                                                                          
              |         |                                                                                                     
              |         |--> call ->callback() or ->func(), e.g.,                                                             
              |         |    +------------------------+                                                                       
              |         |    | aspeed_mctp_rx_tasklet | for each valid header, prepare rx_packet and dispatch to target client
              |         |    +------------------------+                                                                       
              |         |                                                                                                     
              |         +--> continue                                                                                         
              |                                                                                                               
              |--> (reaching here means lock fails or it's disabled)                                                          
              |                                                                                                               
              |--> attach remaining nodes back to global list                                                                 
              |                                                                                                               
              |    +------------------------+                                                                                 
              +--> | __raise_softirq_irqoff | raise the corresponding flag in softirq bitmap                                  
                   +------------------------+                                                                                 
```
