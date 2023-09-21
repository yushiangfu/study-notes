```
src/libsystemd/sd-event/sd-event.c                                                   
+--------------+                                                                      
| sd_event_new | : prepare event and its priority queue, create epoll_fd, return event
+-|------------+                                                                      
  |                                                                                   
  |--> prepare 'sd_event'                                                             
  |                                                                                   
  |    +------------------------+                                                     
  |--> | prioq_ensure_allocated | alloc priority queue and install compare()          
  |    +------------------------+                                                     
  |    +---------------+                                                              
  |--> | epoll_create1 |                                                              
  |    +---------------+                                                              
  |    +---------------------+                                                        
  +--> | fd_move_above_stdio | ensure the epoll_fd isn't in range [0, 2]              
       +---------------------+                                                        
```

```
src/libsystemd/sd-event/sd-event.c                                                        
+------------------+                                                                       
| sd_event_default | : ensure we have a default event, return its address                  
+-|----------------+                                                                       
  |                                                                                        
  |--> if default_event (global) has value                                                 
  |    -                                                                                   
  |    +--> return its address                                                             
  |                                                                                        
  |    +--------------+                                                                    
  |--> | sd_event_new | prepare event and its priority queue, create epoll_fd, return event
  |    +--------------+                                                                    
  |                                                                                        
  +--> default_event = new event                                                           
```

```
src/libsystemd/sd-event/sd-event.c                                                        
+--------------+                                                                       
| sd_event_ref | : ref++
+-------------+                                                                       
```

```
src/libsystemd/sd-event/sd-event.c                                                        
+----------------+                                                                       
| sd_event_unref | : ref--, free event if no one uses
+----------------+                                                                       
```

```
src/libsystemd/sd-event/sd-event.c                                                   
+-----------------+                                                                   
| sd_event_add_io | : prepare 'source' and add to arg 'event, register the source's io
+-|---------------+                                                                   
  |                                                                                   
  |--> if arg callback isn't provided                                                 
  |    -                                                                              
  |    +--> use the default one                                                       
  |         +------------------+                                                      
  |         | io_exit_callback | set event's exit fields                              
  |         +------------------+                                                      
  |    +------------+                                                                 
  |--> | source_new | alloc and init 'event source', prepend to event's queue         
  |    +------------+                                                                 
  |                                                                                   
  |--> save fd/events/callback in that 'event source'                                 
  |                                                                                   
  |    +--------------------+                                                         
  +--> | source_io_register | register arg source's io to epoll                       
       +--------------------+                                                         
```

```
src/libsystemd/sd-event/sd-event.c                       
+--------------------+                                    
| source_io_register | : register arg source's io to epoll
+-|------------------+                                    
  |                                                       
  |--> set up 'epoll event'                               
  |                                                       
  |    +-----------+                                      
  +--> | epoll_ctl | register 'epoll event'               
       +-----------+                                      
```

```
src/libsystemd/sd-event/sd-event.c                                                                       
+-------------------+                                                                                     
| sd_event_add_time | : set up timer fields of event (fd/queues), prepare 'source' and add to those queues
+-|-----------------+                                                                                     
  |    +----------------------------+                                                                     
  |--> | clock_to_event_source_type | covnert clock mode to type                                          
  |    +----------------------------+                                                                     
  |                                                                                                       
  |--> if arg callback isn't provided                                                                     
  |    -                                                                                                  
  |    +--> use the default one                                                                           
  |         +--------------------+                                                                        
  |         | time_exit_callback | set event's exit fields                                                
  |         +--------------------+                                                                        
  |                                                                                                       
  |    +----------------------+                                                                           
  |--> | event_get_clock_data | given type, get target field in event accordingly                         
  |    +----------------------+                                                                           
  |    +------------------+                                                                               
  |--> | setup_clock_data | ensure the timer fd and two priority queues of 'clock data' are ready         
  |    +------------------+                                                                               
  |    +------------+                                                                                     
  |--> | source_new | prepare 'source' for event                                                          
  |    +------------+                                                                                     
  |                                                                                                       
  |--> set up 'source' (next, callback, enabled, ...)                                                     
  |                                                                                                       
  |    +-----------------------------+                                                                    
  +--> | event_source_time_prioq_put | place 'source' into clock_data's earliest and latest queues        
       +-----------------------------+                                                                    
```

```
src/libsystemd/sd-event/sd-event.c                                                         
+------------------+                                                                        
| setup_clock_data | : ensure the timer fd and two priority queues of 'clock data' are ready
+-|----------------+                                                                        
  |                                                                                         
  |--> if 'clock data' has no fd yet                                                        
  |    |                                                                                    
  |    |    +----------------------+                                                        
  |    +--> | event_setup_timer_fd | create timer fd, add to epoll, save in 'clock data'    
  |         +----------------------+                                                        
  |    +------------------------+                                                           
  |--> | prioq_ensure_allocated | ensure priority queue (clock_data->earliest) exists       
  |    +------------------------+                                                           
  |    +------------------------+                                                           
  +--> | prioq_ensure_allocated | ensure priority queue (clock_data->latest) exists         
       +------------------------+                                                           
```

```
src/libsystemd/sd-event/sd-event.c                                           
+----------------------+                                                      
| event_setup_timer_fd | : create timer fd, add to epoll, save in 'clock data'
+-|--------------------+                                                      
  |    +----------------+                                                     
  |--> | timerfd_create | create timer fd for later poll                      
  |    +----------------+                                                     
  |    +---------------------+                                                
  |--> | fd_move_above_stdio | ensure the fd isn't in range [0, 2]            
  |    +---------------------+                                                
  |    +-----------+                                                          
  |--> | epoll_ctl | add the timer fd to epoll                                
  |    +-----------+                                                          
  |                                                                           
  +--> save the timer fd in arg 'clock data'                                  
```

```
src/libsystemd/sd-event/sd-event.c                                                    
+--------------------+                                                                 
| sd_event_add_defer | : prepare 'source' for defer, set pending = true                
+-|------------------+                                                                 
  |                                                                                    
  |--> if arg callback isn't provided                                                  
  |    -                                                                               
  |    +--> use the default one                                                        
  |         +-----------------------+                                                  
  |         | generic_exit_callback | set event's exit fields                          
  |         +-----------------------+                                                  
  |    +------------+                                                                  
  |--> | source_new | prepare 'source' for defer                                       
  |    +------------+                                                                  
  |                                                                                    
  |--> set up source (install callback, set oneshot)                                   
  |                                                                                    
  |    +--------------------+                                                          
  +--> | source_set_pending | given target value and source type, handle pending action
       +--------------------+                                                          
```

```
src/libsystemd/sd-event/sd-event.c                                                                                 
+--------------------+                                                                                              
| source_set_pending | : given target value and source type, handle pending action                                  
+-|------------------+                                                                                              
  |                                                                                                                 
  |--> if 'pending' is the target value already, return                                                             
  |                                                                                                                 
  |--> if target value == true                                                                                      
  |    |                                                                                                            
  |    |    +-----------+                                                                                           
  |    +--> | prioq_put | place 'source' on event's pending queue                                                   
  |         +-----------+                                                                                           
  |                                                                                                                 
  |--> else, remove 'source' from pending queue                                                                     
  |                                                                                                                 
  |--> if the source is a timer source                                                                              
  |    |                                                                                                            
  |    |    +-----------------------------------+                                                                   
  |    +--> | event_source_time_prioq_reshuffle | reshuffle target source in event's both queues (earliest & latest)
  |         +-----------------------------------+                                                                   
  |                                                                                                                 
  |--> if source type is 'signal' and target value is 'false'                                                       
  |    |                                                                                                            
  |    |    +-------------+                                                                                         
  |    +--> | hashmap_get | get 'signal data' from event's hashmap                                                  
  |         +-------------+                                                                                         
  |                                                                                                                 
  +--> if souce type is 'inotify'                                                                                   
       -                                                                                                            
       +--> given target value, increment or decrement the pending# in 'inotify data' (embedded in 'source')        
```

```
src/libsystemd/sd-event/sd-event.c                                                                       
+-----------------------------------+                                                                     
| event_source_time_prioq_reshuffle | : reshuffle target source in event's both queues (earliest & latest)
+-|---------------------------------+                                                                     
  |                                                                                                       
  |--> determine and get 'clock data' from event                                                          
  |                                                                                                       
  |    +-----------------+                                                                                
  |--> | prioq_reshuffle | reshuffle target node in queue (earliest)                                      
  |    +-----------------+                                                                                
  |    +-----------------+                                                                                
  +--> | prioq_reshuffle | reshuffle target node in queue (latest)                                        
       +-----------------+                                                                                
```

```
src/basic/prioq.c                                                                                                     
+-----------------+                                                                                                    
| prioq_reshuffle | : reshuffle target node in queue                                                                   
+-|---------------+                                                                                                    
  |    +-----------+                                                                                                   
  |--> | find_item | given idx, find target item in priority queue                                                     
  |    +-----------+                                                                                                   
  |                                                                                                                    
  |--> if it's not even there, return                                                                                  
  |                                                                                                                    
  |    +--------------+                                                                                                
  |--> | shuffle_down | apply binary tree search idea on queue, adjust target node downward (do nothing if unnecessary)
  |    +--------------+                                                                                                
  |    +------------+                                                                                                  
  +--> | shuffle_up | anjudst replaced node upward (could be the original one)                                         
       +------------+                                                                                                  
```

```
src/libsystemd/sd-event/sd-event.c                                            
+-------------------+                                                          
| sd_event_add_post | : prepare 'source' for post and add to event's hash table
+-|-----------------+                                                          
  |    +------------+                                                          
  |--> | source_new | create 'source' for post                                 
  |    +------------+                                                          
  |                                                                            
  |--> set up source (install callback, set event_on)                          
  |                                                                            
  +--> +----------------+                                                      
       | set_ensure_put | insert 'source' into event's hash table of post      
       +----------------+                                                      
```

```
src/libsystemd/sd-event/sd-event.c                                            
+-------------------+                                                          
| sd_event_add_exit | : prepare 'source' for exit and add to even'ts exit queue
+-|-----------------+                                                          
  |    +------------------------+                                              
  |--> | prioq_ensure_allocated | ensure priority queue is allocated           
  |    +------------------------+                                              
  |    +------------+                                                          
  |--> | source_new | prepare 'source' for exit                                
  |    +------------+                                                          
  |                                                                            
  |--> set up 'source' (install callback, set oneshot)                         
  |                                                                            
  |    +-----------+                                                           
  +--> | prioq_put | insert 'source' into event's exit queue                   
       +-----------+                                                           
```
