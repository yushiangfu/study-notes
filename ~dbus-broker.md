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
| run | : prepare broker, continuously poll and handle events
+-|---+                                                        
  |    +------------+                                          
  |--> | broker_new | prepare broker, init bus/epoll/controller
  |    +------------+                                          
  |    +------------+                                          
  +--> | broker_run | add file to context, loop: poll and handle events in context
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

```
src/broker/broker.c                                                                                  
+------------+                                                                                        
| broker_run | : add file to context, loop: poll and handle events in context                         
+-|----------+                                                                                        
  |                                                                                                   
  |--> prepare mask = SIGTERM | SIGINT                                                                
  |                                                                                                   
  |    +-------------+                                                                                
  |--> | sigprocmask | block                                                                          
  |    +-------------+                                                                                
  |    +-----------------+                                                                            
  |--> | connection_open | add file to the end of context's link                                      
  |    +-----------------+                                                                            
  |                                                                                                   
  |--> loop                                                                                           
  |    |                                                                                              
  |    |    +---------------------------+                                                             
  |    +--> | dispatch_context_dispatch | poll events and add to context, handle each event in context
  |         +---------------------------+                                                             
  |    +---------------------+                                                                        
  |--> | peer_registry_flush | release every peer in peer-registry                                    
  |    +---------------------+                                                                        
  |    +--------------------+                                                                         
  |--> | broker_log_metrics | prepare log and commit to journal                                       
  |    +--------------------+                                                                         
  |    +-------------+                                                                                
  +--> | sigprocmask | restore old signal mask                                                        
       +-------------+                                                                                
```

```
src/dbus/connection.c                                                
+-----------------+                                                   
| connection_open | : add file to the end of context's link           
+-|---------------+                                                   
  |                                                                   
  |--> if connection has no server                                    
  |    -                                                              
  |    +--> (skip, probably not our case)                             
  |                                                                   
  |    +----------------------+                                       
  +--> | dispatch_file_select | add file to the end of context's link 
       +----------------------+                                       
```

```
src/util/dispatch.c                                                                        
+---------------------------+                                                               
| dispatch_context_dispatch | : poll events and add to context, handle each event in context
+-|-------------------------+                                                               
  |    +-----------------------+                                                            
  |--> | dispatch_context_poll | fetch event from kernel, append file to list of context    
  |    +-----------------------+                                                            
  |    +-------------+                                                                      
  |--> | c_list_swap | move file list from context to local                                 
  |    +-------------+                                                                      
  |                                                                                         
  +--> for each file in local list                                                          
       |                                                                                    
       |    +---------------+                                                               
       |--> | c_list_unlink | remove file from local list                                   
       |    +---------------+                                                               
       |    +------------------+                                                            
       |--> | c_list_link_tail | move file to the end of list of context                    
       |    +------------------+                                                            
       |                                                                                    
       +--> call file->fn(), e.g.,                                                          
            +-------------------------+                                                     
            | broker_dispatch_signals | read signal-info from signal-fd                     
            +-------------------------+                                                     
```

```
src/bus/peer.c                                              
+---------------------+                                      
| peer_registry_flush | : release every peer in peer-registry
+-|-------------------+                                      
  |                                                          
  +--> for each peer in rbtree                               
       |                                                     
       |    +----------------+                               
       |--> | driver_goodbye | release everything in peer    
       |    +----------------+                               
       |    +-----------+                                    
       +--> | peer_free | release peer                       
            +-----------+                                    
```

```
src/bus/driver.c                                               
+----------------+                                              
| driver_goodbye | : release peer                               
+-|--------------+                                              
  |    +--------------------+                                   
  |--> | peer_flush_matches | unref all rules in peer           
  |    +--------------------+                                   
  |                                                             
  |--> for each reply in list of peer                           
  |    |                                                        
  |    |    +-----------------+                                 
  |    +--> | reply_slot_free | release slot                    
  |         +-----------------+                                 
  |    +----------------------+                                 
  |--> | match_registry_flush | flush sender matches            
  |    +----------------------+                                 
  |                                                             
  |--> for each ownership in list of peer                       
  |    |                                                        
  |    |    +-----------------------------+                     
  |    +--> | peer_release_name_ownership | release ownership   
  |         +-----------------------------+                     
  |                                                             
  |--> if peer is registered                                    
  |    |                                                        
  |    |    +-----------------+                                 
  |    +--> | peer_unregister |                                 
  |         +-----------------+                                 
  |                                                             
  |--> else if peer is monitor                                  
  |    |                                                        
  |    |    +-------------------+                               
  |    +--> | peer_stop_monitor |                               
  |         +-------------------+                               
  |    +----------------------+                                 
  |--> | match_registry_flush | flush name-owner-changed matches
  |    +----------------------+                                 
  |                                                             
  +--> for each reply-slot in list of peer                      
       |                                                        
       |    +-----------------+                                 
       +--> | reply_slot_free | release reply-slot              
            +-----------------+                                 
```

```
src/bus/peer.c                                 
+--------------------+                          
| peer_flush_matches | : unref all rules in peer
+-|------------------+                          
  |                                             
  +--> for each node in rule tree of peer       
       |                                        
       |--> given node, get outer rule          
       |                                        
       |    +-----------------------+           
       +--> | match_rule_user_unref | unref rule
            +-----------------------+           
```

```
src/bus/match.c                                                                  
+----------------------+                                                          
| match_registry_flush | : flush match-registry                                   
+-|--------------------+                                                          
  |                                                                               
  |--> prepare tree array consists of two trees from registry                     
  |                                                                               
  +--> for each tree in array                                                     
       -                                                                          
       +--> for each path-registry in tree                                        
            |                                                                     
            |    +----------------------------+                                   
            |--> | match_registry_by_path_ref | registry ref++                    
            |    +----------------------------+                                   
            |    +------------------------------+                                 
            |--> | match_registry_by_path_flush | flush path-registry             
            |    +------------------------------+                                 
            |    +------------------------------+                                 
            +--> | match_registry_by_path_unref | unlink and release path-registry
                 +------------------------------+                                 
```

```
src/bus/match.c                                                                       
+------------------------------+                                                       
| match_registry_by_path_flush | : flush path-registry                                 
+-|----------------------------+                                                       
  |                                                                                    
  +--> for each interface-registry in path-registry                                    
       |                                                                               
       |    +---------------------------------+                                        
       |--> | match_registry_by_interface_ref | registry ref++                         
       |    +---------------------------------+                                        
       |    +-----------------------------------+                                      
       |--> | match_registry_by_interface_flush | flush interface-registry             
       |    +-----------------------------------+                                      
       |    +-----------------------------------+                                      
       +--> | match_registry_by_interface_unref | unlink and release interface-registry
            +-----------------------------------+                                      
```

```
src/bus/match.c                                                                 
+-----------------------------------+                                            
| match_registry_by_interface_flush | : flush interface-registry                 
+-|---------------------------------+                                            
  |                                                                              
  +--> for each member-registry in interface-registry                            
       |                                                                         
       |    +------------------------------+                                     
       |--> | match_registry_by_member_ref | ref++                               
       |    +------------------------------+                                     
       |    +--------------------------------+                                   
       |--> | match_registry_by_member_flush | flush key-registry                
       |    +--------------------------------+                                   
       |    +--------------------------------+                                   
       +--> | match_registry_by_member_unref | unlink and release member-registry
            +--------------------------------+                                   
```

```
src/bus/match.c                                                                         
+--------------------------------+                                                       
| match_registry_by_member_flush | : flush member-registry                               
+-|------------------------------+                                                       
  |    +----------------------------+                                                    
  |--> | match_registry_by_keys_ref | ref++                                              
  |    +----------------------------+                                                    
  |    +------------------------------+                                                  
  |--> | match_registry_by_keys_flush | for each rule in key-registry: unlink and release
  |    +------------------------------+                                                  
  |    +------------------------------+                                                  
  +--> | match_registry_by_keys_unref | unlink and release key-registry                  
       +------------------------------+                                                  
```
