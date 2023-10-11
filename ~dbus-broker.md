```
src/broker/main.c                                                                       
+------+                                                                                 
| main |                                                                                 
+-|----+                                                                                 
  |    +------------+                                                                    
  |--> | parse_argv | parse options and check log socket & controller socket & machine id
  |    +------------+                                                                    
  |    +-------+                                                                         
  |--> | setup | clear ambient set so it won't be inherited by child task                
  |    +-------+                                                                         
  |                                                                                      
  |--> if main_arg_audit is set (not our case)                                           
  |    |                                                                                 
  |    |    +------------------------+                                                   
  |    +--> | util_audit_init_global |                                                   
  |         +------------------------+                                                   
  |    +-------------------------+                                                       
  |--> | bus_selinux_init_global | (skip)                                                
  |    +-------------------------+                                                       
  |    +-----+                                                                           
  +--> | run | prepare broker and run                                                    
       +-----+                                                                           
```

```
src/broker/main.c                                                                  
+------------+                                                                      
| parse_argv | : parse options and check log socket & controller socket & machine id
+-|----------+                                                                      
  |                                                                                 
  |--> loop, parse each option, e.g.,                                               
  |        --log 4                                                                  
  |        --controller 9                                                           
  |        --machine-id 74f2acf751014c619fd386d9b94f3f13                            
  |        --max-bytes 536870912                                                    
  |        --max-fds 4096                                                           
  |        --max-matches 16384                                                      
  |                                                                                 
  |--> if main_arg_log >= 0 (e.g., 8, note: this is socket fd)                      
  |    -                                                                            
  |    +--> get socket option and check                                             
  |                                                                                 
  |--> get controller socket option and check (e.g., 9, note: this is socket fd)    
  |                                                                                 
  +--> check machine id is passed thru options                                      
```

```
src/broker/main.c                                             
+-----+                                                        
| run | : prepare broker and run                               
+-|---+                                                        
  |    +------------+                                          
  |--> | broker_new | prepare broker, init bus/epoll/controller
  |    +------------+                                          
  |    +------------+                                          
  +--> | broker_run | (skip)                                   
       +------------+                                          
```

```
src/broker/broker.c                                                                      
+------------+                                                                            
| broker_new | : prepare broker, init bus/epoll/controller                                
+-|----------+                                                                            
  |                                                                                       
  |--> given log_fd, get log type                                                         
  |                                                                                       
  |--> alloc and init broker                                                              
  |                                                                                       
  |--> if log type == stream                                                              
  |    |                                                                                  
  |    |    +------------------+                                                          
  |    +--> | log_init_journal |                                                          
  |         +------------------+                                                          
  |    +----------+                                                                       
  |--> | bus_init | init bus (machine id, guid, users)                                    
  |    +----------+                                                                       
  |    +---------------------+                                                            
  |--> | sockopt_get_peersec | set sec-label (skip)                                       
  |    +---------------------+                                                            
  |    +------------------------+                                                         
  |--> | sockopt_get_peergroups | get peer groups (skip)                                  
  |    +------------------------+                                                         
  |    +------------------------+                                                         
  |--> | user_registry_ref_user | search user in registry (skip)                          
  |    +------------------------+                                                         
  |    +-----------------------+                                                          
  |--> | dispatch_context_init | create epoll                                             
  |    +-----------------------+                                                          
  |    +----------+                                                                       
  |--> | signalfd | get signal fd of mask (SIGTERM | SIGINT)                              
  |    +----------+                                                                       
  |    +--------------------+                                                             
  |--> | dispatch_file_init | config epoll, set up arg file                               
  |    +--------------------+                                                             
  |    +----------------------+                                                           
  |--> | dispatch_file_select | add file to the end of context's link                     
  |    +----------------------+                                                           
  |    +-----------------+                                                                
  +--> | controller_init | init controller (install broker, init connection & sasl server)
       +-----------------+                                                                
```

```
src/bus/bus.c                                                     
+----------+                                                       
| bus_init | : init bus (machine id, guid, users)                  
+-|--------+                                                       
  |                                                                
  |--> bus->machine_id = machine_id                                
  |                                                                
  |--> bus->guid = random                                          
  |                                                                
  |    +--------------------+                                      
  +--> | user_registry_init | set up registry, copy maxima from arg
       +--------------------+                                      
```

```
src/util/user.c                                              
+--------------------+                                        
| user_registry_init | : set up registry, copy maxima from arg
+-|------------------+                                        
  |                                                           
  |--> alloc maxima for registry                              
  |                                                           
  |--> set up registry                                        
  |                                                           
  +--> copy arg maxima array to registry                      
```

```
src/broker/controller.c                                                             
+-----------------+                                                                  
| controller_init | : init controller (install broker, init connection & sasl server)
+-|---------------+                                                                  
  |                                                                                  
  |--> controller->broker = broker                                                   
  |                                                                                  
  |    +------------------------+                                                    
  +--> | connection_init_server | init connection and sasl server                    
       +------------------------+                                                    
```

```
src/dbus/connection.c                                                                                 
+------------------------+                                                                             
| connection_init_server | : init connection and sasl server                                           
+-|----------------------+                                                                             
  |    +-----------------+                                                                             
  |--> | connection_init | init socket, config epoll, set up file (fn = controller_dispatch_connection)
  |    +-----------------+                                                                             
  |    +------------------+                                                                            
  +--> | sasl_server_init | init sasl server                                                           
       +------------------+                                                                            
```

```
src/dbus/connection.c                                                                            
+-----------------+                                                                               
| connection_init | : init socket, config epoll, set up file (fn = controller_dispatch_connection)
+-|---------------+                                                                               
  |    +-------------+                                                                            
  |--> | socket_init | init socket (user, fd, input queue)                                        
  |    +-------------+                                                                            
  |    +--------------------+                                                                     
  +--> | dispatch_file_init | config epoll, set up arg file                                       
       +--------------------+ (fn = controller_dispatch_connection)                               
```

```
src/util/dispatch.c                                              
+--------------------+                                            
| dispatch_file_init | : config epoll, set up arg file            
+-|------------------+                                            
  |    +-----------+                                              
  |--> | epoll_ctl | add poll-in                                  
  |    +-----------+                                              
  |                                                               
  |--> set up arg file                                            
  |                                                               
  +--> install arg fn, e.g.,                                      
       +-------------------------+                                
       | broker_dispatch_signals | read signal-info from signal-fd
       +--------------------------------+                         
       | controller_dispatch_connection |                         
       +--------------------------------+                         
```
