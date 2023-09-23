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

```
src/libsystemd/sd-event/sd-event.c                                                                   
+------------------+                                                                                  
| sd_event_prepare | : register event's sources to epoll, arm all kinds of timers, set state = armed  
+-|----------------+                                                                                  
  |                                                                                                   
  |--> set event state = prepared                                                                     
  |                                                                                                   
  |    +---------------+                                                                              
  |--> | event_prepare | for each source in event: register bus of source to epoll                    
  |    +---------------+                                                                              
  |                                                                                                   
  |--> set event state = initial                                                                      
  |                                                                                                   
  |    +----------------------------------+                                                           
  |--> | event_memory_pressure_write_list | for each source in event queue: write data and free buffer
  |    +----------------------------------+                                                           
  |    +-----------------+                                                                            
  |--> | event_arm_timer | determine sleep time and arm timer                                         
  |    +-----------------+                                                                            
  |    +-----------------+                                                                            
  |--> | event_arm_timer | (the above is for real time, this is for boot time)                        
  |    +-----------------+                                                                            
  |    +-----------------+                                                                            
  |--> | event_arm_timer | (the is for monotonic)                                                     
  |    +-----------------+                                                                            
  |    +-----------------+                                                                            
  |--> | event_arm_timer | (the is for realtime alarm)                                                
  |    +-----------------+                                                                            
  |    +-----------------+                                                                            
  |--> | event_arm_timer | (the is for boottime alarm)                                                
  |    +-----------------+                                                                            
  |    +----------------------------+                                                                 
  |--> | event_close_inode_data_fds | for each fd in event's close list: close fd and remove from list
  |    +----------------------------+                                                                 
  |                                                                                                   
  +--> set state = armed                                                                              
```

```
src/libsystemd/sd-event/sd-event.c                                                               
+---------------+                                                                                 
| event_prepare | : for each source in event: register bus of source to epoll                     
+-|-------------+                                                                                 
  |                                                                                               
  |--> endless loop                                                                               
  |                                                                                               
  |    +------------+                                                                             
  |--> | prioq_peek | get the first 'source' from event's prepare queue                           
  |    +------------+                                                                             
  |                                                                                               
  |--> source iteration = event iteration                                                         
  |                                                                                               
  |    +-----------------+                                                                        
  |--> | prioq_reshuffle | adjust 'source' position in prepare queue                              
  |    +-----------------+                                                                        
  |                                                                                               
  |--> call ->prepare(), e.g.,                                                                    
  |    +------------------+                                                                       
  |    | prepare_callback | register bus source to epoll, determine timeout, set enabled = oneshot
  |    +------------------+                                                                       
  |                                                                                               
  +--> if source ref == 0                                                                         
       |                                                                                          
       |     +-------------+                                                                      
       +-->  | source_free | given source type, free itself and related resource (fd, buffer)     
             +-------------+                                                                      
```

```
src/libsystemd/sd-bus/sd-bus.c                                                                       
+------------------+                                                                                  
| prepare_callback | : register bus source to epoll, determine timeout, set enabled = oneshot         
+-|----------------+                                                                                  
  |    +-------------------+                                                                          
  |--> | sd_bus_get_events | given bus state, determine flags (poll in and/or poll out)               
  |    +-------------------+                                                                          
  |                                                                                                   
  |--> if bus has different fd for input and output                                                   
  |    |                                                                                              
  |    |    +-------------------------------+                                                         
  |    |--> | sd_event_source_set_io_events | register (source, event) to epoll, save events in source
  |    |    +-------------------------------+                                                         
  |    |    +-------------------------------+                                                         
  |    +--> | sd_event_source_set_io_events | (the above one is for input, this one is for output)    
  |         +-------------------------------+                                                         
  |                                                                                                   
  |--> else                                                                                           
  |    |                                                                                              
  |    |    +-------------------------------+                                                         
  |    +--> | sd_event_source_set_io_events | (this one is for both input and output)                 
  |         +-------------------------------+                                                         
  |    +--------------------+                                                                         
  |--> | sd_bus_get_timeout | given bus state, determine timeout                                      
  |    +--------------------+                                                                         
  |                                                                                                   
  |--> if timeout is meaningful (not forever)                                                         
  |    |                                                                                              
  |    |    +--------------------------+                                                              
  |    +--> | sd_event_source_set_time | unset pending, save timeout in source, reshuffle the prio-q  
  |         +--------------------------+                                                              
  |    +-----------------------------+                                                                
  +--> | sd_event_source_set_enabled | given target value, register or unregister source accordingly  
       +-----------------------------+                                                                
```

```
src/libsystemd/sd-event/sd-event.c                                                         
+-------------------------------+                                                           
| sd_event_source_set_io_events | : register (source, event) to epoll, save events in source
+-|-----------------------------+                                                           
  |    +--------------------+                                                               
  |--> | source_set_pending | set source pending = false                                    
  |    +--------------------+                                                               
  |                                                                                         
  |--> if source is online (enabled or oneshot)                                             
  |    |                                                                                    
  |    |    +--------------------+                                                          
  |    +--> | source_io_register | register (source, event) to epoll                        
  |         +--------------------+                                                          
  |                                                                                         
  +--> save arg events (poll_in and/or poll_out) in source                                  
```

```
src/libsystemd/sd-event/sd-event.c                                                                       
+-----------------------------+                                                                           
| sd_event_source_set_enabled | : given target value, register or unregister source accordingly           
+-|---------------------------+                                                                           
  |                                                                                                       
  |--> if ->enabled == target value already, return                                                       
  |                                                                                                       
  |--> if target value is off                                                                             
  |    |                                                                                                  
  |    |    +----------------------+                                                                      
  |    +--> | event_source_offline | given source type, unregister it accordingly                         
  |         +----------------------+                                                                      
  |                                                                                                       
  |--> else                                                                                               
  |    |                                                                                                  
  |    |--> if source is already on                                                                       
  |    |    -                                                                                             
  |    |    +--> source->enabled = target value, and return                                               
  |    |                                                                                                  
  |    |    +---------------------+                                                                       
  |    +--> | event_source_online | given source type, register source to epoll accordingly               
  |         +---------------------+                                                                       
  |    +---------------------------------+                                                                
  +--> | event_source_pp_prioq_reshuffle | reshuffle source's 'pending' and 'prepare' queues if they exist
       +---------------------------------+                                                                
```

```
src/libsystemd/sd-event/sd-event.c                                                      
+----------------------+                                                                 
| event_source_offline | : given source type, unregister it accordingly                  
+-|--------------------+                                                                 
  |                                                                                      
  |--> if event will be disabled                                                         
  |    |                                                                                 
  |    |    +--------------------+                                                       
  |    +--> | source_set_pending | unset pending flag                                    
  |         +--------------------+                                                       
  |                                                                                      
  |--> switch source type                                                                
  |    caee io                                                                           
  |    -    +----------------------+                                                     
  |    +--> | source_io_unregister | unregister io from epoll                            
  |         +----------------------+                                                     
  |    case signal                                                                       
  |    -    +----------------------+                                                     
  |    +--> | event_gc_signal_data | unset signal bit from all three possible masks      
  |         +----------------------+                                                     
  |    case child                                                                        
  |    |--> if it's a pid event source                                                   
  |    |    -    +-------------------------------+                                       
  |    |    +--> | source_child_pidfd_unregister | unregister source from epoll          
  |    |         +-------------------------------+                                       
  |    +--> else                                                                         
  |         -    +----------------------+                                                
  |         +--> | event_gc_signal_data | unset signal bit from all three possible masks 
  |              +----------------------+                                                
  |    case exit                                                                         
  |         -    +-----------------+                                                     
  |         +--> | prioq_reshuffle | adjust its position in queue                        
  |          +-->+-----------------+                                                     
  |    case memory-pressure                                                              
  |              +-----------------------------------+                                   
  |              | source_memory_pressure_unregister | unregister source from epoll      
  |              +-----------------------------------+                                   
  |    +-----------------------------------+                                             
  +--> | event_source_time_prioq_reshuffle | reshuffle source in two queues of clock data
       +-----------------------------------+                                             
```

```
src/libsystemd/sd-event/sd-event.c                                      
+----------------------+                                                 
| event_gc_signal_data | : unset signal bit from all three possible masks
+-|--------------------+                                                 
  |                                                                      
  |--> if event still has interests in signal, return                    
  |                                                                      
  |--> if 'priority' is provided && in event hash map                    
  |    |                                                                 
  |    |    +--------------------------+                                 
  |    +--> | event_unmask_signal_data | unset the bit in mask           
  |         +--------------------------+                                 
  |                                                                      
  |--> if arg signal is in signal source                                 
  |    |                                                                 
  |    |    +--------------------------+                                 
  |    +--> | event_unmask_signal_data | unset the bit in mask           
  |         +--------------------------+                                 
  |                                                                      
  +--> if there's zero priority in event hash map                        
       |                                                                 
       |    +--------------------------+                                 
       +--> | event_unmask_signal_data | unset the bit in mask           
            +--------------------------+                                 
```

```
src/libsystemd/sd-event/sd-event.c                                                   
+---------------------+                                                               
| event_source_online | : given source type, register source to epoll accordingly     
+-|-------------------+                                                               
  |                                                                                   
  |--> if source will be turned on                                                    
  |    |                                                                              
  |    |    +--------------------+                                                    
  |    +--> | source_set_pending | set pending = false                                
  |         +--------------------+                                                    
  |                                                                                   
  |--> switch source type                                                             
  |    case io                                                                        
  |    -    +--------------------+                                                    
  |    +--> | source_io_register | register source to epoll                           
  |         +--------------------+                                                    
  |    case signal                                                                    
  |    -    +------------------------+                                                
  |    +--> | event_make_signal_data | ensure target signal is monitored by epoll     
  |         +------------------------+                                                
  |    case child                                                                     
  |    ---> if source is a pid event source                                           
  |         -    +-----------------------------+                                      
  |         +--> | source_child_pidfd_register | register source to epoll             
  |              +-----------------------------+                                      
  |         else                                                                      
  |         -    +------------------------+                                           
  |         +--> | event_make_signal_data | ensure target signal is monitored by epoll
  |              +------------------------+                                           
  |    case memory_pressure                                                           
  |    -    +---------------------------------+                                       
  |    +--> | source_memory_pressure_register | register source to epoll              
  |         +---------------------------------+                                       
  |                                                                                   
  |--> save arg 'enabled' and ratelimited' to source                                  
  |                                                                                   
  +--> reshuffle related prio-q if necessary                                          
```

```
src/libsystemd/sd-event/sd-event.c                                        
+------------------------+                                                 
| event_make_signal_data | : ensure target signal is monitored by epoll    
+-|----------------------+                                                 
  |                                                                        
  |--> given priority, get signal_data from event hash table               
  |                                                                        
  |--> if arg signal is already saved in signal data, return               
  |                                                                        
  |--> else                                                                
  |    |                                                                   
  |    |--> prepare signal data                                            
  |    |                                                                   
  |    |    +--------------------+                                         
  |    +--> | hashmap_ensure_put | insert 'signal data' to event hash table
  |         +--------------------+                                         
  |                                                                        
  |--> add signal to mask                                                  
  |                                                                        
  |    +----------+                                                        
  |--> | signalfd | register mask to signal fd (which might not exist yet) 
  |    +----------+                                                        
  |                                                                        
  |--> if it exists already, return                                        
  |                                                                        
  |    +-----------+                                                       
  +--> | epoll_ctl | registe signal fd to epoll                            
       +-----------+                                                       
```

```
src/libsystemd/sd-event/sd-event.c                                               
+-------------+                                                                   
| source_free | : given source type, free itself and related resource (fd, buffer)
+-|-----------+                                                                   
  |                                                                               
  |--> if source type is io, close fd                                             
  |                                                                               
  |--> if source type is child                                                    
  |    |                                                                          
  |    |--> if child is process-owned (?)                                         
  |    |    -                                                                     
  |    |    +--> if it doesn't exit yet                                           
  |    |         |                                                                
  |    |         |--> if it has pid fd                                            
  |    |         |    |                                                           
  |    |         |    |    +-------------------+                                  
  |    |         |    +--> | pidfd_send_signal | send kill to that child thru fd  
  |    |         |         +-------------------+                                  
  |    |         |                                                                
  |    |         +--> else                                                        
  |    |              |                                                           
  |    |              |    +------+                                               
  |    |              +--> | kill | send kill to that child thru pid              
  |    |                   +------+                                               
  |    |                                                                          
  |    |--> if child isn't waited (nobody wants it?)                              
  |    |    |                                                                     
  |    |    |    +--------+                                                       
  |    |    +--> | waitid | try to adopt it                                       
  |    |         +--------+                                                       
  |    |                                                                          
  |    +--> if child is pid-owned (?), close fd                                   
  |                                                                               
  |--> if source type is 'memory pressure'                                        
  |    -                                                                          
  |    +--> close fd and free buffer                                              
  |                                                                               
  |--> if source has destroy_callback()                                           
  |    -                                                                          
  |    +--> call it                                                               
  |                                                                               
  +--> free source related mem                                                    
```

```
src/libsystemd/sd-event/sd-event.c                                                              
+----------------------------------+                                                             
| event_memory_pressure_write_list | : for each source in event queue: write data and free buffer
+-|--------------------------------+                                                             
  |                                                                                              
  +--> endless loop                                                                              
       |                                                                                         
       |--> pop first node (source) from event's queue                                           
       |                                                                                         
       |--> if nothing left, break                                                               
       |                                                                                         
       |    +------------------------------+                                                     
       +--> | source_memory_pressure_write | write data and free buffer                          
            +------------------------------+                                                     
```

```
src/libsystemd/sd-event/sd-event.c                          
+------------------------------+                             
| source_memory_pressure_write | : write data and free buffer
+-|----------------------------+                             
  |                                                          
  |--> set locked = true (for later writing)                 
  |                                                          
  |    +-------+                                             
  |--> | write |                                             
  |    +-------+                                             
  |                                                          
  +--> free write buffer                                     
```

```
src/libsystemd/sd-event/sd-event.c                                                                 
+-----------------+                                                                                 
| event_arm_timer | : determine sleep time and arm timer                                            
+-|---------------+                                                                                 
  |    +------------+                                                                               
  |--> | prioq_peek | get the earliest source from clock data                                       
  |    +------------+                                                                               
  |                                                                                                 
  |--> if it's not on                                                                               
  |    -                                                                                            
  |    +--> disarm the timer and return                                                             
  |                                                                                                 
  |    +------------+                                                                               
  |--> | prioq_peek | get the latest source from clock data                                         
  |    +------------+                                                                               
  |    +---------------+                                                                            
  |--> | sleep_between | given the earliest and latest, calculate the sleep time till the next timer
  |    +---------------+                                                                            
  |    +----------------+                                                                           
  |--> | timespec_store | save the sleep time in struct                                             
  |    +----------------+                                                                           
  |    +-----------------+                                                                          
  |--> | timerfd_settime | arm timer                                                                
  |    +-----------------+                                                                          
  |                                                                                                 
  +--> save 'sleep time' value in clock data                                                        
```
