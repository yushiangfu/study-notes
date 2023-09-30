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

```
src/libsystemd/sd-event/sd-event.c                                                     
+---------------+                                                                       
| sd_event_wait | : wait for each child finishes                                        
+-|-------------+                                                                       
  |                                                                                     
  |--> for each threshold (from max to min)                                             
  |    |                                                                                
  |    |    +---------------+                                                           
  |    |--> | process_epoll | wait for data and read it, set source pending = true      
  |    |    +---------------+                                                           
  |    |    +---------------+                                                           
  |    |--> | process_child | wait till child finishes, set pending = true              
  |    |    +---------------+                                                           
  |    |                                                                                
  |    +--> if no new child event, break early                                          
  |                                                                                     
  |    +------------------+                                                             
  |--> | process_watchdog | arm a watchdog                                              
  |    +------------------+                                                             
  |    +-----------------+                                                              
  |--> | process_inotify | for each source has interest in the event, set pending = true
  |    +-----------------+                                                              
  |    +---------------+                                                                
  |--> | process_timer | find the earliest timer, set pending = true                    
  |    +---------------+ (for realtime)                                                 
  |    +---------------+                                                                
  |--> | process_timer | for boottime                                                   
  |    +---------------+                                                                
  |    +---------------+                                                                
  |--> | process_timer | for realtime_alarm                                             
  |    +---------------+                                                                
  |    +---------------+                                                                
  |--> | process_timer | for boottime_alarm                                             
  |    +---------------+                                                                
  |    +---------------+                                                                
  |--> | process_timer | for monotonic                                                  
  |    +---------------+                                                                
  |                                                                                     
  |--> if there's still source in event's pending queue                                 
  |    |                                                                                
  |    |--> set event state = pending                                                   
  |    +--> return                                                                      
  |                                                                                     
  +--> set event state = initial                                                        
```

```
src/libsystemd/sd-event/sd-event.c                                                                                   
+---------------+                                                                                                     
| process_epoll | : wait for data and read it, set source pending = true                                              
+---------------+                                                                                                     
  |--> adjust event's queue size                                                                                      
  |--> endless loop                                                                                                   
  |    |    +-----------------+                                                                                       
  |    |--> | epoll_wait_usec |                                                                                       
  |    |    +-----------------+                                                                                       
  |    +--> if condition met, break                                                                                   
  |--> if it's called first time                                                                                      
  |    -    +----------------------+                                                                                  
  |    +--> | triple_timestamp_get | get current time of three bases                                                  
  |         +----------------------+                                                                                  
  +--> for each node in event queue                                                                                   
       |--> if source type is 'watchdog'                                                                              
       |    -    +-------------+                                                                                      
       |    +--> | flush_timer | read data from fd (unused), set next = infinity                                      
       |         +-------------+                                                                                      
       +--> else                                                                                                      
            +--> switch wakeup type                                                                                   
                 case event_source                                                                                    
                 +--> switch source_type                                                                              
                      case io                                                                                         
                      -    +------------+                                                                             
                      +--> | process_io | save event in source, set pending = true                                    
                           +------------+                                                                             
                      case pidfd                                                                                      
                      -    +---------------+                                                                          
                      +--> | process_pidfd | wait till child exits                                                    
                           +---------------+                                                                          
                      case memory pressure                                                                            
                      -    +-------------------------+                                                                
                      +--> | process_memory_pressure | save event in source, set pending = true                       
                           +-------------------------+                                                                
                 case clock data                                                                                      
                 -    +-------------+                                                                                 
                 +--> | flush_timer | read data from fd (unused), set next = infinity                                 
                      +-------------+                                                                                 
                 case signal data                                                                                     
                 -    +----------------+                                                                              
                 +--> | process_signal | read signal to get target source, save it in signal_data, set pending = true 
                      +----------------+                                                                              
                 case inotify data                                                                                    
                 -    +-------------------------+                                                                     
                 +--> | event_inotify_data_read | given inotify_data, read data, prepend inotify_data to event's queue
                      +-------------------------+                                                                     
```

```
src/libsystemd/sd-event/sd-event.c                                                              
+----------------+                                                                               
| process_signal | : read signal to get target source, save it in signal_data, set pending = true
+-|--------------+                                                                               
  |                                                                                              
  |--> is signal_data->current exists (got somethign to do), return                              
  |                                                                                              
  |--> read signal info                                                                          
  |                                                                                              
  |--> given signal, get source                                                                  
  |                                                                                              
  |--> signal_data->current = source                                                             
  |                                                                                              
  |--> set pending = true                                                                        
  |                                                                                              
  +--> if the priority is smaller, update to min_priority                                        
```

```
src/libsystemd/sd-event/sd-event.c                             
+---------------+                                               
| process_child | : wait till child finishes, set pending = true
+-|-------------+                                               
  |                                                             
  +--> for each child source                                    
       |                                                        
       |    +--------+                                          
       |--> | waitid | wait till child finishes                 
       |    +--------+                                          
       |                                                        
       +--> if we got something                                 
            |                                                   
            |--> set pending = true                             
            |                                                   
            +--> update min_priority                            
```

```
src/libsystemd/sd-event/sd-event.c                                  
+------------------+                                                 
| process_watchdog | : arm a watchdog                                
+-|----------------+                                                 
  |    +-----------+                                                 
  |--> | sd_notify | get addr from env 'NOTIFY_SOCKET', create socket and connect, send msg
  |    +-----------+                                                 
  |    +--------------+                                              
  +--> | arm_watchdog | calculate wait interval, set thru watchdog fd
       +--------------+                                              
```

```
src/libsystemd/sd-daemon/sd-daemon.c                                                                         
+-----------+                                                                                                 
| sd_notify | : get addr from env 'NOTIFY_SOCKET', create socket and connect, send msg                        
+------------------------+                                                                                    
| sd_pid_notify_with_fds | : get addr from env 'NOTIFY_SOCKET', create socket and connect, send msg           
+-|----------------------+                                                                                    
  |    +------------------------------+                                                                       
  |--> | pid_notify_with_fds_internal | get addr from env 'NOTIFY_SOCKET', create socket and connect, send msg
  |    +------------------------------+                                                                       
  |                                                                                                           
  +--> if arg unset_environment is set                                                                        
       -                                                                                                      
       +--> unset 'NOTIFY_SOCKET'                                                                             
```

```
src/libsystemd/sd-daemon/sd-daemon.c                                                                    
+------------------------------+                                                                         
| pid_notify_with_fds_internal | : get addr from env 'NOTIFY_SOCKET', create socket and connect, send msg
+-|----------------------------+                                                                         
  |                                                                                                      
  |--> get env 'NOTIFY_SOCKET'                                                                           
  |                                                                                                      
  |--> parse env string into address                                                                     
  |                                                                                                      
  |    +--------+                                                                                        
  |--> | socket | create socket                                                                          
  |    +--------+                                                                                        
  |    +---------+                                                                                       
  |--> | connect |                                                                                       
  |    +---------+                                                                                       
  |    +---------------+                                                                                 
  |--> | fd_inc_sndbuf | config send buffer                                                              
  |    +---------------+                                                                                 
  |                                                                                                      
  |--> alloc msg                                                                                         
  |                                                                                                      
  |    +---------+                                                                                       
  +--> | sendmsg |                                                                                       
       +---------+                                                                                       
```

```
src/libsystemd/sd-event/sd-event.c                                                                                       
+-----------------+                                                                                                       
| process_inotify | : for each source has interest in the event, set pending = true                                       
+-|---------------+                                                                                                       
  |                                                                                                                       
  +--> for each inotify_data in event                                                                                     
       |                                                                                                                  
       |    +----------------------------+                                                                                
       +--> | event_inotify_data_process | for each source has interest in the inotify_data: set pending = true (trigger?)
            +----------------------------+                                                                                
```

```
src/libsystemd/sd-event/sd-event.c                                          
+-------------------+                                                        
| sd_event_dispatch | : dispatch 1st source from event's pending queue       
+-|-----------------+                                                        
  |                                                                          
  |--> is exit_requested is set                                              
  |    |                                                                     
  |    |    +---------------+                                                
  |    +--> | dispatch_exit | get 1st source from event, dispatch that source
  |         +---------------+                                                
  |    +--------------------+                                                
  |--> | event_next_pending | get 1st source from event's pending queue      
  |    +--------------------+                                                
  |                                                                          
  |--> set event state = running                                             
  |                                                                          
  |    +-----------------+                                                   
  |--> | source_dispatch | given source type, call its callback accordingly  
  |    +-----------------+                                                   
  |                                                                          
  +--> set event state = initial                                             
```

```
src/libsystemd/sd-event/sd-event.c                                        
+---------------+                                                          
| dispatch_exit | : get 1st source from event, dispatch that source        
+-|-------------+                                                          
  |    +------------+                                                      
  |--> | prioq_peek | get first source from event's exit queue             
  |    +------------+                                                      
  |                                                                        
  |--> event iteration++                                                   
  |                                                                        
  |--> set event state = exiting                                           
  |                                                                        
  |    +-----------------+                                                 
  |--> | source_dispatch | given source type, call its callback accordingly
  |    +-----------------+                                                 
  |                                                                        
  +--> set event state = initial                                           
```

```
src/libsystemd/sd-event/sd-event.c                                                    
+-----------------+                                                                    
| source_dispatch | : given source type, call its callback accordingly                 
+-|---------------+                                                                    
  |                                                                                    
  +--> switch source type                                                              
       case io:                                                                        
       +--> call ->callback(), e.g.,                                                   
            +------------------+                                                       
            | io_exit_callback | set event's exit fields                               
            +------------------+                                                       
       case time_xxx                                                                   
       +--> call ->callback(), e.g.,                                                   
            +--------------------+                                                     
            | time_exit_callback | set event's exit fields                             
            +--------------------+                                                     
       case signal                                                                     
       +--> call ->callback(), e.g.,                                                   
            +----------------------+                                                   
            | signal_exit_callback | set event's exit fields                           
            +----------------------+                                                   
       case child                                                                      
       +--> call ->callback(), e.g.,                                                   
            +---------------------+                                                    
            | child_exit_callback | set event's exit fields                            
            +---------------------+                                                    
            if child is zombie, reap it                                                
       case defer                                                                      
       +--> call ->callback(), e.g.,                                                   
            +-----------------------+                                                  
            | generic_exit_callback | set event's exit fields                          
            +-----------------------+                                                  
       case post                                                                       
       +--> call ->callback(), e.g.,                                                   
            +-----------------------+                                                  
            | generic_exit_callback | set event's exit fields                          
            +-----------------------+                                                  
       case exit                                                                       
       +--> call ->callback(), e.g.,                                                   
            +---------------+                                                          
            | quit_callback | if 'close on exit' is set, flush bus wqueue and close bus
            +---------------+                                                          
       case inotify                                                                    
       +--> call ->callback(), e.g.,                                                   
            +-----------------------+                                                  
            | inotify_exit_callback | set event's exit fields                          
            +-----------------------+                                                  
       case memory_pressure                                                            
       +--> call ->callback(), e.g.,                                                   
            +--------------------------+                                               
            | memory_pressure_callback | free unused hash maps, return unused memory to kernel
            +--------------------------+                                               
```

```
src/libsystemd/sd-bus/sd-bus.c                                              
+---------------+                                                            
| quit_callback | : if 'close on exit' is set, flush bus wqueue and close bus
+-|-------------+                                                            
  |                                                                          
  +--> if bus has set 'close on exit'                                        
       |                                                                     
       |    +--------------+                                                 
       |--> | sd_bus_flush | ensure bus is running, flush bus wqueue         
       |    +--------------+                                                 
       |    +--------------+                                                 
       +--> | sd_bus_close | release sources/event/fd of bus
            +--------------+                                                 
```

```
src/libsystemd/sd-bus/sd-bus.c                                                      
+--------------+                                                                     
| sd_bus_flush | : ensure bus is running, flush bus wqueue                           
+-|------------+                                                                     
  |    +--------------------+                                                        
  |--> | bus_ensure_running | ensure bus is running                                  
  |    +--------------------+                                                        
  |                                                                                  
  +--> endless loop                                                                  
       |                                                                             
       |    +-----------------+                                                      
       |--> | dispatch_wqueue | for each msg in write queue: write to bus's output fd
       |    +-----------------+                                                      
       |                                                                             
       |--> if nothing left in wqueue, return                                        
       |                                                                             
       |    +----------+                                                             
       +--> | bus_poll | determine flags and timeout, then poll
            +----------+                                                             
```

```
src/libsystemd/sd-bus/sd-bus.c                                      
+--------------------+                                               
| bus_ensure_running | : ensure bus is running                       
+-|------------------+                                               
  |                                                                  
  |--> if it's already running, return                               
  |                                                                  
  +--> endless loop                                                  
       |                                                             
       |    +----------------+                                       
       |--> | sd_bus_process | given bus state, handle it accordingly
       |    +----------------+                                       
       |                                                             
       |--> if bus state == running, return                          
       |                                                             
       |--> if not running, but everything goes well, continue       
       |                                                             
       |    +-------------+                                          
       +--> | sd_bus_wait | (skip)                                   
            +-------------+                                          
```

```
src/libsystemd/sd-bus/sd-bus.c                                                                                                          
+----------------+                                                                                                                       
| sd_bus_process | : given bus state, handle it accordingly                                                                              
+----------------------+                                                                                                                 
| bus_process_internal | : given bus state, handle it accordingly                                                                        
+-|--------------------+                                                                                                                 
  |                                                                                                                                      
  +--> switch bus state                                                                                                                  
       case watch_bind                                                                                                                   
       -    +-------------------------------+                                                                                            
       +--> | bus_socket_process_watch_bind | create socket for input/output, connect, prepare io/inotify sources, register them to epoll
            +-------------------------------+                                                                                            
       case opening                                                                                                                      
       -    +----------------------------+                                                                                               
       +--> | bus_socket_process_opening | given bus address, prepare socket & connect, register sources to epoll                        
            +----------------------------+                                                                                               
       case authenticating                                                                                                               
       -    +-----------------------------------+                                                                                        
       +--> | bus_socket_process_authenticating | (skip)                                                                                 
            +-----------------------------------+                                                                                        
       case running/hello                                                                                                                
       -    +-----------------+                                                                                                          
       +--> | process_running | handle timeout, dispatch write queue, receive msg and process it                                         
            +-----------------+                                                                                                          
       case closing                                                                                                                      
       -    +-----------------+                                                                                                          
       +--> | process_closing | prepare msg of 'disconnected', close bus and exit                                                        
            +-----------------+                                                                                                          
```

```
src/libsystemd/sd-bus/bus-socket.c                                                                                                      
+-------------------------------+                                                                                                        
| bus_socket_process_watch_bind | : create socket for input/output, connect, prepare io source and inotify source, register them to epoll
+-|-----------------------------+                                                                                                        
  |    +----------+                                                                                                                      
  |--> | flush_fd | wait and read data to empty the queue                                                                                
  |    +----------+                                                                                                                      
  |    +--------------------+                                                                                                            
  |--> | bus_socket_connect | create one socket for both input/output, connect, close inotify                                            
  |    +--------------------+                                                                                                            
  |    +----------------------+                                                                                                          
  |--> | bus_attach_io_events | for input/output fd: prepare source and register to epoll & set source's priority/description            
  |    +----------------------+                                                                                                          
  |    +--------------------------+                                                                                                      
  +--> | bus_attach_inotify_event | prepare source and register to epoll & set source's priority/description                             
       +--------------------------+                                                                                                      
```

```
src/libsystemd/sd-bus/bus-socket.c                                                     
+--------------------+                                                                  
| bus_socket_connect | : create one socket for both input/output, connect, close inotify
+-|------------------+                                                                  
  |                                                                                     
  |--> endless loop                                                                     
  |    |                                                                                
  |    |--> create socket as input fd for bus                                           
  |    |                                                                                
  |    |    +------------------+                                                        
  |    |--> | bind_description | prepare bind name and bind to sock fd                  
  |    |    +------------------+                                                        
  |    |                                                                                
  |    |--> output fd = input fd                                                        
  |    |                                                                                
  |    |    +------------------+                                                        
  |    |--> | bus_socket_setup | set input/output buffer size to 8mb separately         
  |    |    +------------------+                                                        
  |    |    +---------+                                                                 
  |    |--> | connect |                                                                 
  |    |    +---------+                                                                 
  |    |                                                                                
  |    +--> if everything goes well, break                                              
  |                                                                                     
  |    +----------------------+                                                         
  +--> | bus_close_inotify_fd | we have socket now, don't need inotify anymore          
       +----------------------+                                                         
```

```
src/libsystemd/sd-bus/bus-socket.c                            
+------------------+                                           
| bind_description | : prepare bind name and bind to sock fd   
+-|----------------+                                           
  |    +------------------------+                              
  |--> | sd_bus_get_description | get bus description          
  |    +------------------------+                              
  |    +------------------+                                    
  |--> | get_process_comm | get process name                   
  |    +------------------+                                    
  |                                                            
  |--> assemble bind_name = /bus/$process_name/$bus_description
  |                                                            
  |    +----------------------+                                
  |--> | sockaddr_un_set_path | save bind_name in 'sock addr'  
  |    +----------------------+                                
  |    +------+                                                
  +--> | bind | bind 'sock addr' to fd                         
       +------+                                                
```

```
src/libsystemd/sd-bus/sd-bus.c                                                                                         
+----------------------+                                                                                                
| bus_attach_io_events | : for input/output fd: prepare source and register to epoll & set source's priority/description
+-|--------------------+                                                                                                
  |                                                                                                                     
  |--> if bus has no event source of input                                                                              
  |    |                                                                                                                
  |    |    +-----------------+                                                                                         
  |    +--> | sd_event_add_io | prepare 'source' and add to arg 'event, register the source's io                        
  |    |    +-----------------+                                                                                         
  |    |    +-----------------------------+                                                                             
  |    |--> | sd_event_source_set_prepare | update s->callback, insert to or remove from queue accordingly              
  |    |    +-----------------------------+                                                                             
  |    |    +------------------------------+                                                                            
  |    |--> | sd_event_source_set_priority | s->priority = priority, reposition source in data structure                
  |    |    +------------------------------+                                                                            
  |    |    +---------------------------------+                                                                         
  |    +--> | sd_event_source_set_description | duplicate description into source                                       
  |         +---------------------------------+                                                                         
  |                                                                                                                     
  |--> else                                                                                                             
  |    |                                                                                                                
  |    |    +---------------------------+                                                                               
  |    +--> | sd_event_source_set_io_fd | add new fd to epoll, remove the old one                                       
  |         +---------------------------+                                                                               
  |                                                                                                                     
  +--> if input fd != output fd                                                                                         
       |                                                                                                                
       |--> if bus has no event source of output                                                                        
       |    |                                                                                                           
       |    |    +-----------------+                                                                                    
       |    +--> | sd_event_add_io | prepare 'source' and add to arg 'event, register the source's io                   
       |    |    +-----------------+                                                                                    
       |    |    +------------------------------+                                                                       
       |    |--> | sd_event_source_set_priority | s->priority = priority, reposition source in data structure           
       |    |    +------------------------------+                                                                       
       |    |    +---------------------------------+                                                                    
       |    +--> | sd_event_source_set_description | duplicate description into source                                  
       |         +---------------------------------+                                                                    
       |                                                                                                                
       +--> else                                                                                                        
            |                                                                                                           
            |    +---------------------------+                                                                          
            +--> | sd_event_source_set_io_fd | add new fd to epoll, remove the old one                                  
                 +---------------------------+                                                                          
```

```
src/libsystemd/sd-event/sd-event.c                                                             
+-----------------------------+                                                                 
| sd_event_source_set_prepare | : update s->callback, insert to or remove from queue accordingly
+-|---------------------------+                                                                 
  |    +------------------------+                                                               
  |--> | prioq_ensure_allocated | ensure 'prepare' queue exists                                 
  |    +------------------------+                                                               
  |                                                                                             
  |--> install arg callback to source                                                           
  |                                                                                             
  |--> if callback is provided                                                                  
  |    |                                                                                        
  |    |    +-----------+                                                                       
  |    +--> | prioq_put | insert source to 'prepare' queue                                      
  |         +-----------+                                                                       
  |                                                                                             
  +--> else                                                                                     
       |                                                                                        
       |    +--------------+                                                                    
       +--> | prioq_remove | remove source from 'prepare' queue                                 
            +--------------+                                                                    
```

```
src/libsystemd/sd-event/sd-event.c                                                                       
+------------------------------+                                                                          
| sd_event_source_set_priority | : s->priority = priority, reposition source in data structure            
+-|----------------------------+                                                                          
  |                                                                                                       
  |--> if source type == inotify                                                                          
  |    |                                                                                                  
  |    |    +-------------------------+                                                                   
  |    |--> | event_make_inotify_data | ensure inotify_data exists and registered to epoll                
  |    |    +-------------------------+                                                                   
  |    |    +-----------------------+                                                                     
  |    |--> | event_make_inode_data | ensure inode_data in inotify_data's hashmap                         
  |    |    +-----------------------+                                                                     
  |    |                                                                                                  
  |    |--> move source from old inode_data to new inode_data                                             
  |    |                                                                                                  
  |    |    +--------------------------+                                                                  
  |    |--> | inode_data_realize_watch | add arg inode_data to inotify                                    
  |    |    +--------------------------+                                                                  
  |    |                                                                                                  
  |    |--> s->priority = arg priority                                                                    
  |    |                                                                                                  
  |    |    +---------------------+                                                                       
  |    +--> | event_gc_inode_data | garbage collect inode_data & inotify_data                             
  |         +---------------------+                                                                       
  |                                                                                                       
  |--> else if source type == signal                                                                      
  |    |                                                                                                  
  |    |--> s->priority = arg priority                                                                    
  |    |                                                                                                  
  |    |    +------------------------+                                                                    
  |    |--> | event_make_signal_data | ensure target signal is monitored by epoll                         
  |    |    +------------------------+                                                                    
  |    |    +--------------------------+                                                                  
  |    +--> | event_unmask_signal_data | unset the bit in mask                                            
  |         +--------------------------+                                                                  
  |                                                                                                       
  |--> else                                                                                               
  |    -                                                                                                  
  |    +--> s->priority = arg priority                                                                    
  |                                                                                                       
  |    +---------------------------------+                                                                
  |--> | event_source_pp_prioq_reshuffle | reshuffle source's 'pending' and 'prepare' queues if they exist
  |    +---------------------------------+                                                                
  |                                                                                                       
  +--> if source type == exit                                                                             
       |                                                                                                  
       |    +-----------------+                                                                           
       +--> | prioq_reshuffle | reshuffle target node in queue (exit)                                     
            +-----------------+                                                                           
```

```
src/libsystemd/sd-event/sd-event.c                                             
+-------------------------+                                                     
| event_make_inotify_data | : ensure inotify_data exists and registered to epoll
+-|-----------------------+                                                     
  |    +-------------+                                                          
  |--> | hashmap_get | given priority, get inotify_data from event hashmap      
  |    +-------------+                                                          
  |                                                                             
  |--> if got, return                                                           
  |                                                                             
  |    +---------------+                                                        
  |--> | inotify_init1 | get inotify fd                                         
  |    +---------------+                                                        
  |                                                                             
  |--> alloc inotify_data & set up                                              
  |                                                                             
  |    +--------------------+                                                   
  |--> | hashmap_ensure_put | insert inotify_data to event hashmap              
  |    +--------------------+                                                   
  |                                                                             
  |--> set up epoll_eventz                                                      
  |                                                                             
  |    +-----------+                                                            
  +--> | epoll_ctl | register fd to epoll                                       
       +-----------+                                                            
```

```
src/libsystemd/sd-event/sd-event.c                                         
+-----------------------+                                                   
| event_make_inode_data | : ensure inode_data in inotify_data's hashmap     
+-|---------------------+                                                   
  |                                                                         
  |--> set up key                                                           
  |                                                                         
  |    +-------------+                                                      
  |--> | hashmap_get | given key, get inode_data from inotify_data's hashmap
  |    +-------------+                                                      
  |                                                                         
  |--> if found, return                                                     
  |                                                                         
  |    +--------------------------+                                         
  |--> | hashmap_ensure_allocated | ensure inotify_data's hashmap exists    
  |    +--------------------------+                                         
  |                                                                         
  |--> alloc inode_data & set up                                            
  |                                                                         
  |    +-------------+                                                      
  +--> | hashmap_put | insert inode_data to inotify_data's hashmap          
       +-------------+                                                      
```

```
src/libsystemd/sd-event/sd-event.c                                     
+--------------------------+                                            
| inode_data_realize_watch | : add arg inode_data to inotify            
+-|------------------------+                                            
  |    +--------------------------+                                     
  |--> | hashmap_ensure_allocated | ensure inotify_data's hashmap exists
  |    +--------------------------+                                     
  |    +----------------------+                                         
  +--> | inotify_add_watch_fd | add arg inode_data to inotify           
       +----------------------+                                         
```

```
src/libsystemd/sd-event/sd-event.c                                               
+---------------------+                                                           
| event_gc_inode_data | : garbage collect inode_data & inotify_data               
+-|-------------------+                                                           
  |    +-----------------------+                                                  
  |--> | event_free_inode_data | remove inode_data from event/inotify, and free it
  |    +-----------------------+                                                  
  |    +-----------------------+                                                  
  +--> | event_gc_inotify_data | remove inotify_data from event, and free it      
       +-----------------------+                                                  
```

```
src/libsystemd/sd-bus/sd-bus.c                                                                         
+--------------------------+                                                                            
| bus_attach_inotify_event | : prepare source and register to epoll & set source's priority/description 
+-|------------------------+                                                                            
  |                                                                                                     
  |--> if bus has no inotify event source                                                               
  |    |                                                                                                
  |    |    +-----------------+                                                                         
  |    |--> | sd_event_add_io | prepare 'source' and add to arg 'event, register the source's io        
  |    |    +-----------------+                                                                         
  |    |    +------------------------------+                                                            
  |    |--> | sd_event_source_set_priority | s->priority = priority, reposition source in data structure
  |    |    +------------------------------+                                                            
  |    |    +---------------------------------+                                                         
  |    +--> | sd_event_source_set_description | duplicate description into source                       
  |         +---------------------------------+                                                         
  |                                                                                                     
  +--> else                                                                                             
       |                                                                                                
       |    +---------------------------+                                                               
       +--> | sd_event_source_set_io_fd | add new fd to epoll, remove the old one                       
            +---------------------------+                                                               
```

```
src/libsystemd/sd-bus/bus-socket.c                                                                                        
+----------------------------+                                                                                             
| bus_socket_process_opening | : given bus address, prepare socket & connect, register sources to epoll                    
+-|--------------------------+                                                                                             
  |    +-------------------+                                                                                               
  |--> | fd_wait_for_event | poll (wait for event)                                                                         
  |    +-------------------+                                                                                               
  |    +-----------------------+                                                                                           
  |--> | bus_socket_start_auth | (skip)                                                                                    
  |    +-----------------------+                                                                                           
  |    +------------------+                                                                                                
  +--> | bus_next_address | given bus address, spawn child if required, prepare socket & connect, register sources to epoll
       +------------------+                                                                                                
```

```
src/libsystemd/sd-bus/sd-bus.c                                                                                             
+------------------+                                                                                                        
| bus_next_address | : given bus address, spawn child if required, prepare socket & connect, register sources to epoll      
+-|----------------+                                                                                                        
  |    +--------------------------+                                                                                         
  |--> | bus_reset_parsed_address | reset bus structure                                                                     
  |    +--------------------------+                                                                                         
  |    +-------------------+                                                                                                
  +--> | bus_start_address | given bus address, spawn child if required, prepare socket & connect, register sources to epoll
       +-------------------+                                                                                                
```

```
src/libsystemd/sd-bus/sd-bus.c                                                                                                   
+-------------------+                                                                                                             
| bus_start_address | : given bus address, spawn child if required, prepare socket & connect, register sources to epoll           
+-|-----------------+                                                                                                             
  |                                                                                                                               
  +--> endless loop                                                                                                               
       |                                                                                                                          
       |    +------------------+                                                                                                  
       |--> | bus_close_io_fds | detach event (source->enabled = off, ref--), close input/output fd                               
       |    +------------------+                                                                                                  
       |    +----------------------+                                                                                              
       |--> | bus_close_inotify_fd | release inotify source, close fd                                                             
       |    +----------------------+                                                                                              
       |    +---------------+                                                                                                     
       |--> | bus_kill_exec | send 'terminate' signal to target task, wait till it exits                                          
       |    +---------------+                                                                                                     
       |                                                                                                                          
       |--> if exec path is provided                                                                                              
       |    |                                                                                                                     
       |    |    +-----------------+                                                                                              
       |    +--> | bus_socket_exec | fork and exec child, set up input/output buffer size (8m)                                    
       |         +-----------------+                                                                                              
       |                                                                                                                          
       |--> elif target is in another namespace (???)                                                                             
       |    |                                                                                                                     
       |    |    +------------------------------+                                                                                 
       |    +--> | bus_container_connect_socket | fork a task into target namespace, have it connect to bus's input fd
       |         +------------------------------+                                                                                 
       |                                                                                                                          
       |--> elif socket addr is specified                                                                                         
       |    |                                                                                                                     
       |    |    +--------------------+                                                                                           
       |    +--> | bus_socket_connect | create one socket for both input/output, connect, close inotify                           
       |         +--------------------+                                                                                           
       |                                                                                                                          
       |--> else, go to 'next'                                                                                                    
       |                                                                                                                          
       |    +----------------------+                                                                                              
       |--> | bus_attach_io_events | for input/output fd: prepare source and register to epoll & set source's priority/description
       |    +----------------------+                                                                                              
       |    +--------------------------+                                                                                          
       |--> | bus_attach_inotify_event | prepare source and register to epoll & set source's priority/description                 
       |    +--------------------------+                                                                                          
       |                                                                                                                          
       |--> return                                                                                                                
       |next:                                                                                                                     
       |    +------------------------+                                                                                            
       +--> | bus_parse_next_address | parse address, save to sock_addr in bus                                                    
            +------------------------+                                                                                            
```

```
src/libsystemd/sd-bus/bus-socket.c                                            
+-----------------+                                                            
| bus_socket_exec | : fork and exec child, set up input/output buffer size (8m)
+-|---------------+                                                            
  |    +------------+                                                          
  |--> | socketpair | create socket pair that connect to each other            
  |    +------------+                                                          
  |    +----------------+                                                      
  |--> | safe_fork_full | fork                                                 
  |    +----------------+                                                      
  |                                                                            
  |--> if I'm the child                                                        
  |    |                                                                       
  |    |    +--------+    +--------+                                           
  |    +--> | execvp | or | execvp |                                           
  |         +--------+    +--------+                                           
  |                                                                            
  |--> if I'm the parent, close socket[1]                                      
  |                                                                            
  |    +------------------+                                                    
  +--> | bus_socket_setup | set bus's input/output buffer to 8mb each          
       +------------------+                                                    
```

```
src/libsystemd/sd-bus/bus-container.c                                                                        
+------------------------------+                                                                              
| bus_container_connect_socket | : fork a task into target namespace, have it connect to bus's input fd       
+-|----------------------------+                                                                              
  |    +----------------+                                                                                     
  |--> | namespace_open |                                                                                     
  |    +----------------+                                                                                     
  |                                                                                                           
  |--> create socket for input fd                                                                             
  |                                                                                                           
  |--> output fd = input fd                                                                                   
  |                                                                                                           
  |    +------------------+                                                                                   
  |--> | bus_socket_setup | config send/receive buffer size                                                   
  |    +------------------+                                                                                   
  |    +------------+                                                                                         
  |--> | socketpair |                                                                                         
  |    +------------+                                                                                         
  |    +----------------+                                                                                     
  |--> | namespace_fork | parent forks child, child enters target ns & forks grandchild & waits for its finish
  |    +----------------+                                                                                     
  |                                                                                                           
  |--> if we are the grandchild                                                                               
  |    |                                                                                                      
  |    |--> close socker pair[0]                                                                              
  |    |                                                                                                      
  |    +--> connect bus's input fd and exit (why?)                                                            
  |                                                                                                           
  |    (parent reaches here)                                                                                  
  |                                                                                                           
  |--> close socket pair[1]                                                                                   
  |                                                                                                           
  |    +------------------------------+                                                                       
  |--> | wait_for_terminate_and_check | wait till grandchild finishes                                         
  |    +------------------------------+                                                                       
  |    +-----------------------+                                                                              
  +--> | bus_socket_start_auth | (skip)                                                                       
       +-----------------------+                                                                              
```

```
src/libsystemd/sd-bus/bus-container.c                                                                   
+----------------+                                                                                       
| namespace_fork | : parent forks child, child enters target ns & forks grandchild & waits for its finish
+-|--------------+                                                                                       
  |    +----------------+                                                                                
  |--> | safe_fork_full | fork, parent returns immediatly, child handles flags properly                  
  |    +----------------+                                                                                
  |                                                                                                      
  +--> if we are the forked child                                                                        
       |                                                                                                 
       |    +----------------+                                                                           
       +--> | safe_fork_full | fork, parent returns immediatly, child handles flags properly             
       |    +----------------+                                                                           
       |                                                                                                 
       |--> if we are the forked grandchild                                                              
       |    -                                                                                            
       |    +--> return pid thru arg, return                                                             
       |                                                                                                 
       |    (child reaches here)                                                                         
       |                                                                                                 
       |    +------------------------------+                                                             
       +--> | wait_for_terminate_and_check | grandchild exits                                            
       |    +------------------------------+                                                             
       |    +-------+                                                                                    
       +--> | _exit | child exits                                                                        
            +-------+                                                                                    
```

```
src/basic/process-util.c                                                         
+----------------+                                                                
| safe_fork_full | : fork, parent returns immediatly, child handles flags properly
+-|--------------+                                                                
  |                                                                               
  |--> given flags, determine signal mask for block                               
  |                                                                               
  |    +-------------+                                                            
  |--> | sigprocmask | block                                                      
  |    +-------------+                                                            
  |                                                                               
  |--> if flag has 'detach'                                                       
  |    |                                                                          
  |    |    +-------------------+                                                 
  |    |--> | is_reaper_process | check if we are reaper task (pid = 1)           
  |    |    +-------------------+                                                 
  |    |                                                                          
  |    +--> if we are not the reaper                                              
  |         |                                                                     
  |         |    +------+                                                         
  |         |--> | fork |                                                         
  |         |    +------+                                                         
  |         |                                                                     
  |         +--> if we are the parent return                                      
  |                                                                               
  |    +------+                                                                   
  |--> | fork |                                                                   
  |    +------+                                                                   
  |                                                                               
  |--> if we are child && the middle task                                         
  |    -                                                                          
  |    +--> exit to let the grandchild be reaped                                  
  |                                                                               
  |--> if we are the parent                                                       
  |    -                                                                          
  |    +--> if flag has 'wait'                                                    
  |    |    |                                                                     
  |    |    |    +------------------------------+                                 
  |    |    +--> | wait_for_terminate_and_check | wait for child to terminate     
  |    |         +------------------------------+                                 
  |    |                                                                          
  |    +--> return                                                                
  |                                                                               
  |--> (only child task reaches here)                                             
  |                                                                               
  +--> handle flags accordingly                                                   
```

```
src/libsystemd/sd-bus/sd-bus.c                                     
+------------------------+                                          
| bus_parse_next_address | : parse address, save to sock_addr in bus
+-|----------------------+                                          
  |    +--------------------------+                                 
  |--> | bus_reset_parsed_address | reset bus address               
  |    +--------------------------+                                 
  |                                                                 
  |--> get first address                                            
  |                                                                 
  +--> while address is still valid                                 
       |                                                            
       |--> if it starts with 'unix:'                               
       |    |                                                       
       |    |    +--------------------+                             
       |    |--> | parse_unix_address |                             
       |    |    +--------------------+                             
       |    +--> break                                              
       |                                                            
       |--> elif it starts with 'tcp:'                              
       |    |                                                       
       |    |    +-------------------+                              
       |    |--> | parse_tcp_address |                              
       |    |    +-------------------+                              
       |    +--> break                                              
       |                                                            
       |--> elif it starts with 'unixexec:'                         
       |    |                                                       
       |    |    +--------------------+                             
       |    |--> | parse_exec_address |                             
       |    |    +--------------------+                             
       |    +--> break                                              
       |                                                            
       |--> elif it starts with 'x-machine-unix:'                   
       |    |                                                       
       |    |    +------------------------------+                   
       |    |--> | parse_container_unix_address |                   
       |    |    +------------------------------+                   
       |    +--> break                                              
       |                                                            
       +--> advance to next address                                 
```

```
src/libsystemd/sd-bus/sd-bus.c                                                                             
+-----------------+                                                                                         
| process_running | : handle timeout, dispatch write queue, receive msg and process it                      
+-|---------------+                                                                                         
  |    +-----------------+                                                                                  
  |--> | process_timeout | remove 1st reply_callback from bus, prepare msg and set sender, call ->callback()
  |    +-----------------+                                                                                  
  |    +-----------------+                                                                                  
  |--> | dispatch_wqueue | for each msg in write queue: write to bus's output fd                            
  |    +-----------------+                                                                                  
  |    +----------------+                                                                                   
  |--> | dispatch_track | (skip)                                                                            
  |    +----------------+                                                                                   
  |    +-----------------+                                                                                  
  |--> | dispatch_rqueue | ensure rqueue has msg, get 1st msg and remove it from rqueue                     
  |    +-----------------+                                                                                  
  |    +-----------------+                                                                                  
  +--> | process_message | process callbacks of reply/filter/match, process method                          
       +-----------------+                                                                                  
```

```
src/libsystemd/sd-bus/sd-bus.c                                                                        
+-----------------+                                                                                    
| process_timeout | : remove 1st reply_callback from bus, prepare msg and set sender, call ->callback()
+-|---------------+                                                                                    
  |    +------------+                                                                                  
  |--> | prioq_peek | get first callback from bus prio queue                                           
  |    +------------+                                                                                  
  |    +---------------------------------+                                                             
  |--> | bus_message_new_synthetic_error | prepare msg, append information, set msg sender             
  |    +---------------------------------+                                                             
  |    +----------------------------+                                                                  
  |--> | bus_seal_synthetic_message | finish the msg build                                             
  |    +----------------------------+                                                                  
  |    +------------------------+                                                                      
  |--> | ordered_hashmap_remove | remove callback from bus                                             
  |    +------------------------+                                                                      
  |                                                                                                    
  |--> set up bus fields for callback                                                                  
  |                                                                                                    
  |--> call ->callback(), e.g.,                                                                        
  |    +----------------+                                                                              
  |    | hello_callback | read data from reply, change bus state from 'hello' to 'running'             
  |    +--------------------+                                                                          
  |    | add_match_callback | call ->install_callback()                                                
  |    +--------------------+                                                                          
  |                                                                                                    
  +--> reset those bus fields                                                                          
```

```
src/libsystemd/sd-bus/bus-message.c                                                 
+---------------------------------+                                                  
| bus_message_new_synthetic_error | : prepare msg, append information, set msg sender
+-|-------------------------------+                                                  
  |    +--------------------+                                                        
  |--> | sd_bus_message_new | prepare msg                                            
  |    +--------------------+                                                        
  |    +-----------------------------+                                               
  |--> | message_append_reply_cookie | append 'reply cookie' to msg                  
  |    +-----------------------------+                                               
  |                                                                                  
  |--> if bus has unique name                                                        
  |    |                                                                             
  |    |    +-----------------------------+                                          
  |    +--> | message_append_field_string | append 'unique name' to msg              
  |         +-----------------------------+                                          
  |    +-----------------------------+                                               
  |--> | message_append_field_string | append 'error name' to msg                    
  |    +-----------------------------+                                               
  |                                                                                  
  |--> if error has msg                                                              
  |    |                                                                             
  |    |    +-----------------------------+                                          
  |    +--> | message_append_field_string | append 'error msg' to msg                
  |         +-----------------------------+                                          
  |    +-------------------------------+                                             
  +--> | bus_message_set_sender_driver | set msg sender = "org.freedesktop.DBus"     
       +-------------------------------+                                             
```

```
src/libsystemd/sd-bus/sd-bus.c                                                                
+----------------+                                                                             
| hello_callback | : read data from reply, change bus state from 'hello' to 'running'          
+-|--------------+                                                                             
  |    +---------------------+                                                                 
  |--> | sd_bus_message_read | read data (type = string) from reply                            
  |    +---------------------+                                                                 
  |                                                                                            
  +--> if bus state == hello                                                                   
       |                                                                                       
       |    +---------------+                                                                  
       |--> | bus_set_state | set state = arg running                                          
       |    +---------------+                                                                  
       |    +-----------------------------+                                                    
       +--> | synthesize_connected_signal | prepare msg of signal type, insert to bus rqueue[0]
            +-----------------------------+                                                    
```

```
src/libsystemd/sd-bus/sd-bus.c                                                                   
+-----------------------------+                                                                   
| synthesize_connected_signal | : prepare msg of signal type, insert to bus rqueue[0]             
+-|---------------------------+                                                                   
  |    +---------------------------+                                                              
  |--> | sd_bus_message_new_signal | prepare msg of signal type, append path/interface/member info
  |    +---------------------------+                                                              
  |    +------------------------------+                                                           
  |--> | bus_message_set_sender_local | set msg sender = org.freedesktop.DBus.Local               
  |    +------------------------------+                                                           
  |    +----------------------------+                                                             
  |--> | bus_seal_synthetic_message | finish msg build                                            
  |    +----------------------------+                                                             
  |    +----------------------+                                                                   
  |--> | bus_rqueue_make_room | realloc bus rqueue (msg array)                                    
  |    +----------------------+                                                                   
  |                                                                                               
  +--> insert msg to rqueue[0]                                                                    
```

```
src/libsystemd/sd-bus/bus-message.c                                                            
+---------------------------+                                                                   
| sd_bus_message_new_signal | : prepare msg of signal type, append path/interface/member info   
+------------------------------+                                                                
| sd_bus_message_new_signal_to | : prepare msg of signal type, append path/interface/member info
+-|----------------------------+                                                                
  |    +--------------------+                                                                   
  |--> | sd_bus_message_new | prepare msg of signal type                                        
  |    +--------------------+                                                                   
  |    +-----------------------------+                                                          
  |--> | message_append_field_string | append 'path' to msg                                     
  |    +-----------------------------+                                                          
  |    +-----------------------------+                                                          
  |--> | message_append_field_string | append 'interface' to msg                                
  |    +-----------------------------+                                                          
  |    +-----------------------------+                                                          
  |--> | message_append_field_string | append 'member' to msg                                   
  |    +-----------------------------+                                                          
  |                                                                                             
  +--> if arg destination is provided                                                           
       |                                                                                        
       |    +-----------------------------+                                                     
       +--> | message_append_field_string | append 'destination' to msg                         
            +-----------------------------+                                                     
```

```
src/libsystemd/sd-bus/sd-bus.c                                                 
+-----------------+                                                             
| dispatch_wqueue | : for each msg in write queue: write to bus's output fd     
+-|---------------+                                                             
  |                                                                             
  +--> while wqueue_size > 0                                                    
       |                                                                        
       |    +-------------------+                                               
       |--> | bus_write_message | convert msg to iovec, write to bus's output fd
       |    +-------------------+                                               
       |                                                                        
       +--> remove the msg rom bus write queue                                  
```

```
src/libsystemd/sd-bus/sd-bus.c                                              
+-------------------+                                                        
| bus_write_message | : convert msg to iovec, write to bus's output fd       
+--------------------------+                                                 
| bus_socket_write_message | : convert msg to iovec, write to bus's output fd
+-|------------------------+                                                 
  |    +-------------------------+                                           
  |--> | bus_message_setup_iovec | convert msg to iovec                      
  |    +-------------------------+                                           
  |                                                                          
  +--> either writev() or sendmsg() to bus output fd                         
```

```
src/libsystemd/sd-bus/bus-socket.c               
+-------------------------+                       
| bus_message_setup_iovec | : convert msg to iovec
+-|-----------------------+                       
  |                                               
  |--> determine iovec buffer                     
  |                                               
  |    +--------------+                           
  |--> | append_iovec | append one vector         
  |    +--------------+                           
  |                                               
  +--> for each part in msg                       
       |                                          
       |    +-------------------+                 
       |--> | bus_body_part_map | perform mmap    
       |    +-------------------+                 
       |    +--------------+                      
       +--> | append_iovec | append one vector    
            +--------------+                      
```

```
src/libsystemd/sd-bus/sd-bus.c                                                                                     
+-----------------+                                                                                                 
| dispatch_rqueue | : ensure rqueue has msg, get 1st msg and remove it from rqueue                                  
+-|---------------+                                                                                                 
  |                                                                                                                 
  +--> endless loop                                                                                                 
       |                                                                                                            
       |--> if there's already something in rqueue                                                                  
       |    |                                                                                                       
       |    |--> get rqueue[0]                                                                                      
       |    |                                                                                                       
       |    |    +-----------------+                                                                                
       |    |--> | rqueue_drop_one | remove target msg from rqueue                                                  
       |    |    +-----------------+                                                                                
       |    +--> return                                                                                             
       |                                                                                                            
       |    +------------------+                                                                                    
       +--> | bus_read_message | prepare rbuffer, read data from bus's input fd, convert to msg and append to rqueue
            +------------------+                                                                                    
```

```
src/libsystemd/sd-bus/sd-bus.c                                                                                  
+------------------+                                                                                             
| bus_read_message | : prepare rbuffer, read data from bus's input fd, convert to msg and append to rqueue       
+-------------------------+                                                                                      
| bus_socket_read_message | : prepare rbuffer, read data from bus's input fd, convert to msg and append to rqueue
+-|-----------------------+                                                                                      
  |    +------------------------------+                                                                          
  |--> | bus_socket_read_message_need | determine how much size is needed                                        
  |    +------------------------------+                                                                          
  |                                                                                                              
  |--> if bus has enough size for msg                                                                            
  |    |                                                                                                         
  |    |    +-------------------------+                                                                          
  |    |--> | bus_socket_make_message | alloc msg & parse buffer, append to bus rqueue                           
  |    |    +-------------------------+                                                                          
  |    +--> return                                                                                               
  |                                                                                                              
  |    +---------+                                                                                               
  |--> | realloc | realloc bus rbuffer                                                                           
  |    +---------+                                                                                               
  |                                                                                                              
  |--> either readv() or recvmsg_safe()                                                                          
  |                                                                                                              
  |    +------------------------------+                                                                          
  |--> | bus_socket_read_message_need | determine how much size is needed                                        
  |    +------------------------------+                                                                          
  |                                                                                                              
  +--> if bus has enough size for msg                                                                            
       |                                                                                                         
       |    +-------------------------+                                                                          
       |--> | bus_socket_make_message | alloc msg & parse buffer, append to bus rqueue                           
       |    +-------------------------+                                                                          
       +--> return                                                                                               
```

```
src/libsystemd/sd-bus/bus-socket.c                                         
+-------------------------+                                                 
| bus_socket_make_message | : alloc msg & parse buffer, append to bus rqueue
+-|-----------------------+                                                 
  |    +----------------------+                                             
  |--> | bus_rqueue_make_room | realloc bus's rqueue                        
  |    +----------------------+                                             
  |    +-------------------------+                                          
  |--> | bus_message_from_malloc | alloc msg, and parse buffer              
  |    +-------------------------+                                          
  |                                                                         
  +--> add msg to the last position of bus rqueue                           
```

```
src/libsystemd/sd-bus/sd-bus.c                                                                         
+-----------------+                                                                                     
| process_message | : process callbacks of reply/filter/match, process method                           
+-|---------------+                                                                                     
  |    +---------------+                                                                                
  |--> | process_hello | simple check                                                                   
  |    +---------------+                                                                                
  |    +---------------+                                                                                
  |--> | process_reply | given msg cookie, remove target 'reply callback' from bus and call ->callback()
  |    +---------------+                                                                                
  |    +------------------+                                                                             
  |--> | process_fd_check | check fd                                                                    
  |    +------------------+                                                                             
  |    +----------------+                                                                               
  |--> | process_filter | for each filter callback in bus: call ->callback()                            
  |    +----------------+                                                                               
  |    +---------------+                                                                                
  |--> | process_match | recursively traverse children/sibling, if leaf: run ->callback()               
  |    +---------------+                                                                                
  |    +-----------------+                                                                              
  |--> | process_builtin | if necessary, prepare msg and reply                                          
  |    +-----------------+                                                                              
  |    +--------------------+                                                                           
  +--> | bus_process_object | handle method get/set/get_all/introspect/get_managed_objects              
       +--------------------+                                                                           
```

```
src/libsystemd/sd-bus/sd-bus.c                                                                    
+---------------+                                                                                  
| process_reply | : given msg cookie, remove target 'reply callback' from bus and call ->callback()
+-|-------------+                                                                                  
  |                                                                                                
  |--> given msg cookie (key), remove 'reply callback' from bus hashmap                            
  |                                                                                                
  |--> given 'reply callback', get outer 'slot'                                                    
  |                                                                                                
  |    +-----------------------+                                                                   
  |--> | sd_bus_message_rewind | (???)                                                             
  |    +-----------------------+                                                                   
  |                                                                                                
  |--> call ->callback()                                                                           
  |                                                                                                
  +--> slot ref--                                                                                  
```

```
src/libsystemd/sd-bus/sd-bus.c                                        
+----------------+                                                     
| process_filter | : for each filter callback in bus: call ->callback()
+-|--------------+                                                     
  |                                                                    
  +--> for each filter callback in bus                                 
       |                                                               
       |    +-----------------------+                                  
       |--> | sd_bus_message_rewind |                                  
       |    +-----------------------+                                  
       |                                                               
       +--> call ->callback()                                          
```

```
src/libsystemd/sd-bus/sd-bus.c                                                     
+---------------+                                                                   
| process_match | : recursively traverse children/sibling, if leaf: run ->callback()
+---------------+                                                                   
| bus_match_run | : recursively traverse children/sibling, if leaf: run ->callback()
+-|-------------+                                                                   
  |                                                                                 
  |--> switch node type                                                             
  |--> case root/value                                                              
  |    -    +---------------+                                                       
  |    +--> | bus_match_run | recursively apply to child                            
  |         +---------------+                                                       
  |--> case leaf                                                                    
  |    |--> run ->callback()                                                        
  |    |    +---------------+                                                       
  |    +--> | bus_match_run | recursively apply to sibling (next)                   
  |         +---------------+                                                       
  |--> case type/sender/destination/...                                             
  |    +--> get info from header                                                    
  |--> case args_xxx                                                                
  |    -    +---------------------+                                                 
  |    +--> | bus_message_get_arg |                                                 
  |         +---------------------+                                                 
  |                                                                                 
  |--> if node type is in hash table                                                
  |    |                                                                            
  |    |    +-------------+                                                         
  |    |--> | hashmap_get |                                                         
  |    |    +-------------+                                                         
  |    |    +---------------+                                                       
  |    +--> | bus_match_run | recursively apply to value found from hash table      
  |         +---------------+                                                       
  |                                                                                 
  |--> for each child of current node                                               
  |    |                                                                            
  |    |    +---------------+                                                       
  |    +--> | bus_match_run | recursively apply to current child                    
  |         +---------------+                                                       
  |    +---------------+                                                            
  +--> | bus_match_run | recursively apply to sibling (next)                        
       +---------------+                                                            
```

```
src/libsystemd/sd-bus/bus-objects.c                                                            
+--------------------+                                                                          
| bus_process_object | ： handle method get/set/get_all/introspect/get_managed_objects           
+-|------------------+                                                                          
  |                                                                                             
  |--> if it's broadcast msg, return (ignore)                                                   
  |                                                                                             
  |--> alloc buffer for prefix                                                                  
  |                                                                                             
  |    +---------------------+                                                                  
  |--> | object_find_and_run | handle method get/set/get_all/introspect/get_managed_objects     
  |    +---------------------+                                                                  
  |                                                                                             
  +--> for each prefix in msg path                                                              
       |                                                                                        
       |    +---------------------+                                                             
       +--> | object_find_and_run | handle method get/set/get_all/introspect/get_managed_objects
            +---------------------+                                                             
```

```
src/libsystemd/sd-bus/bus-objects.c                                                        
+---------------------+                                                                     
| object_find_and_run | : handle method get/set/get_all/introspect/get_managed_objects      
+-|-------------------+                                                                     
  |                                                                                         
  |--> given arg key, get node from bus's hashmap                                           
  |                                                                                         
  |    +--------------------+                                                               
  |--> | node_callbacks_run | for each callback in node, call ->callback()                  
  |    +--------------------+                                                               
  |                                                                                         
  |--> set up vtable key, get method from bus's another hashmap                             
  |                                                                                         
  |    +----------------------+                                                             
  |--> | method_callbacks_run | get user data and signature, call method.handler()          
  |    +----------------------+                                                             
  |                                                                                         
  |--> if interface == org.freedesktop.DBus.Properties                                      
  |    |                                                                                    
  |    |--> if member is 'get' or 'set'                                                     
  |    |    |                                                                               
  |    |    |    +---------------------+                                                    
  |    |    |--> | sd_bus_message_read | read interface/member from msg to set up vtable key
  |    |    |    +---------------------+                                                    
  |    |    |                                                                               
  |    |    |--> given key, get properties from bus's hashmap                               
  |    |    |                                                                               
  |    |    |    +--------------------------------+                                         
  |    |    +--> | property_get_set_callbacks_run | (skip)                                  
  |    |         +--------------------------------+                                         
  |    |                                                                                    
  |    +--> elif member is 'get all'                                                        
  |         |                                                                               
  |         |    +---------------------+                                                    
  |         |--> | sd_bus_message_read | read interface from msg                            
  |         |    +---------------------+                                                    
  |         |    +--------------------------------+                                         
  |         +--> | property_get_all_callbacks_run | (skip)                                  
  |              +--------------------------------+                                         
  |                                                                                         
  |--> elif method == 'introspect'                                                          
  |    |                                                                                    
  |    |    +--------------------+                                                          
  |    +--> | process_introspect | (skip)                                                   
  |         +--------------------+                                                          
  |                                                                                         
  +--> elif method == 'get managed objects'                                                 
       |                                                                                    
       |    +-----------------------------+                                                 
       +--> | process_get_managed_objects | (skip)                                          
            +-----------------------------+                                                 
```

```
src/libsystemd/sd-bus/bus-objects.c                                              
+----------------------+                                                          
| method_callbacks_run | : get user data and signature, call method.handler()     
+-|--------------------+                                                          
  |    +--------------------------+                                               
  |--> | node_vtable_get_userdata | call ->find() to get user data                
  |    +--------------------------+                                               
  |    +--------------------------------+                                         
  |--> | vtable_method_convert_userdata | += offset                               
  |    +--------------------------------+                                         
  |    +------------------------------+                                           
  |--> | sd_bus_message_get_signature | determine container, get signature from it
  |    +------------------------------+                                           
  |                                                                               
  +--> call method.handler()                                                      
```

```
src/libsystemd/sd-bus/bus-objects.c                                                                              
+--------------------------------+                                                                                
| property_get_set_callbacks_run | : prepare msg of 'method return' (method is get or set), send msg out          
+-|------------------------------+                                                                                
  |    +------------------------------+                                                                           
  |--> | vtable_property_get_userdata | call ->find() to get user data                                            
  |    +------------------------------+                                                                           
  |                                                                                                               
  |--> given vtable parent, get outer slot                                                                        
  |                                                                                                               
  |    +----------------------------------+                                                                       
  |--> | sd_bus_message_new_method_return | prepare msg of 'method return'                                        
  |    +----------------------------------+                                                                       
  |                                                                                                               
  |--> if arg is_get == 1                                                                                         
  |    |                                                                                                          
  |    |    +-------------------------------+                                                                     
  |    |--> | sd_bus_message_open_container | ensure contents in signature, set up container for it and add to msg
  |    |    +-------------------------------+                                                                     
  |    |    +---------------------+                                                                               
  |    |--> | invoke_property_get | get property and append to msg                                                
  |    |    +---------------------+                                                                               
  |    |    +--------------------------------+                                                                    
  |    +--> | sd_bus_message_close_container | close container and free signature                                 
  |         +--------------------------------+                                                                    
  |                                                                                                               
  |--> else                                                                                                       
  |    |                                                                                                          
  |    |    +--------------------------+                                                                          
  |    |--> | sd_bus_message_peek_type | peek type (and signature)                                                
  |    |    +--------------------------+                                                                          
  |    |    +--------------------------------+                                                                    
  |    |--> | sd_bus_message_enter_container | enter container, set up another container in msg                   
  |    |    +--------------------------------+                                                                    
  |    |    +---------------------+                                                                               
  |    |--> | invoke_property_set | read msg and return user data (where's the set action???)                     
  |    |    +---------------------+                                                                               
  |    |    +-------------------------------+                                                                     
  |    +--> | sd_bus_message_exit_container | free last container in msg                                          
  |         +-------------------------------+                                                                     
  |    +-------------+                                                                                            
  +--> | sd_bus_send | seal, send msg out or append to wqueue                                                     
       +-------------+                                                                                            
```

```
src/libsystemd/sd-bus/bus-objects.c                                       
+----------------------------------+                                       
| sd_bus_message_new_method_return | : prepare msg of 'method return'      
+-------------------+--------------+                                       
| message_new_reply | : prepare msg of arg type                            
+-|-----------------+                                                      
  |    +--------------------+                                              
  |--> | sd_bus_message_new | prepare msg of arg type (e.g., method_return)
  |    +--------------------+                                              
  |    +-----------------------------+                                     
  |--> | message_append_reply_cookie | append 'reply cookie' to msg        
  |    +-----------------------------+                                     
  |                                                                        
  +--> if call sender is set                                               
       |                                                                   
       |    +-----------------------------+                                
       +--> | message_append_field_string | append 'destination' to msg    
            +-----------------------------+                                
```

```
src/libsystemd/sd-bus/bus-message.c                                                                    
+-------------------------------+                                                                       
| sd_bus_message_open_container | : ensure contents in signature, set up container for it and add to msg
+-|-----------------------------+                                                                       
  |                                                                                                     
  |--> increase container array size                                                                    
  |                                                                                                     
  |    +----------------------------+                                                                   
  |--> | message_get_last_container | get last container from msg                                       
  |    +----------------------------+                                                                   
  |                                                                                                     
  |--> if type == array                                                                                 
  |    |                                                                                                
  |    |    +------------------------+                                                                  
  |    +--> | bus_message_open_array | ensure signature contains contents                               
  |         +------------------------+                                                                  
  |                                                                                                     
  |--> elif itype == variant                                                                            
  |    |                                                                                                
  |    |    +--------------------------+                                                                
  |    +--> | bus_message_open_variant | extend msg body and copy contents to it                        
  |         +--------------------------+                                                                
  |                                                                                                     
  |--> elif itype == struct                                                                             
  |    |                                                                                                
  |    |    +-------------------------+                                                                 
  |    +--> | bus_message_open_struct | ensure signature contains contents                              
  |         +-------------------------+                                                                 
  |                                                                                                     
  |--> elif itype == dict entry                                                                         
  |    |                                                                                                
  |    |    +-----------------------------+                                                             
  |    +--> | bus_message_open_dict_entry | (expect that signature contains contents already)           
  |         +-----------------------------+                                                             
  |                                                                                                     
  +--> prepare a container of the contents and add to msg (end of container array)                      
```

```
src/libsystemd/sd-bus/bus-message.c                              
+------------------------+                                        
| bus_message_open_array | : ensure signature contains contents   
+-|----------------------+                                        
  |                                                               
  |--> if signature contains contents already, update end index   
  |                                                               
  |                                                               
  +--> else                                                       
  |    |                                                          
  |    |    +-----------+                                         
  |    +--> | strextend | extend signature and copy contents to it
  |         +-----------+                                         
  |    +---------------------+                                    
  |    | message_extend_body | extend msg body                    
  |    +---------------------+                                    
  |                                                               
  +--> if contaoiner enclosing != array                           
       -                                                          
       +--> save end index in container                           
```

```
src/libsystemd/sd-bus/bus-message.c                                  
+--------------------------+                                          
| bus_message_open_variant | : extend msg body and copy contents to it
+-|------------------------+                                          
  |    +--------------------+                                         
  |--> | message_extend_body|                                         
  |    +--------------------+                                         
  |    +--------+                                                     
  +--> | memcpy | copy contents to msg body                           
       +--------+                                                     
```

```
src/libsystemd/sd-bus/bus-message.c                             
+-------------------------+                                      
| bus_message_open_struct | : ensure signature contains contents 
+-|-----------------------+                                      
  |                                                              
  |--> if signature already contains contents, update end index  
  |                                                              
  |--> else                                                      
  |    |                                                         
  |    |    +-----------+                                        
  |    +--> | strextend | extend signatue and copy contents to it
  |         +-----------+                                        
  |                                                              
  +--> if container enclosing != array                           
       -                                                         
       +--> save end index in container                          
```

```
src/libsystemd/sd-bus/bus-message.c                            
+--------------------------+                                    
| sd_bus_message_peek_type | : peek type (and signature)        
+-|------------------------+                                    
  |    +----------------------------+                           
  +--> | message_get_last_container | get msg's last container  
  |    +----------------------------+                           
  |                                                             
  |--> peek signature, if it's basic type, return type to caller
  |                                                             
  |--> peek signature, if it's array                            
  |    -                                                        
  |    +--> peek further, return type/content to caller         
  |                                                             
  |--> peek signature, if it's part of struct or dict           
  |    -                                                        
  |    +--> peek further, return type/content to caller         
  |                                                             
  +--> peek signature, if it's variant                          
       -                                                        
       +--> peek msg body, return type/content to caller        
```

```
src/libsystemd/sd-bus/bus-objects.c                                                 
+--------------------------------+                                                   
| sd_bus_message_enter_container | : enter container, set up another container in msg
+-|------------------------------+                                                   
  |    +----------------------------+                                                
  |--> | message_get_last_container |                                                
  |    +----------------------------+                                                
  |                                                                                  
  |--> if type == array                                                              
  |    |                                                                             
  |    |    +-------------------------+                                              
  |    +--> | bus_message_enter_array | peek msg body to get rindex and array size   
  |         +-------------------------+                                              
  |                                                                                  
  |--> elif type == variant                                                          
  |    |                                                                             
  |    |    +---------------------------+                                            
  |    +--> | bus_message_enter_variant | peek msg body to get rindex                
  |         +---------------------------+                                            
  |                                                                                  
  |--> elif type == struct                                                           
  |    |                                                                             
  |    |    +--------------------------+                                             
  |    +--> | bus_message_enter_struct | peek msg body to get rindex                 
  |         +--------------------------+                                             
  |                                                                                  
  |--> elif type == dict entry                                                       
  |    |                                                                             
  |    |    +------------------------------+                                         
  |    +--> | bus_message_enter_dict_entry | peek msg body to get rindex             
  |         +------------------------------+                                         
  |                                                                                  
  +--> set up container in msg (end of container array)                              
```

```
src/libsystemd/sd-bus/sd-bus.c                                                 
+-------------+                                                                 
| sd_bus_send | : seal, send msg out or append to wqueue                        
+-|-----------+                                                                 
  |    +------------------+                                                     
  |--> | bus_seal_message |                                                     
  |    +------------------+                                                     
  |    +-----------------------+                                                
  |--> | bus_remarshal_message | if necessary, remarshal msg                    
  |    +-----------------------+                                                
  |                                                                             
  |--> if no reply is required, return                                          
  |                                                                             
  |--> if bus state is hello or running && wqueue is empty                      
  |    |                                                                        
  |    |    +-------------------+                                               
  |    +--> | bus_write_message | convert msg to iovec, write to bus's output fd
  |         +-------------------+                                               
  |                                                                             
  +--> else                                                                     
       -                                                                        
       +--> append msg to wqueue                                                
```

```
src/libsystemd/sd-bus/sd-bus.c
+-----------------------+                                                  
| bus_remarshal_message | : if necessary, remarshal msg                    
+-|---------------------+                                                  
  |                                                                        
  |--> check if wee need remarshal                                         
  |                                                                        
  +--> if we do                                                            
       |                                                                   
       |    +-----------------------+                                      
       +--> | bus_message_remarshal | alloc new msg, copy data from old one
            +-----------------------+                                      
```

```
src/libsystemd/sd-bus/bus-message.c                                                                                         
+-----------------------+                                                                                                    
| bus_message_remarshal | : alloc new msg, copy data from old one                                                            
+-|---------------------+                                                                                                    
  |                                                                                                                          
  |--> switch header type                                                                                                    
  |--> case signal                                                                                                           
  |    -    +---------------------------+                                                                                    
  |    +--> | sd_bus_message_new_signal | prepare msg of signal type, append path/interface/member info                      
  |         +---------------------------+                                                                                    
  |--> case method_call                                                                                                      
  |    -    +--------------------------------+                                                                               
  |    +--> | sd_bus_message_new_method_call | prepare msg of method_call type, append path/member/interface/destination info
  |         +--------------------------------+                                                                               
  |--> case method_return/method_error                                                                                       
  |    |    +--------------------+                                                                                           
  |    |--> | sd_bus_message_new | prepare msg                                                                               
  |    |    +--------------------+                                                                                           
  |    |    +-----------------------------+                                                                                  
  |    |--> | message_append_reply_cookie | append reply_cookie to msg                                                       
  |    |    +-----------------------------+                                                                                  
  |    |                                                                                                                     
  |    +--> if it's method_error                                                                                             
  |         |                                                                                                                
  |         |    +-----------------------------+                                                                             
  |         +--> | message_append_field_string | append error name to msg                                                    
  |              +-----------------------------+                                                                             
  |                                                                                                                          
  |--> if orig msg has destination                                                                                           
  |    |                                                                                                                     
  |    |    +-----------------------------+                                                                                  
  |    +--> | message_append_field_string | append destination to msg                                                        
  |         +-----------------------------+                                                                                  
  |                                                                                                                          
  |--> if orig msg has sender                                                                                                
  |    |                                                                                                                     
  |    |    +-----------------------------+                                                                                  
  |    +--> | message_append_field_string |                                                                                  
  |         +-----------------------------+                                                                                  
  |    +---------------------+                                                                                               
  |--> | sd_bus_message_copy | copy data from old msg to new one                                                             
  |    +---------------------+                                                                                               
  |    +---------------------+                                                                                               
  +--> | sd_bus_message_seal |                                                                                               
       +---------------------+                                                                                               
```

```
src/libsystemd/sd-bus/bus-objects.c                                                                                            
+--------------------------------+                                                                                              
| property_get_all_callbacks_run | ：prepare msg of type 'method return' (method is get_all), append all properties, send msg out
+-|------------------------------+                                                                                              
  |    +----------------------------------+                                                                                     
  |--> | sd_bus_message_new_method_return | prepare msg (type is 'method return')                                               
  |    +----------------------------------+                                                                                     
  |    +-------------------------------+                                                                                        
  |--> | sd_bus_message_open_container | given arg type, interpret container, extend msg and append container                   
  |    +-------------------------------+                                                                                        
  |                                                                                                                             
  |--> for each vtable                                                                                                          
  |    |                                                                                                                        
  |    |    +--------------------------+                                                                                        
  |    |--> | node_vtable_get_userdata | call ->find() to get user data                                                         
  |    |    +--------------------------+                                                                                        
  |    |                                                                                                                        
  |    |--> if found no obj, continue                                                                                           
  |    |                                                                                                                        
  |    |--> found_obj = true                                                                                                    
  |    |                                                                                                                        
  |    |--> if found no iface, continue                                                                                         
  |    |                                                                                                                        
  |    |--> found_iface = true                                                                                                  
  |    |                                                                                                                        
  |    |    +------------------------------+                                                                                    
  |    +--> | vtable_append_all_properties | given node, append all properties (name/value) to msg                              
  |         +------------------------------+                                                                                    
  |                                                                                                                             
  |--> if found_obj == false, return                                                                                            
  |                                                                                                                             
  |--> if found_iface == false, return error                                                                                    
  |                                                                                                                             
  |    +--------------------------------+                                                                                       
  |--> | sd_bus_message_close_container |                                                                                       
  |    +--------------------------------+                                                                                       
  |    +-------------+                                                                                                          
  +--> | sd_bus_send | seal, send msg out or append to wqueue                                                                   
       +-------------+                                                                                                          
```

```
src/libsystemd/sd-bus/bus-objects.c                                                    
+------------------------------+                                                        
| vtable_append_all_properties | : given node, append all properties (name/value) to msg
+-|----------------------------+                                                        
  |                                                                                     
  +--> for each vtable in node                                                          
       |                                                                                
       |    +----------------------------+                                              
       +--> | vtable_append_one_property | append property name/value to msg            
            +----------------------------+                                              
```

```
src/libsystemd/sd-bus/bus-objects.c                                                                                      
+----------------------------+                                                                                            
| vtable_append_one_property | : append property name/value to msg                                                        
+-|--------------------------+                                                                                            
  |    +-------------------------------+                                                                                  
  |--> | sd_bus_message_open_container | given arg type (dict entry), interpret container, extend msg and append container
  |    +-------------------------------+                                                                                  
  |    +-----------------------+                                                                                          
  +--> | sd_bus_message_append | append property (type = string) to msg                                                   
  |    +-----------------------+                                                                                          
  |    +-------------------------------+                                                                                  
  |--> | sd_bus_message_open_container | given arg type (variant), interpret container, extend msg and append container   
  |    +-------------------------------+                                                                                  
  |                                                                                                                       
  |--> given node, get outer slot                                                                                         
  |                                                                                                                       
  |    +---------------------+                                                                                            
  |--> | invoke_property_get | append property to msg                                                                     
  |    +---------------------+                                                                                            
  |    +--------------------------------+                                                                                 
  |--> | sd_bus_message_close_container | close container and free signature                                              
  |    +--------------------------------+                                                                                 
  |    +--------------------------------+                                                                                 
  +--> | sd_bus_message_close_container | (bc we opened the container twice)                                              
       +--------------------------------+                                                                                 
```

```
src/libsystemd/sd-bus/bus-objects.c                          
+---------------------+                                       
| invoke_property_get | : append property to msg              
+-|-------------------+                                       
  |                                                           
  |--> if property.get exists                                 
  |    |                                                      
  |    |--> call property.get()                               
  |    +--> return                                            
  |                                                           
  |--> get user data                                          
  |                                                           
  |    +-----------------------------+                        
  +--> | sd_bus_message_append_basic | append user data to msg
       +-----------------------------+                        
```

```
src/libsystemd/sd-bus/sd-bus.c                                                          
+-----------------+                                                                      
| process_closing | : prepare msg of 'disconnected', close bus and exit                  
+-|---------------+                                                                      
  |                                                                                      
  |--> prepare msg of 'disconnected'                                                     
  |                                                                                      
  |    +------------------------------+                                                  
  |--> | bus_message_set_sender_local | set sender = org.freedesktop.DBus.Local          
  |    +------------------------------+                                                  
  |    +----------------------------+                                                    
  |--> | bus_seal_synthetic_message | set timestamp and seal msg                         
  |    +----------------------------+                                                    
  |    +--------------+                                                                  
  |--> | sd_bus_close | release sources/event/fd of bus
  |    +--------------+                                                                  
  |    +----------------+                                                                
  |--> | process_filter | for each filter callback in bus: call ->callback()             
  |    +----------------+                                                                
  |    +---------------+                                                                 
  |--> | process_match | recursively traverse children/sibling, if leaf: run ->callback()
  |    +---------------+                                                                 
  |    +--------------+                                                                  
  +--> | bus_exit_now | (skip)                                                           
       +--------------+                                                                  
```

```
src/libsystemd/sd-bus/sd-bus.c                                                    
+--------------+                                                                   
| sd_bus_close | : release sources/event/fd of bus                                 
+-|------------+                                                                   
  |    +---------------+                                                           
  |--> | bus_kill_exec | kill task of bus                                          
  |    +---------------+                                                           
  |    +---------------+                                                           
  |--> | bus_set_state | set bus state = closed                                    
  |    +---------------+                                                           
  |    +---------------------+                                                     
  |--> | sd_bus_detach_event | unref sources, unref event                          
  |    +---------------------+                                                     
  |    +------------------+                                                        
  |--> | bus_reset_queues | for bus rqueue/wqueue: empty messages and release queue
  |    +------------------+                                                        
  |    +------------------+                                                        
  |--> | bus_close_io_fds | close bus input/output fd                              
  |    +------------------+                                                        
  |    +----------------------+                                                    
  +--> | bus_close_inotify_fd | close inotify fd                                   
       +----------------------+                                                    
```

```
src/libsystemd/sd-bus/sd-bus.c                                                       
+---------------------+                                                               
| sd_bus_detach_event | : unref sources, unref event                                  
+-|-------------------+                                                               
  |    +----------------------+                                                       
  |--> | bus_detach_io_events | for io input/output sources: set enabled = off & ref--
  |    +----------------------+                                                       
  |                                                                                   
  +--> for inotify/time/quite event source: set enabled = off & ref--                 
  |                                                                                   
  |    +----------------+                                                             
  +--> | sd_event_unref | event ref--, release if nobody needs it                     
       +----------------+                                                             
```

```
src/libsystemd/sd-event/sd-event.c                                                                
+----------------+                                                                                 
| sd_event_unref | : event ref--, release if nobody needs it                                       
+-|--------------+                                                                                 
  |                                                                                                
  |--> event ref--                                                                                 
  |                                                                                                
  +--> if ref > 0, return null                                                                     
       |                                                                                           
       |    +------------+                                                                         
       +--> | event_free | disconnect all sources from event, free other resources and event itself
            +------------+                                                                         
```

```
src/libsystemd/sd-event/sd-event.c                                                                  
+------------+                                                                                       
| event_free | : disconnect all sources from event, free other resources and event itself            
+-|----------+                                                                                       
  |                                                                                                  
  |--> for each source in event                                                                      
  |    |                                                                                             
  |    |    +-------------------+                                                                    
  |    +--> | source_disconnect | given source type, disconnect source properly, remove it from event
  |         +-------------------+                                                                    
  |                                                                                                  
  |--> close epoll/watchdog fd                                                                       
  |                                                                                                  
  |--> free clock data of realtime/boottime/monotonic/realtime_alarm/boottime_alarm                  
  |                                                                                                  
  |--> free event's pending/prepare/exit queue                                                       
  |                                                                                                  
  |--> free signal_sources and other hash maps                                                       
  |                                                                                                  
  +--> blabla, eventually free event itself                                                          
```

```
src/libsystemd/sd-event/sd-event.c                                                         
+-------------------+                                                                       
| source_disconnect | : given source type, disconnect source properly, remove it from event 
+-|-----------------+                                                                       
  |                                                                                         
  |--> switch source type                                                                   
  |--> case io                                                                              
  |    -    +----------------------+                                                        
  |    +--> | source_io_unregister | unregister from epoll                                  
  |         +----------------------+                                                        
  |--> case realtime/boottime/monotonic/realtime_alarm/boottime_alarm                       
  |    -    +--------------------------------+                                              
  |    +--> | event_source_time_prioq_remove | remove source from earliest/latest prio queue
  |         +--------------------------------+                                              
  |--> case signal                                                                          
  |    -    +----------------------+                                                        
  |    +--> | event_gc_signal_data | unset signal bit from all three possible masks         
  |         +----------------------+                                                        
  |--> case child                                                                           
  |    |    +----------------+                                                              
  |    |--> | hashmap_remove | remove source from event                                     
  |    |    +----------------+                                                              
  |    |    +-------------------------------+                                               
  |    +--> | source_child_pidfd_unregister | unregister source from epoll                  
  |         +-------------------------------+                                               
  |--> case post                                                                            
  |    -    +------------+                                                                  
  |    +--> | set_remove | remove source from event                                         
  |         +------------+                                                                  
  |--> case exit                                                                            
  |    -    +--------------+                                                                
  |    +--> | prioq_remove | remove source from event                                       
  |         +--------------+                                                                
  |--> case inotify                                                                         
  |    |    +-------------+                                                                 
  |    |--> | LIST_REMOVE | remove souroce from inode                                       
  |    |    +-------------+                                                                 
  |    |    +---------------------+                                                         
  |    +--> | event_gc_inode_data | garbage collect inode_data & inotify_data               
  |         +---------------------+                                                         
  |--> case memory_pressure                                                                 
  |    +--> remove source from event, from epoll                                            
  |                                                                                         
  |--> remove source from event pending/prepare prio queue                                  
  |                                                                                         
  |--> remove source from signal data's earliest/latest queue                               
  |                                                                                         
  +--> remove source from event, event source num--                                         
```

```
src/libsystemd/sd-bus/sd-bus.c
+----------+                                                               
| bus_poll | : determine flags and timeout, then poll                      
+-|--------+                                                               
  |                                                                        
  |--> if bus state == watch_bind                                          
  |    -                                                                   
  |    +--> set up pollfd from inotify fd                                  
  |                                                                        
  +--> else                                                                
       |                                                                   
       |    +-------------------+                                          
       |--> | sd_bus_get_events | determine flags (e.g., poll in, poll out)
       |    +-------------------+                                          
       |    +--------------------+                                         
       |--> | sd_bus_get_timeout | given bus state, determine timeout      
       |    +--------------------+                                         
       |                                                                   
       |--> set up pollfd                                                  
       |                                                                   
       |    +------------+                                                 
       +--> | ppoll_usec |                                                 
            +------------+                                                 
```

```
src/libsystemd/sd-event/sd-event.c                                                 
+--------------------------+                                                        
| memory_pressure_callback | : free unused hash maps, return unused memory to kernel
+----------------------+---+                                                        
| sd_event_trim_memory | : free unused hash maps, return unused memory to kernel    
+-|--------------------+                                                            
  |    +--------------------+                                                       
  |--> | hashmap_trim_pools | free unused hash maps                                 
  |    +--------------------+                                                       
  |    +-------------+                                                              
  +--> | malloc_trim | return unused memory to kernel                               
       +-------------+                                                              
```
