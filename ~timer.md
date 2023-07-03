```
kernel/time/timer.c                                                                 
+-------------+                                                                      
| init_timers | : init percpu timer bases, register softirq action                   
+-|-----------+                                                                      
  |    +-----------------+                                                           
  |--> | init_timer_cpus | init percpu timer_bases                                   
  |    +-----------------+                                                           
  |    +---------------------------+                                                 
  |--> | posix_cputimers_init_work | do nothing bc of disabled config                
  |    +---------------------------+                                                 
  |    +--------------+                                                              
  +--> | open_softirq | register softirq action                                      
       +--------------+ +-------------------+                                        
                        | run_timer_softirq | run each expired timer of timer base(s)
                        +-------------------+                                        
```

```
kernel/time/timer.c                                           
+-------------------+                                          
| run_timer_softirq | : run each expired timer of timer base(s)
+-|-----------------+                                          
  |                                                            
  |--> get percpu timer base (std)                             
  |                                                            
  |    +--------------+                                        
  |--> | __run_timers | run each expired timer                 
  |    +--------------+                                        
  |                                                            
  +--> if deferrable timer base exists                         
       |                                                       
       |    +--------------+                                   
       +--> | __run_timers | run each expired timer            
            +--------------+                                   
```

```
kernel/time/timer.c                                                                 
+--------------+                                                                     
| __run_timers | : run each expired timer                                            
+-|------------+                                                                     
  |                                                                                  
  |--> if nothing expires yet, return                                                
  |                                                                                  
  +--> while there's at least one expired timer                                      
      |                                                                              
      |    +------------------------+                                                
      |--> | collect_expired_timers | collect expired timers from arg base           
      |    +------------------------+                                                
      |                                                                              
      |--> base->clk++                                                               
      |                                                                              
      |    +------------------------+                                                
      |--> | __next_timer_interrupt | look into timer base to get next expiration    
      |    +------------------------+                                                
      |                                                                              
      +--> while level--                                                             
           |                                                                         
           |    +---------------+                                                    
           +--> | expire_timers | given list, detach each timer and call its function
                +---------------+                                                    
```

```
kernel/time/timer.c                                                           
+------------------------+                                                     
| collect_expired_timers | : collect expired timers from arg base              
+-|----------------------+                                                     
  |                                                                            
  +--> for each level_depth                                                    
       |                                                                       
       |--> determine idx                                                      
       |                                                                       
       |--> if it's set in bitmap (clear it)                                   
       |    |                                                                  
       |    |--> get vector from base (base contains all vectors in all wheels)
       |    |                                                                  
       |    |--> add vector to head list                                       
       |    |                                                                  
       |    +--> heads++, level++                                              
       |                                                                       
       |--> if no need to check next level, break                              
       |                                                                       
       +--> adjust level to fit next granularity                               
```

```
kernel/time/timer.c                                                    
+------------------------+                                              
| __next_timer_interrupt | : look into timer base to get next expiration
+-|----------------------+                                              
  |                                                                     
  +--> for each level depth                                             
       |                                                                
       |    +---------------------+                                     
       |--> | next_pending_bucket | get pos of next pending bucket      
       |    +---------------------+                                     
       |                                                                
       |--> if it expires early than next, update next with it          
       |                                                                
       |--> if no timer can expire earlier, break                       
       |                                                                
       +--> adjust clk for next round                                   
```

```
kernel/time/timer.c                                                   
+---------------+                                                      
| expire_timers | : given list, detach each timer and call its function
+-|-------------+                                                      
  |                                                                    
  +--> while list isn't empty                                          
       |                                                               
       |--> get first timer from list                                  
       |                                                               
       |    +--------------+                                           
       |--> | detach_timer |                                           
       |    +--------------+                                           
       |    +---------------+                                          
       +--> | call_timer_fn |                                          
            +---------------+                                          
```
