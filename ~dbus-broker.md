## Index

- [Cheat Sheet](#cheat-sheet)

```
src/launch/main.c                                                                           
+------+                                                                                     
| main |                                                                                     
+-|----+                                                                                     
  |    +-------------+                                                                       
  |--> | inherit_fds | Assume fd 3 is an inherited socket at '/run/dbus/system_bus_socket'.  
  |    +-------------+                                                                       
  |    +-----+                                                                               
  +--> | run |                                                                               
       +-|---+                                                                               
         |    +--------------+                                                               
         |--> | launcher_new |                                                               
         |    +-|------------+                                                               
         |      |                                                                            
         |      |--> Prepare an event and add signal sources for SIGTERM, SIGINT, and SIGHUP.
         |      |                                                                            
         |      +--> Obtain bus for controller.                                              
         |                                                                                   
         |    +--------------+                                                               
         +--> | launcher_run |                                                               
              +-|------------+                                                               
                |    +------------------------+                                              
                |--> | launcher_load_services | Load service files into the launcher.        
                |    +------------------------+                                              
                |                                                                            
                +--> Prepare socket pair, [0] for launcher, [1] for broker.                  
                |                                                                            
                |    +---------------+                                                       
                |--> | launcher_fork | Launch broker.                                        
                |    +---------------+                                                       
                |    +--------------------------+                                            
                |--> | sd_bus_add_object_vtable | Register launcher's method calls.          
                |    +--------------------------+                                            
                |    +-------------------+                                                   
                |--> | sd_bus_add_filter | Register service activation filter.               
                |    +-------------------+                                                   
                |    +--------------+                                                        
                |--> | sd_bus_start | Start controller bus, ready for message exchange.      
                |    +--------------+                                                        
                |    +-----------------------+                                               
                |--> | launcher_add_services | Register services with the broker.            
                |    +-----------------------+                                               
                |    +-----------------------+                                               
                |--> | launcher_add_listener | Add listener to broker (purpose?).            
                |    +-----------------------+                                               
                |    +------------------+                                                    
                |--> | launcher_connect | Open and start regular bus for message exchange.   
                |    +------------------+                                                    
                |    +---------------+                                                       
                +--> | sd_event_loop | In a loop, handle events.                             
                     +---------------+                                                       
```

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

```
src/broker/controller.c                                                                                           
+--------------------------------+                                                                                 
| controller_dispatch_connection | : as controller, endlessly fetch msg & perform action & reply                   
+-|------------------------------+                                                                                 
  |                                                                                                                
  |--> given arg file, get outer controller                                                                        
  |                                                                                                                
  |    +---------------------+                                                                                     
  +--> | connection_dispatch | handle arg events (in/out/hup) accordingly (read/write/discard)                     
  |    +---------------------+                                                                                     
  |                                                                                                                
  +--> loop                                                                                                        
       |                                                                                                           
       |    +--------------------+                                                                                 
       +--> | connection_dequeue | fetch msg from input buffer, return to caller                                   
       |    +--------------------+                                                                                 
       |    +--------------------------+                                                                           
       +--> | controller_dbus_dispatch | given msg, perform action (method call or reply), prepare msg sending back
            +--------------------------+                                                                           
```

```
src/dbus/connection.c                                                                   
+---------------------+                                                                  
| connection_dispatch | : handle arg events (in/out/hup) accordingly (read/write/discard)
+-|-------------------+                                                                  
  |                                                                                      
  +--> for event in [epoll-in, epoll-hup, epoll-out]                                     
       -                                                                                 
       +--> if it matches with arg event                                                 
            |                                                                            
            |    +-----------------+                                                     
            |--> | socket_dispatch | given event (in/out/hup), handle it accordingly     
            |    +-----------------+                                                     
            |    +---------------------+                                                 
            +--> | dispatch_file_clear | clear mask in file, unlink file if it's handled 
                 +---------------------+                                                 
```

```
src/dbus/socket.c                                                                                
+-----------------+                                                                               
| socket_dispatch | : given event (in/out/hup), handle it accordingly                             
+-|---------------+                                                                               
  |                                                                                               
  |--> switch event                                                                               
  |                                                                                               
  |--> case epoll-in                                                                              
  |    |                                                                                          
  |    |    +----------------------+                                                              
  |    +--> | socket_dispatch_read | read data from buffer, read socket to add more data to buffer
  |         +----------------------+                                                              
  |                                                                                               
  |--> case epoll-out                                                                             
  |    |                                                                                          
  |    |    +-----------------------+                                                             
  |    +--> | socket_dispatch_write | set up msg array, send msg                                  
  |         +-----------------------+                                                             
  |                                                                                               
  +--> case epoll-hup                                                                             
       |                                                                                          
       |    +----------------------+                                                              
       +--> | socket_hangup_output | discard output buffers                                       
            +----------------------+                                                              
```

```
src/dbus/socket.c                                                                                   
+----------------------+                                                                             
| socket_dispatch_read | : read data from buffer, read socket to add more data to buffer             
+-|--------------------+                                                                             
  |    +-------------------+                                                                         
  |--> | iqueue_get_cursor | move data to buffer start, read data from pending buffer or input buffer
  |    +-------------------+                                                                         
  |    +----------------+                                                                            
  +--> | socket_recvmsg | receive msg                                                                
       +----------------+                                                                            
```

```
src/dbus/queue.c                                                                               
+-------------------+                                                                           
| iqueue_get_cursor | : move data to buffer start, read data from pending buffer or input buffer
+-|-----------------+                                                                           
  |    +---------+                                                                              
  |--> | memmove | move data to buffer start                                                    
  |    +---------+                                                                              
  |                                                                                             
  |--> if there's pending buffer, read and return                                               
  |                                                                                             
  +--> read from input buffer, and return                                                       
```

```
src/dbus/socket.c                           
+----------------+                           
| socket_recvmsg | : receive msg             
+-|--------------+                           
  |                                          
  |--> set up msg header                     
  |                                          
  |    +---------+                           
  |--> | recvmsg |                           
  |    +---------+                           
  |                                          
  |--> for each cmsg-header in msg-header    
  |    -                                     
  |    +--> get fds and n_fd from cmsg-header
  |                                          
  +--> from += received_bytes                
```

```
src/dbus/socket.c                                    
+-----------------------+                             
| socket_dispatch_write | : set up msg array, send msg
+-|---------------------+                             
  |                                                   
  |--> if out-pending list isn't empty                
  |    -                                              
  |    +--> for each buffer in out-pending list       
  |         |                                         
  |         |    +--------------------+               
  |         +--> | socket_buffer_free | release buffer
  |              +--------------------+               
  |                                                   
  |--> for each buffer in out-queue list              
  |    -                                              
  |    +--> set up msg array                          
  |                                                   
  |    +----------+                                   
  |--> | sendmmsg |                                   
  |    +----------+                                   
  |                                                   
  +--> for each buffer in out-queue list              
       |                                              
       |    +-----------------------+                 
       +--> | socket_buffer_consume |                 
            +-----------------------+                 
```

```
src/dbus/connection.c                                                
+--------------------+                                                
| connection_dequeue | : fetch msg from input buffer, return to caller
+-|------------------+                                                
  |    +----------------+                                             
  |--> | socket_dequeue | fetch msg from input buffer (skip)          
  |    +----------------+                                             
  |                                                                   
  +--> if no msg, return                                              
```

```
src/broker/controller-dbus.c                                                                                             
+--------------------------+                                                                                              
| controller_dbus_dispatch | : given msg, perform action (method call or reply), prepare msg sending back                 
+-|------------------------+                                                                                              
  |    +------------------------+                                                                                         
  |--> | message_parse_metadata | validate msg header & body, truncate extra fds                                          
  |    +------------------------+                                                                                         
  |                                                                                                                       
  +--> switch header type                                                                                                 
       case method_call                                                                                                   
       -    +----------------------------+                                                                                
       +--> | controller_dispatch_object | given object path, write reply header, call method->fn, prepare msg sending out
            +----------------------------+                                                                                
       case method_return                                                                                                 
       -    +---------------------------+                                                                                 
       +--> | controller_dispatch_reply | given sender id, find peer, prepare msg sending to it                           
            +---------------------------+                                                                                 
```

```
src/dbus/message.c                                                        
+------------------------+                                                 
| message_parse_metadata | : validate msg header & body, truncate extra fds
+-|----------------------+                                                 
  |    +----------------------+                                            
  |--> | message_parse_header | (skip)                                     
  |    +----------------------+                                            
  |    +--------------------+                                              
  |--> | message_parse_body | (skip)                                       
  |    +--------------------+                                              
  |                                                                        
  +--> if message->fds exists                                              
       |                                                                   
       |    +-----------------+                                            
       +--> | fdlist_truncate | truncate extra fds                         
            +-----------------+                                            
```

```
src/broker/controller-dbus.c                                                                                                   
+----------------------------+                                                                                                  
| controller_dispatch_object | : given object path, write reply header, call method->fn, prepare msg sending out                
+-|--------------------------+                                                                                                  
  |                                                                                                                             
  |--> if path == /org/bus1/DBus/Broker                                                                                         
  |    |                                                                                                                        
  |    |    +--------------------------------+                                                                                  
  |    +--> | controller_dispatch_controller | find matched method, write reply header, call method->fn, prepare msg sending out
  |         +--------------------------------+ (for AddName or AddListener)                                                     
  |                                                                                                                             
  |--> elif path == /org/bus1/DBus/Name/                                                                                        
  |    |                                                                                                                        
  |    |    +--------------------------+                                                                                        
  |    +--> | controller_dispatch_name | find matched method, write reply header, call method->fn, prepare msg sending out      
  |         +--------------------------+ (for Reset or Release)                                                                 
  |                                                                                                                             
  +--> elif path == /org/bus1/DBus/Listener/                                                                                    
       |                                                                                                                        
       |    +------------------------------+                                                                                    
       +--> | controller_dispatch_listener | find matched method, write reply header, call method->fn, prepare msg sending out  
            +------------------------------+ (for Release or SetPolicy)                                                         
```

```
src/broker/controller-dbus.c                                                                                                     
+--------------------------------+                                                                                                
| controller_dispatch_controller | : find matched method, write reply header, call method->fn, prepare msg sending out            
+-|------------------------------+                                                                                                
  |                                                                                                                               
  +--> for each method in predefined array                                                                                        
       -                                                                                                                          
       +--> if arg method matches                                                                                                 
            |                                                                                                                     
            |    +--------------------------+                                                                                     
            +--> | controller_handle_method | write generic reply header, call method->fn, prepare msg & add connection to context
                 +--------------------------+                                                                                     
```

```
src/broker/controller-dbus.c                                                                                      
+--------------------------+                                                                                       
| controller_handle_method | : write generic reply header, call method->fn, prepare msg & add connection to context
+-|------------------------+                                                                                       
  |    +-------------------------------------+                                                                     
  |--> | controller_dvar_verify_signature_in | verify input signature                                              
  |    +-------------------------------------+                                                                     
  |    +-------------------+                                                                                       
  |--> | c_dvar_begin_read | set up var_in                                                                         
  |    +-------------------+                                                                                       
  |    +--------------------+                                                                                      
  |--> | c_dvar_begin_write | set up var_out                                                                       
  |    +--------------------+                                                                                      
  |    +-------------------------------+                                                                           
  |--> | controller_write_reply_header | write generic reply header (skip)                                         
  |    +-------------------------------+                                                                           
  |                                                                                                                
  |--> call method->fn(), e.g.,                                                                                    
  |    +--------------------------------+                                                                          
  |    | controller_method_add_listener | prepare registry & import policy, prepare listener & add to controller   
  |    +--------------------------------+                                                                          
  |                                                                                                                
  |    +----------------------+                                                                                    
  |--> | message_new_outgoing | prepare msg for sending out                                                        
  |    +----------------------+                                                                                    
  |    +------------------+                                                                                        
  +--> | connection_queue | add msg to connection, add connnection to context                                      
       +------------------+                                                                                        
```

```
src/broker/controller-dbus.c                                                                               
+--------------------------------+                                                                          
| controller_method_add_listener | : prepare registry & import policy, prepare listener & add to controller 
+-|------------------------------+                                                                          
  |    +---------------------+                                                                              
  |--> | policy_registry_new | prepare registry (apparmor/selinux/policy-batch)                             
  |    +---------------------+                                                                              
  |    +------------------------+                                                                           
  |--> | policy_registry_import | import policy registry (?)                                                
  |    +------------------------+                                                                           
  |                                                                                                         
  |--> get listener fd                                                                                      
  |                                                                                                         
  |    +-------------------------+                                                                          
  |--> | controller_add_listener | prepare listener & add to controller, init dispatch-file & add to context
  |    +-------------------------+                                                                          
  |    +--------------+                                                                                     
  +--> | fdlist_steal | drop listener fd from array                                                         
       +--------------+                                                                                     
```

```
src/bus/policy.c                                                         
+---------------------+                                                   
| policy_registry_new | : prepare registry (apparmor/selinux/policy-batch)
+-|-------------------+                                                   
  |                                                                       
  |--> alloc registry                                                     
  |                                                                       
  |    +---------------------------+                                      
  |--> | bus_apparmor_registry_new | prepare apparmor registry            
  |    +---------------------------+                                      
  |    +--------------------------+                                       
  |--> | bus_selinux_registry_new | (skip)                                
  |    +--------------------------+                                       
  |    +------------------+                                               
  +--> | policy_batch_new | prepare policy-batch                          
       +------------------+                                               
```

```
src/bus/policy.c                                                                                            
+------------------------+                                                                                   
| policy_registry_import | : import policy registry (?)                                                      
+-|----------------------+                                                                                   
  |                                                                                                          
  |--> while there's more                                                                                    
  |    |                                                                                                     
  |    |--> read uid                                                                                         
  |    |                                                                                                     
  |    |--> if uid == -1                                                                                     
  |    |    |                                                                                                
  |    |    |    +------------------------------+                                                            
  |    |    +--> | policy_registry_import_batch | prepare two xmit, add to send/recv lists of name separately
  |    |         +------------------------------+                                                            
  |    |                                                                                                     
  |    +--> else                                                                                             
  |         |                                                                                                
  |         |    +------------------------+                                                                  
  |         +--> | policy_registry_at_uid | (skip)                                                           
  |         |    +------------------------+                                                                  
  |         |    +------------------------------+                                                            
  |         +--> | policy_registry_import_batch | prepare two xmit, add to send/recv lists of name separately
  |              +------------------------------+                                                            
  |                                                                                                          
  +--> (skip)                                                                                                
```

```
src/bus/policy.c                                                                             
+------------------------------+                                                              
| policy_registry_import_batch | : prepare two xmit, add to send/recv lists of name separately
+-|----------------------------+                                                              
  |                                                                                           
  |--> read out verdict and priority                                                          
  |                                                                                           
  |--> while there's more                                                                     
  |    |                                                                                      
  |    |--> read out some data                                                                
  |    |                                                                                      
  |    |--> if is_prefix                                                                      
  |    |    |                                                                                 
  |    |    |    +-----------------------------+                                              
  |    |    +--> | policy_batch_add_own_prefix | ensure name has higher priority              
  |    |         +-----------------------------+                                              
  |    |                                                                                      
  |    +--> else                                                                              
  |         |                                                                                 
  |         |    +----------------------+                                                     
  |         +--> | policy_batch_add_own | ensure name has higher priority                     
  |              +----------------------+                                                     
  |                                                                                           
  |--> while there's more                                                                     
  |    |                                                                                      
  |    |--> read out some data                                                                
  |    |                                                                                      
  |    |    +-----------------------+                                                         
  |    +--> | policy_batch_add_send | prepare xmit and insert to list end of name             
  |         +-----------------------+                                                         
  |                                                                                           
  +--> while there's more                                                                     
       |                                                                                      
       |--> read out some data                                                                
       |                                                                                      
       |    +-----------------------+                                                         
       +--> | policy_batch_add_recv | prepare xmit and insert to list end of name             
            +-----------------------+                                                         
```

```
src/bus/policy.c                                                
+-----------------------------+                                  
| policy_batch_add_own_prefix | : ensure name has higher priority
+-|---------------------------+                                  
  |    +----------------------+                                  
  +--> | policy_batch_at_name | ensure name is in tree of batch  
  |    +----------------------+                                  
  |                                                              
  +--> if name's priority is lower than arg verdict              
       -                                                         
       +--> save arg verdict to name                             
```

```
src/bus/policy.c                                         
+----------------------+                                  
| policy_batch_at_name | : ensure name is in tree of batch
+-|--------------------+                                  
  |                                                       
  |--> given name, find slot from tree of batch           
  |                                                       
  +--> if slot found                                      
       |                                                  
       |    +-----------------------+                     
       |--> | policy_batch_name_new | prepare name        
       |    +-----------------------+                     
       |    +--------------+                              
       +--> | c_rbtree_add | add name to tree             
            +--------------+                              
```

```
src/bus/policy.c                                                      
+-----------------------+                                              
| policy_batch_add_send | : prepare xmit and insert to list end of name
+-|---------------------+                                              
  |    +-----------------+                                             
  |--> | policy_xmit_new | prepare xmit                                
  |    +-----------------+                                             
  |                                                                    
  |--> xmit->verdict = verdict                                         
  |                                                                    
  |    +----------------------+                                        
  |--> | policy_batch_at_name | ensure name is in tree of batch        
  |    +----------------------+                                        
  |                                                                    
  +--> insert xmit to list end of name                                 
```

```
src/broker/controller.c                                                                                
+-------------------------+                                                                             
| controller_add_listener | : prepare listener & add to controller, init dispatch-file & add to context 
+-|-----------------------+                                                                             
  |    +-------------------------+                                                                      
  |--> | controller_listener_new | prepare listener, save path in it, add listener to tree of controller
  |    +-------------------------+                                                                      
  |    +-----------------------+                                                                        
  +--> | listener_init_with_fd | set up listener, init dispatch-file and add to context                 
       +-----------------------+                                                                        
```

```
src/bus/listener.c                                                               
+-----------------------+                                                         
| listener_init_with_fd | : set up listener, init dispatch-file and add to context
+-|---------------------+                                                         
  |    +--------------------+                                                     
  |--> | dispatch_file_init | config epoll, set up arg file                       
  |    +--------------------+ +-------------------+                               
  |                           | listener_dispatch | (skip)                        
  |                           +-------------------+                               
  |                                                                               
  |    +----------------------+                                                   
  +--> | dispatch_file_select | add file to the end of context's link             
       +----------------------+                                                   
```

```
src/broker/controller-dbus.c                                                           
+------------------+                                                                    
| connection_queue | : add msg to connection, add connnection to context                
+-|----------------+                                                                    
  |    +--------------+                                                                 
  |--> | socket_queue | prepare buffer containing msg, add to end of out-queue of socket
  |    +--------------+                                                                 
  |    +----------------------+                                                         
  +--> | dispatch_file_select | add file to the end of context's link                   
       +----------------------+                                                         
```

```
src/dbus/socket.c                                                                 
+--------------+                                                                   
| socket_queue | : prepare buffer containing msg, add to end of out-queue of socket
+-|------------+                                                                   
  |    +---------------------------+                                               
  |--> | socket_buffer_new_message | prepare buffer, save arg msg in it            
  |    +---------------------------+                                               
  |    +------------------+                                                        
  +--> | c_list_link_tail | add buffer to the end of out-queue of socket           
       +------------------+                                                        
```

```
src/dbus/socket.c                                                                     
+---------------------------+                                                          
| socket_buffer_new_message | : prepare buffer, save arg msg in it                     
+-|-------------------------+                                                          
  |    +----------------------------+                                                  
  |--> | socket_buffer_new_internal | prepare buffer                                   
  |    +----------------------------+                                                  
  |                                                                                    
  |--> save arg msg in buffer                                                          
  |                                                                                    
  |    +-------------+                                                                 
  |--> | user_charge | given slot idx, update user slot and usage slot by amount change
  |    +-------------+ (this is for USER_SLOT_BYTES)                                   
  |    +-------------+                                                                 
  +--> | user_charge | this is for USER_SLOT_FDS                                       
       +-------------+                                                                 
```

```
src/dbus/socket.c                                                                
+-------------+                                                                   
| user_charge | : given slot idx, update user slot and usage slot by amount change
+-|-----------+                                                                   
  |                                                                               
  |--> determine actor                                                            
  |                                                                               
  |--> get usage and ref++                                                        
  |                                                                               
  |--> given arg slot (idx), get user slot and usage slot                         
  |                                                                               
  |--> user slot -= amount                                                        
  |                                                                               
  +--> usage slot += amount                                                       
```

```
src/broker/controller-dbus.c                                                               
+---------------------------+                                                               
| controller_dispatch_reply | : given sender id, find peer, prepare msg sending to it       
+-|-------------------------+                                                               
  |    +------------------------+                                                           
  |--> | controller_find_reload | given controller, find outer 'reload'                     
  |    +------------------------+                                                           
  |    +-----------------------------+                                                      
  |--> | controller_reload_completed | given sender id, find peer, prepare msg sending to it
  |    +-----------------------------+                                                      
  |    +------------------------+                                                           
  +--> | controller_reload_free | release 'reload'                                          
       +------------------------+                                                           
```

```
src/broker/controller-dbus.c                                                             
+-----------------------------+                                                           
| controller_reload_completed | : given sender id, find peer, prepare msg sending to it   
+--------------------------------+                                                        
| driver_reload_config_completed | : given sender id, find peer, prepare msg sending to it
+-|------------------------------+                                                        
  |    +-------------------------+                                                        
  |--> | peer_registry_find_peer | given sender id, find peer (sender) from bus           
  |    +-------------------------+                                                        
  |                                                                                       
  +--> if sender found                                                                    
       |                                                                                  
       |--> prepare data                                                                  
       |                                                                                  
       |    +-------------------+                                                         
       +--> | driver_send_reply | prepare msg, send to arg peer                           
            +-------------------+                                                         
```

```
src/bus/driver.c                                          
+-------------------+                                      
| driver_send_reply | : prepare msg, send to arg peer      
+-|-----------------+                                      
  |    +----------------------+                            
  |--> | message_new_outgoing | prepare msg for sending out
  |    +----------------------+                            
  |    +---------------------+                             
  +--> | driver_send_unicast | send msg to arg receiver    
       +---------------------+                             
```

```
src/bus/driver.c                                                                         
+---------------------+                                                                   
| driver_send_unicast | : send msg to arg receiver                                        
+-|-------------------+                                                                   
  |    +----------------+                                                                 
  |--> | driver_monitor | parse metadata to get destinations, send msg to each destination
  |    +----------------+                                                                 
  |    +------------------+                                                               
  +--> | connection_queue | add msg to connection, add connnection to context             
       +------------------+                                                               
```

```
src/bus/driver.c                                                                    
+----------------+                                                                   
| driver_monitor | : parse metadata to get destinations, send msg to each destination
+-|--------------+                                                                   
  |    +------------------------+                                                    
  |--> | message_parse_metadata | validate msg header & body, truncate extra fds     
  |    +------------------------+                                                    
  |    +------------------------------+                                              
  |--> | bus_get_monitor_destinations | given metadata, get destinations             
  |    +------------------------------+                                              
  |                                                                                  
  +--> for each destination                                                          
       |                                                                             
       |--> get outer owner, then get outer peer (receiver)                          
       |                                                                             
       |    +---------------+                                                        
       |--> | c_list_unlink | remove destination from list                           
       |    +---------------+                                                        
       |    +------------------+                                                     
       +--> | connection_queue | add msg to connection, add connnection to context   
            +------------------+                                                     
```

```
driver_method_hello
```

```
driver_method_add_match
```

```
driver_method_remove_match
```

```
src/bus/driver.c                                                                      
+----------------------------+                                                         
| driver_method_request_name | : add ownership of name, send reply to peer             
+-|--------------------------+                                                         
  |    +-------------------+                                                           
  |--> | peer_request_name | add ownership of name to structures (peer tree, name list)
  |    +-------------------+                                                           
  |                                                                                    
  |--> if 'chagne'                                                                     
  |    |                                                                               
  |    |    +---------------------------+                                              
  |    |--> | driver_name_owner_changed | notify                                       
  |    |    +---------------------------+                                              
  |    |    +-----------------------+                                                  
  |    +--> | driver_name_activated | send msgs on lists of activation                 
  |         +-----------------------+                                                  
  |    +-------------------+                                                           
  +--> | driver_send_reply | prepare msg, send to arg peer                             
       +-------------------+                                                           
```

```
src/bus/peer.c                                                                            
+-------------------+                                                                      
| peer_request_name | : add ownership of name to structures (peer tree, name list)         
+----------------------------+                                                             
| name_registry_request_name | : add ownership of name to structures (peer tree, name list)
+----------------------------+                                                             
```

```
src/bus/name.c                                                                            
+----------------------------+                                                             
| name_registry_request_name | : add ownership of name to structures (peer tree, name list)
+-|--------------------------+                                                             
  |    +------------------------+                                                          
  |--> | name_registry_ref_name | given string, find name struct from registry and ref++   
  |    +------------------------+                                                          
  |    +--------------------+                                                              
  |--> | c_rbtree_find_slot | given name struct, find slot from ownership tree             
  |    +--------------------+                                                              
  |                                                                                        
  |--> if slot not found (there's already ownership)                                       
  |    -                                                                                   
  |    +--> given parent, get outer ownership                                              
  |                                                                                        
  |--> else                                                                                
  |    |                                                                                   
  |    |    +--------------------+                                                         
  |    |--> | name_ownership_new | new a ownership                                         
  |    |    +--------------------+                                                         
  |    |    +---------------------+                                                        
  |    +--> | name_ownership_link | insert ownership to tree                               
  |         +---------------------+                                                        
  |    +-----------------------+                                                           
  +--> | name_ownership_update | ensure arg ownership in the list of name, save changes    
       +-----------------------+                                                           
```

```
src/bus/name.c                                                                   
+-----------------------+                                                         
| name_ownership_update | : ensure arg ownership in the list of name, save changes
+-|---------------------+                                                         
  |                                                                               
  |--> get primary (first ownership from list of name)                            
  |                                                                               
  |--> if no primary                                                              
  |    |                                                                          
  |    |--> update 'change' struct                                                
  |    |                                                                          
  |    |    +-------------------+                                                 
  |    +--> | c_list_link_front | prepend arg ownership to list of name           
  |         +-------------------+                                                 
  |                                                                               
  |--> elif arg ownership is the primary already, do nothing                      
  |                                                                               
  +--> (skip other conditions)                                                    
```

```
src/bus/driver.c                                                                                           
+-----------------------+                                                                                   
| driver_name_activated | : send msgs on lists of activation                                                
+-|---------------------+                                                                                   
  |                                                                                                         
  |--> for each request in list of activation                                                               
  |    |                                                                                                    
  |    |    +-------------------------+                                                                     
  |    |--> | peer_registry_find_peer | given send id in request, find sender from bus peers                
  |    |    +-------------------------+                                                                     
  |    |                                                                                                    
  |    |--> if found                                                                                        
  |    |    |                                                                                               
  |    |    |    +---------------------------+                                                              
  |    |    |--> | driver_write_reply_header |                                                              
  |    |    |    +---------------------------+                                                              
  |    |    |    +-------------------+                                                                      
  |    |    +--> | driver_send_reply | prepare msg, send to arg peer                                        
  |    |         +-------------------+                                                                      
  |    |    +-------------------------+                                                                     
  |    +--> | activation_request_free | release request                                                     
  |         +-------------------------+                                                                     
  |                                                                                                         
  +--> for each msg in list of activation                                                                   
       |                                                                                                    
       |    +-------------------------+                                                                     
       |--> | peer_registry_find_peer | given send id in msg, find sender from bus peers                    
       |    +-------------------------+                                                                     
       |    +--------------------+                                                                          
       |--> | peer_queue_unicast | prepare reply if needed, add msg to connection, add connection to context
       |    +--------------------+                                                                          
       |    +-------------------------+                                                                     
       +--> | activation_message_free | release msg                                                         
            +-------------------------+                                                                     
```

```
src/bus/peer.c                                                                                   
+--------------------+                                                                            
| peer_queue_unicast | : prepare reply if needed, add msg to connection, add connection to context
+-|------------------+                                                                            
  |    +---------------------+                                                                    
  |--> | message_read_serial | read serial from msg                                               
  |    +---------------------+                                                                    
  |                                                                                               
  |--> if need reply                                                                              
  |    |                                                                                          
  |    |    +----------------+                                                                    
  |    +--> | reply_slot_new | prepare 'reply' and add to structures                              
  |         +----------------+                                                                    
  |    +------------------+                                                                       
  +--> | connection_queue | add msg to connection, add connnection to context                     
       +------------------+                                                                       
```

```
src/bus/reply.c                                                  
+----------------+                                                
| reply_slot_new | : prepare 'reply' and add to structures        
+-|--------------+                                                
  |                                                               
  |--> set up key                                                 
  |                                                               
  |    +--------------------+                                     
  |--> | c_rbtree_find_slot | given key, find slot from reply tree
  |    +--------------------+                                     
  |                                                               
  |--> alloc and init 'reply'                                     
  |                                                               
  |    +--------------+                                           
  |--> | c_rbtree_add | insert reply to reply tree of registry    
  |    +--------------+                                           
  |    +------------------+                                       
  +--> | c_list_link_tail | insert reply to reply list of owner   
       +------------------+                                       
```

```
driver_method_release_name
```

```
driver_method_get_connection_credentials
```

```
driver_method_get_connection_unix_user
```

```
driver_method_get_connection_unix_process_id
```

```
driver_method_get_adt_audit_session_data
```

```
driver_method_get_connection_selinux_security_context
```

```
driver_method_start_service_by_name
```

```
driver_method_list_queued_owners
```

```
driver_method_list_names
```

```
driver_method_list_activatable_names
```

```
driver_method_name_has_owner
```

```
driver_method_update_activation_environment
```

```
driver_method_get_name_owner
```

```
driver_method_reload_config
```

```
driver_method_get_id
```

```
driver_method_become_monitor
```

```
driver_method_introspect
```

```
driver_method_ping
```

```
driver_method_get_machine_id
```

```
driver_method_get
```

```
driver_method_set
```

```
driver_method_get_all
```

```
driver_method_get_stats
```

```
driver_method_get_connection_stats
```

```
driver_method_get_all_match_rules
```

```
src/bus/peer.c                                                                                                        
+---------------+                                                                                                      
| peer_dispatch | : get msg and validate it, handle it locally or unicast request to peer
+-|-------------+                                                                                                      
  |                                                                                                                    
  |--> given arg file, get outer peer                                                                                  
  |                                                                                                                    
  +--> for event in [poll-in, poll-hup | poll-out]                                                                     
       |                                                                                                               
       |    +--------------------------+                                                                               
       +--> | peer_dispatch_connection | in loop: get msg and validate it, handle it locally or unicast request to peer
            +--------------------------+                                                                               
```

```
src/bus/peer.c                                                                                              
+--------------------------+                                                                                 
| peer_dispatch_connection | : in loop: get msg and validate it, handle it locally or unicast request to peer
+-|------------------------+                                                                                 
  |    +---------------------+                                                                               
  |--> | connection_dispatch | handle arg events (in/out/hup) accordingly (read/write/discard)               
  |    +---------------------+                                                                               
  |                                                                                                          
  +--> endless loop                                                                                          
       |                                                                                                     
       |    +--------------------+                                                                           
       |--> | connection_dequeue | fetch msg from input buffer, return to caller                             
       |    +--------------------+                                                                           
       |    +------------------------+                                                                       
       |--> | message_parse_metadata | validate msg header & body, truncate extra fds                        
       |    +------------------------+                                                                       
       |    +-----------------------+                                                                        
       |--> | message_stitch_sender | stitch in new sender field                                             
       |    +-----------------------+                                                                        
       |    +-----------------+                                                                              
       +--> | driver_dispatch | given destination, handle it locally or unicast request to peer              
            +-----------------+                                                                              
```

```
src/bus/driver.c                                                                                  
+-----------------+                                                                                
| driver_dispatch | : given destination, handle it locally or unicast request to peer              
+-|---------------+                                                                                
  |    +--------------------------+                                                                
  +--> | driver_dispatch_internal | given destination, handle it locally or unicast request to peer
       +--------------------------+                                                                
```

```
src/bus/driver.c                                                                                                                   
+--------------------------+                                                                                                        
| driver_dispatch_internal | : given destination, handle it locally or unicast request to peer                                      
+-|------------------------+                                                                                                        
  |    +----------------+                                                                                                           
  |--> | driver_monitor | parse metadata to get destinations, send msg to each destination                                          
  |    +----------------+                                                                                                           
  |                                                                                                                                 
  |--> if destination == org.freedesktop.DBus                                                                                       
  |    |                                                                                                                            
  |    |    +---------------------------+                                                                                           
  |    +--> | driver_dispatch_interface | given interface/method strings, verify in-signature, call method->fn(), send reply to peer
  |         +---------------------------+                                                                                           
  |                                                                                                                                 
  |--> if msg has no destination && msg type is signal                                                                              
  |    |                                                                                                                            
  |    |    +--------------------------+                                                                                            
  |    +--> | driver_forward_broadcast | determine broadcast destinations, for each: queue a connection to context                  
  |         +--------------------------+                                                                                            
  |                                                                                                                                 
  +--> switch header type                                                                                                           
       case signal                                                                                                                  
       case method_call                                                                                                             
       -    +------------------------+                                                                                              
       +--> | driver_forward_unicast | queue connection to toward specific peer                                                     
            +------------------------+                                                                                              
       case method_return                                                                                                           
       case error                                                                                                                   
       -    +------------------+                                                                                                    
       +--> | peer_queue_reply | determine receiver, queue connection to context                                                    
            +------------------+                                                                                                    
```

```
src/bus/driver.c                                                                                                         
+---------------------------+                                                                                             
| driver_dispatch_interface | : given interface/method strings, verify in-signature, call method->fn(), send reply to peer
+-|-------------------------+                                                                                             
  |                                                                                                                       
  |--> if arg interface is provided                                                                                       
  |    |                                                                                                                  
  |    |--> find match from predefined interfaces                                                                         
  |    |                                                                                                                  
  |    |    +------------------------+                                                                                    
  |    +--> | driver_dispatch_method | verify in-signature, call method->fn(), send reply to arg peer                     
  |         +------------------------+                                                                                    
  |                                                                                                                       
  +--> else                                                                                                               
       -                                                                                                                  
       +--> for each interface in interfaces                                                                              
            |                                                                                                             
            |    +------------------------+                                                                               
            +--> | driver_dispatch_method | verify in-signature, call method->fn(), send reply to arg peer                
                 +------------------------+                                                                               
```

```
src/bus/driver.c                                                                                  
+------------------------+                                                                         
| driver_dispatch_method | : verify in-signature, call method->fn(), send reply to arg peer        
+-|----------------------+                                                                         
  |                                                                                                
  |--> find match from arg methods                                                                 
  |                                                                                                
  +--> if found                                                                                    
       |                                                                                           
       |    +----------------------+                                                               
       +--> | driver_handle_method | verify in-signature, call method->fn(), send reply to arg peer
            +----------------------+                                                               
```

```
src/bus/driver.c                                                                        
+----------------------+                                                                 
| driver_handle_method | : verify in-signature, call method->fn(), send reply to arg peer
+-|--------------------+                                                                 
  |    +---------------------------------+                                               
  |--> | driver_dvar_verify_signature_in |                                               
  |    +---------------------------------+                                               
  |    +---------------------------+                                                     
  |--> | driver_write_reply_header |                                                     
  |    +---------------------------+                                                     
  |                                                                                      
  +--> call method->fn(), e.g.,                                                          
       +--------------------------+                                                      
       | driver_method_introspect | given path, prepare data, send reply to arg peer     
       +--------------------------+                                                      
```

```
src/bus/driver.c                                                                                             
+--------------------------+                                                                                  
| driver_forward_broadcast | : determine broadcast destinations, for each: queue a connection to context      
+-|------------------------+                                                                                  
  |    +--------------------------------+                                                                     
  |--> | bus_get_broadcast_destinations | get broadcast destinations from a few sources (matches, sender, bus)
  |    +--------------------------------+                                                                     
  |                                                                                                           
  +--> for each match_owner in destinations                                                                   
       |                                                                                                      
       |--> given match_owner, get outer receiver                                                             
       |                                                                                                      
       |    +---------------+                                                                                 
       |--> | c_list_unlink | remove match_owner from destinations                                            
       |    +---------------+                                                                                 
       |    +------------------+                                                                              
       +--> | connection_queue | add msg to connection, add connnection to context                            
            +------------------+                                                                              
```

```
src/bus/bus.c                                                                                           
+--------------------------------+                                                                       
| bus_get_broadcast_destinations | : get broadcast destinations from a few sources (matches, sender, bus)
+-|------------------------------+                                                                       
  |                                                                                                      
  |--> if arg 'matches' is provided                                                                      
  |    |                                                                                                 
  |    |    +--------------------------------+                                                           
  |    +--> | match_registry_get_subscribers |                                                           
  |         +--------------------------------+                                                           
  |                                                                                                      
  |--> if arg 'sender' is provided                                                                       
  |    -                                                                                                 
  |    +--> get primary ownership from sender                                                            
  |         |                                                                                            
  |         |    +--------------------------------+                                                      
  |         +--> | match_registry_get_subscribers |                                                      
  |              +--------------------------------+                                                      
  |                                                                                                      
  +--> else                                                                                              
       |                                                                                                 
       |    +--------------------------------+                                                           
       +--> | match_registry_get_subscribers |                                                           
            +--------------------------------+                                                           
```

```
src/bus/driver.c                                                                                      
+------------------------+                                                                             
| driver_forward_unicast | : queue connection to toward specific peer                                  
+-|----------------------+                                                                             
  |    +-----------------------+                                                                       
  |--> | bus_find_peer_by_name | given name, get receiver                                              
  |    +-----------------------+                                                                       
  |                                                                                                    
  |--> if no receiver found                                                                            
  |    |                                                                                               
  |    |    +--------------------------+                                                               
  |    +--> | activation_queue_message | prepare msg and append to activation queue                    
  |         +--------------------------+                                                               
  |                                                                                                    
  |    +--------------------+                                                                          
  +--> | peer_queue_unicast | prepare reply if needed, add msg to connection, add connection to context
       +--------------------+                                                                          
```

```
src/bus/activation.c                                                             
+--------------------------+                                                      
| activation_queue_message | : prepare msg and append to activation queue         
+-|------------------------+                                                      
  |    +--------------------+                                                     
  |--> | activation_request | prepare msg of activation, add connection to context
  |    +--------------------+                                                     
  |                                                                               
  |--> alloc and init msg                                                         
  |                                                                               
  |    +------------------+                                                       
  +--> | c_list_link_tail | add msg to list end of activation                     
       +------------------+                                                       
```

```
src/bus/activation.c                                                                   
+--------------------+                                                                  
| activation_request | : prepare msg of activation, add connection to context           
+-|------------------+                                                                  
  |    +--------------------------+                                                     
  +--> | controller_name_activate | prepare msg of activation, add connection to context
       +--------------------------+                                                     
```

```
src/broker/controller.c                                                                  
+--------------------------+                                                              
| controller_name_activate | : prepare msg of activation, add connection to context       
+---------------------------------+                                                       
| controller_dbus_send_activation | : prepare msg of activation, add connection to context
+-|-------------------------------+                                                       
  |                                                                                       
  |--> prepare data                                                                       
  |                                                                                       
  |    +----------------------+                                                           
  |--> | message_new_outgoing | prepare msg for sending out                               
  |    +----------------------+                                                           
  |    +------------------+                                                               
  +--> | connection_queue | add msg to connection, add connnection to context             
       +------------------+                                                               
```

```
src/bus/driver.c                                                            
+------------------+                                                         
| peer_queue_reply | : determine receiver, queue connection to context       
+-|----------------+                                                         
  |                                                                          
  |--> given id/serial, get slot from sender                                 
  |                                                                          
  |--> given slot, get outer receiver (peer)                                 
  |                                                                          
  |    +------------------+                                                  
  +--> | connection_queue | add msg to connection, add connnection to context
       +------------------+                                                  
```

```
src/bus/listener.c                                                                           
+-------------------+                                                                         
| listener_dispatch | : prepare peer, handle msg locally or unicast request to peer           
+-|-----------------+                                                                         
  |                                                                                           
  |--> get outer listener                                                                     
  |                                                                                           
  |    +---------+                                                                            
  |--> | accept4 | accept connection request from service task?                               
  |    +---------+                                                                            
  |    +------------------+                                                                   
  |--> | peer_new_with_fd | prepare peer, get id from bus, add peer to bus                    
  |    +------------------+                                                                   
  |                                                                                           
  |--> append listener to list end of peer                                                    
  |                                                                                           
  |    +------------+                                                                         
  |--> | peer_spawn | add file to the end of context's link                                   
  |    +------------+                                                                         
  |    +---------------+                                                                      
  +--> | peer_dispatch | get msg and validate it, handle it locally or unicast request to peer
       +---------------+                                                                      
```

```
src/bus/peer.c                                                      
+------------------+                                                 
| peer_new_with_fd | : prepare peer, get id from bus, add peer to bus
+-|----------------+                                                 
  |    +---------------------+                                       
  |--> | sockopt_get_peersec | get sec label                         
  |    +---------------------+                                       
  |    +------------------------+                                    
  |--> | sockopt_get_peergroups | (credential related, skip)         
  |    +------------------------+                                    
  |                                                                  
  |--> alloc and init peer                                           
  |                                                                  
  |    +---------------------+                                       
  |--> | policy_snapshot_new | (credential related, skip)            
  |    +---------------------+                                       
  |    +------------------------+                                    
  +--> | connection_init_server | init connection and sasl server    
  |    +------------------------+ +---------------+                  
  |                               | peer_dispatch |                  
  |                               +---------------+                  
  |--> bus assigns id to peer                                        
  |                                                                  
  +--> given perer id, insert to peer tree of bus                    
```

```
src/launch/main.c                                                 
+------+                                                           
| main |                                                           
+-|----+                                                           
  |    +------------+                                              
  |--> | parse_argv | parse argumetns, e.g., --scope system --audit
  |    +------------+                                              
  |    +-------------+                                             
  |--> | inherit_fds | main_fd_listen = SD_LISTEN_FDS_START (3)    
  |    +-------------+                                             
  |                                                                
  |--> have signal mask = (SIGCHLD | SIGTERM | SIGINT | SIGHUP)    
  |                                                                
  |    +-------------+                                             
  |--> | sigprocmask | block them                                  
  |    +-------------+                                             
  |    +-----+                                                     
  +--> | run | prepare launcher and run it in loop                 
       +-----+                                                                                                       
```

```
src/launch/main.c                                                                      
+-------------+                                                                         
| inherit_fds | : main_fd_listen = SD_LISTEN_FDS_START (3)                              
+-|-----------+                                                                         
  |    +---------------+                                                                
  |--> | sd_listen_fds | for fd in target range, set FD_CLOEXEC to flags                
  |    +---------------+                                                                
  |                                                                                     
  |--> (we expect only 1 listener)                                                      
  |                                                                                     
  |    +--------------+                                                                 
  |--> | sd_is_socket | check if fd meets args (e.g., sock? STREAM? listening? PF_UNIX?)
  |    +--------------+                                                                 
  |                                                                                     
  |--> set NONBLOCK to fd flags                                                         
  |                                                                                     
  +--> main_fd_listen = SD_LISTEN_FDS_START (3)                                         
```

```
src/launch/main.c                                                                                                              
+-----+                                                                                                                         
| run | : prepare launcher and run it in loop                                                                                   
+-|---+                                                                                                                         
  |    +--------------+                                                                                                         
  |--> | launcher_new | prepare launcher (event with signal sources, bus)                                                       
  |    +--------------+                                                                                                         
  |    +--------------+                                                                                                         
  +--> | launcher_run | parse config, launcher dbus-broker, start bus, add filter/services/listener, connect to system bus, loop
       +--------------+                                                                                                                                                     
```

```
src/launch/launcher.c                                                                  
+--------------+                                                                        
| launcher_new | : prepare launcher (event with signal sources, bus)                    
+-|------------+                                                                        
  |                                                                                     
  |--> alloc and init launcher                                                          
  |                                                                                     
  |    +-------------------+                                                            
  |--> | launcher_open_log | create a socket connecting to "/run/systemd/journal/socket"
  |    +-------------------+                                                            
  |    +------------------+                                                             
  |--> | sd_event_default | ensure we have a default event, return its address          
  |    +------------------+                                                             
  |    +---------------------+                                                          
  |--> | sd_event_add_signal | add signal source for event                              
  |    +---------------------+ (for SIGTERM)                                            
  |    +---------------------+                                                          
  |--> | sd_event_add_signal | for SIGINT                                               
  |    +---------------------+                                                          
  |    +---------------------+                                                          
  |--> | sd_event_add_signal | for SIGHUP                                               
  |    +---------------------+                                                          
  |    +------------+                                                                   
  +--> | sd_bus_new | alloc and init bus                                                
       +------------+                                                                   
```

```
src/launch/launcher.c                                                             
+-------------------+                                                              
| launcher_open_log | : create a socket connecting to "/run/systemd/journal/socket"
+-|-----------------+                                                              
  |    +--------+                                                                  
  |--> | socket |                                                                  
  |    +--------+                                                                  
  |    +---------+                                                                 
  |--> | connect | connect to "/run/systemd/journal/socket"                        
  |    +---------+                                                                 
  |    +--------------------------+                                                
  +--> | log_init_journal_consume | init log with journal fd                       
       +--------------------------+                                                
```

```
src/launch/main.c                                                                                                         
+--------------+                                                                                                           
| launcher_run | : parse config, launcher dbus-broker, start bus, add filter/services/listener, connect to system bus, loop
+-|------------+                                                                                                           
  |    +-----------------------+                                                                                           
  |--> | launcher_parse_config | determine config file and parse it, save settings into launcher                           
  |    +-----------------------+                                                                                           
  |    +------------------------+                                                                                          
  |--> | launcher_load_services | given type, load service files accordingly                                               
  |    +------------------------+                                                                                          
  |    +----------------------+                                                                                            
  |--> | launcher_load_policy | (skip)                                                                                     
  |    +----------------------+                                                                                            
  |    +------------+                                                                                                      
  |--> | socketpair | create socket pair                                                                                   
  |    +------------+                                                                                                      
  |    +---------------+                                                                                                   
  |--> | sd_bus_set_fd | set bus input/output fd                                                                           
  |    +---------------+                                                                                                   
  |    +---------------+                                                                                                   
  |--> | launcher_fork | parent launches dbus-broker and prepares event monitoring it                                      
  |    +---------------+                                                                                                   
  |    +--------------------------+                                                                                        
  |--> | sd_bus_add_object_vtable | prepare node of path, add each vtable to bus hashmap, prepare slot and add to node     
  |    +--------------------------+                                                                                        
  |    +-------------------+                                                                                               
  |--> | sd_bus_add_filter | prepare slot for filter, prepend to list of bus                                               
  |    +-------------------+                                                                                               
  |    +--------------+                                                                                                    
  |--> | sd_bus_start | set bus state = opening, prepare socket/epoll, send msg (hello) to "org.freedesktop.DBus"          
  |    +--------------+                                                                                                    
  |    +-----------------------+                                                                                           
  |--> | launcher_add_services | add all services in launcher                                                              
  |    +-----------------------+                                                                                           
  |    +-----------------------+                                                                                           
  |--> | launcher_add_listener | send msg of method call (AddListener), append policy, send out and wait for reply         
  |    +-----------------------+                                                                                           
  |    +------------------+                                                                                                
  |--> | launcher_connect | connect launcher to, e.g., system bus                                                          
  |    +------------------+                                                                                                
  |    +---------------------+                                                                                             
  |--> | sd_bus_attach_event | prepare all kinds of sources for event                                                      
  |    +---------------------+ (for bus-controller)                                                                        
  |    +---------------------+                                                                                             
  |--> | sd_bus_attach_event | for bus-regular                                                                             
  |    +---------------------+                                                                                             
  |    +--------------------+                                                                                              
  |--> | launcher_subscribe | send of msg call ("Subscribe") to "org.freedesktop.systemd1"                                 
  |    +--------------------+                                                                                              
  |    +---------------+                                                                                                   
  +--> | sd_event_loop | loop: register event to epoll & wait & dispatch it                                                
       +---------------+                                                                                                   
```

```
src/launch/launcher.c                                                                     
+-----------------------+                                                                  
| launcher_parse_config | : determine config file and parse it, save settings into launcher
+-|---------------------+                                                                  
  |    +--------------+                                                                    
  |--> | dirwatch_new | prepare dir-watch                                                  
  |    +--------------+                                                                    
  |                                                                                        
  |--> determine config file, e.g., "/usr/share/dbus-1/system.conf"                        
  |                                                                                        
  |    +--------------------+                                                              
  |--> | config_parser_init | prepare xml parser                                           
  |    +--------------------+                                                              
  |    +--------------------+                                                              
  |--> | config_parser_read | parse config into tree structure                             
  |    +--------------------+                                                              
  |    +-----------------+                                                                 
  |--> | sd_event_add_io | prepare 'source' and add to arg 'event, register the source's io
  |    +-----------------+                                                                 
  |                                                                                        
  +--> traverse the tree and save settings in launcher
```

```
src/launch/launcher.c                                                                  
+------------------------+                                                              
| launcher_load_services | : given type, load service files accordingly                 
+-|----------------------+                                                              
  |                                                                                     
  +--> for each node in list                                                            
       |                                                                                
       |--> switch type                                                                 
       |                                                                                
       |--> case STANDARD_SESSION_SERVICEDIRS                                           
       |    |                                                                           
       |    |    +-----------------------------------------+                            
       |    +--> | launcher_load_standard_session_services | (skip)                     
       |         +-----------------------------------------+                            
       |                                                                                
       |--> case STANDARD_SYSTEM_SERVICEDIRS                                            
       |    |                                                                           
       |    |    +----------------------------------------+                             
       |    +--> | launcher_load_standard_system_services | load each found service file
       |         +----------------------------------------+                             
       |                                                                                
       +--> case SERVICEDIR                                                             
            |                                                                           
            |    +---------------------------+                                          
            +--> | launcher_load_service_dir | (skip)                                   
                 +---------------------------+                                          
```

```
src/launch/launcher.c                                                   
+----------------------------------------+                               
| launcher_load_standard_system_services | : load each found service file
+-|--------------------------------------+                               
  |                                                                      
  +--> for each folder ["/usr/local/share", "/usr/share", "/lib"]        
       |                                                                 
       |--> prepare path string = $folder/dbus-1/system-services         
       |                                                                 
       |    +---------------------------+                                
       +--> | launcher_load_service_dir | load each found service file   
            +---------------------------+                                
```

```
src/launch/launcher.c                                             
+---------------------------+                                      
| launcher_load_service_dir | : load each found service file       
+-|-------------------------+                                      
  |    +--------------+                                            
  |--> | dirwatch_add | add arg dirpath into our inotify watch list
  |    +--------------+                                            
  |                                                                
  +--> for each *.service in arg dirpath                           
       |                                                           
       |--> prepare file path = arg dirpath + entry name           
       |                                                           
       |    +----------------------------+                         
       +--> | launcher_load_service_file | read service file, ensure service struct is in launcher
            +----------------------------+                         
```

```
src/launch/launcher.c                                                                                       
+----------------------------+                                                                               
| launcher_load_service_file | : read service file, ensure service struct is in launcher                             
+-|--------------------------+                                                                               
  |    +--------------------------------+                                                                    
  |--> | launcher_ini_reader_parse_file | read and parse file        /usr/share/dbus-1/system-services:      
  |    +--------------------------------+                                                                    
  |                                                                  org.freedesktop.Avahi.service           
  |--> get entries, e.g.,                                            org.freedesktop.hostname1.service       
  |    name entry = xyz.openbmc_project.ObjectMapper                 org.freedesktop.network1.service        
  |    unit entry = dbus-xyz.openbmc_project.ObjectMapper.service    org.freedesktop.resolve1.service        
  |    exec_entry = /bin/false                                       org.freedesktop.systemd1.service        
  |    user entry = root                                             org.freedesktop.timedate1.service       
  |                                                                  org.freedesktop.timesync1.service       
  |--> validate entries                                              xyz.openbmc_project.Logging.service     
  |                                                                  xyz.openbmc_project.ObjectMapper.service
  |    +--------------------+                                                                                
  |--> | c_rbtree_find_slot | find target from services in launcher                                          
  |    +--------------------+                                                                                
  |                                                                                                          
  |--> if not found                                                                                          
  |    |                                                                                                     
  |    |    +-------------+                                                                                  
  |    +--> | service_new | prepare service, update info, add to launcher                                    
  |         +-------------+                                                                                  
  |                                                                                                          
  +--> else                                                                                                  
       |                                                                                                     
       |    +----------------+                                                                               
       +--> | service_update | update info                                                                   
            +----------------+                                                                               
```

```
src/launch/launcher.c                                                                
+---------------+                                                                     
| launcher_fork | : parent launches dbus-broker and prepares event monitoring it      
+-|-------------+                                                                     
  |    +------+                                                                       
  |--> | fork |                                                                       
  |    +------+                                                                       
  |                                                                                   
  |--> if we are the child (returned pid is 0)                                        
  |    |                                                                              
  |    |    +--------------------+                                                    
  |    +--> | launcher_run_child | prepare args and launch dbus-broker                
  |         +--------------------+                                                    
  |                                                                                   
  |    (parent reaches here)                                                          
  |                                                                                   
  |    +--------------------+                                                         
  +--> | sd_event_add_child | prepare source monitoring child task                    
       +--------------------+ +------------------------+                              
                              | launcher_on_child_exit | commit log and set exit event
                              +------------------------+                              
```

```
src/launch/launcher.c                                      
+--------------------+                                      
| launcher_run_child | : prepare args and launch dbus-broker
+-|------------------+                                      
  |    +----------------------+                             
  |--> | sd_id128_get_machine | get machine id              
  |    +----------------------+                             
  |                                                         
  |--> prepare strings for execve args                      
  |                                                         
  |    +--------+                                           
  +--> | execve | dbus-broker                               
       +--------+                                           
```

```
src/launch/launcher.c                                                                            
+-----------------------+                                                                         
| launcher_add_services | : add all services in launcher                                          
+-|---------------------+                                                                         
  |                                                                                               
  +--> for each service in launcher                                                               
       |                                                                                          
       |    +-------------+                                                                       
       +--> | service_add | ensure servcie is running (if not, send msg of 'Addname' to somewhere)
            +-------------+                                                                       
```

```
src/launch/service.c                                                                                              
+--------------+                                                                                                   
| service_add  | : ensure servcie is running (if not, send msg of 'Addname' to somewhere)                          
+-|------------+                                                                                                   
  |                                                                                                                
  |--> if service is already running, return                                                                       
  |                                                                                                                
  |--> prepare object string = /org/bus1/DBus/Name/$service_id                                                     
  |                                                                                                                
  |    +--------------------+                                                                                      
  |--> | sd_bus_call_method | prepare msg of 'method call' (AddName), append types to msg, send out, wait for reply
  |    +--------------------+                                                                                      
  |                                                                                                                
  +--> service->running = true                                                                                     
```

```
src/launch/launcher.c                                                                                                           
+-----------------------+                                                                                                        
| launcher_add_listener | : send msg of method call (AddListener), append policy, send out and wait for reply                    
+-|---------------------+                                                                                                        
  |    +--------------------------------+                                                                                        
  |--> | sd_bus_message_new_method_call | prepare msg of method_call (AddListener), append path/member/interface/destination info
  |    +--------------------------------+                                                                                        
  |    +-----------------------+                                                                                                 
  |--> | sd_bus_message_append | "/org/bus1/DBus/Listener/0"                                                                     
  |    +-----------------------+                                                                                                 
  |    +---------------+                                                                                                         
  |--> | policy_export | append policy to msg                                                                                    
  |    +---------------+                                                                                                         
  |    +-------------+                                                                                                           
  +--> | sd_bus_call | send msg out, wait for reply                                                                              
       +-------------+                                                                                                           
```

```
src/launch/launcher.c                                                           
+------------------+                                                             
| launcher_connect | : connect launcher to, e.g., system bus                     
+-|----------------+                                                             
  |                                                                              
  |--> if launcher is user scope (not our case)                                  
  |    |                                                                         
  |    |    +------------------+                                                 
  |    +--> | sd_bus_open_user | (skip)                                          
  |         +------------------+                                                 
  |                                                                              
  +--> else                                                                      
       |                                                                         
       |    +--------------------+                                               
       +--> | sd_bus_open_system | prepare bus, determine address and save in bus
            +--------------------+                                               
```

```
src/launch/launcher.c                                                                                                 
+---------------------+                                                                                                
| launcher_on_message |                                                                                                
+-|-------------------+                                                                                                
  |    +-------------------------+                                                                                     
  |--> | sd_bus_message_get_path | given msg, get path                                                                 
  |    +-------------------------+                                                                                     
  |    +---------------+                                                                                               
  |--> | string_prefix | get anything after '/org/bus1/DBus/Name/'                                                     
  |    +---------------+                                                                                               
  |                                                                                                                    
  |--> if got something                                                                                                
  |    -                                                                                                               
  |    +--> if msg is about signal                                                                                     
  |         |                                                                                                          
  |         |    +---------------------------+                                                                         
  |         +--> | launcher_on_name_activate | given arg id, find service from launcher and activate it                
  |              +---------------------------+                                                                         
  |                                                                                                                    
  +--> elif path == /org/bus1/DBus/Broker                                                                              
       |                                                                                                               
       |    +----------------------------------------+                                                                 
       +--> | launcher_on_set_activation_environment | given in-msg, prepare method call 'set environment' and send out
            +----------------------------------------+                                                                 
```

```
src/launch/launcher.c                                                                  
+---------------------------+                                                           
| launcher_on_name_activate | : given arg id, find service from launcher and activate it
+-|-------------------------+                                                           
  |    +---------------------+                                                          
  |--> | sd_bus_message_read | read serial from msg                                     
  |    +---------------------+                                                          
  |                                                                                     
  |--> given arg id, find service from launcher                                         
  |                                                                                     
  |--> if not found, print warning and return                                           
  |                                                                                     
  |    +------------------+                                                             
  +--> | service_activate | start unit or transient unit, monitor 'job removed' signal  
       +------------------+                                                             
```

```
src/launch/service.c                                                                                                                          
+------------------+                                                                                                                           
| service_activate | : start unit or transient unit, monitor 'job removed' signal                                                              
+-|----------------+                                                                                                                           
  |    +----------------------------+                                                                                                          
  |--> | service_discard_activation |                                                                                                          
  |    +----------------------------+                                                                                                          
  |                                                                                                                                            
  |--> if service has unit                                                                                                                     
  |    |                                                                                                                                       
  |    |    +--------------------+                                                                                                             
  |    +--> | service_start_unit | send method call (add unit), send method call (start unit), monitor 'job removed' signal                    
  |         +--------------------+                                                                                                             
  |                                                                                                                                            
  +--> elif service has multiple arguments                                                                                                     
       |                                                                                                                                       
       |    +------------------------------+                                                                                                   
       +--> | service_start_transient_unit | send method call (add unit), send method call (start transient unit), monitor 'job removed' signal
            +------------------------------+                                                                                                   
```

```
src/launch/service.c                                                                                                   
+--------------------+                                                                                                  
| service_start_unit | : send method call (add unit), send method call (start unit), monitor 'job removed' signal       
+-|------------------+                                                                                                  
  |    +--------------------+                                                                                           
  |--> | service_watch_jobs | register match-rule to 'JobRemoved' of systemd                                            
  |    +--------------------+                                                                                           
  |    +--------------------+                                                                                           
  |--> | service_watch_unit | send method call (add unit), install callback to add match rules                          
  |    +--------------------+                                                                                           
  |    +--------------------------------+                                                                               
  |--> | sd_bus_message_new_method_call | prepare msg of method_call type, append path/member/interface/destination info
  |    +--------------------------------+ (StartUnit)                                                                   
  |    +-----------------------+                                                                                        
  |--> | sd_bus_message_append | append 'unit' to msg                                                                   
  |    +-----------------------+                                                                                        
  |    +-------------------+                                                                                            
  +--> | sd_bus_call_async | prepare slot (install callback, insert to hashmap/prioq), send msg out                     
       +-------------------+ +----------------------------+                                                             
                             | service_start_unit_handler | read 'job' from msg and save in service                     
                             +----------------------------+                                                             
```

```
src/launch/service.c                                                                                                
+--------------------+                                                                                               
| service_watch_jobs | : register match-rule to 'JobRemoved' of systemd                                              
+-|------------------+                                                                                               
  |    +---------------------------+                                                                                 
  +--> | sd_bus_match_signal_async | send msg ('add match' method) to bus clients, add match rule to bus             
       +---------------------------+ +----------------------------+                                                  
                                     | service_watch_jobs_handler | if job failed, send method call 'Reset' to broker
                                     +----------------------------+                                                  
```

```
src/launch/service.c                                                                 
+----------------------------+                                                        
| service_watch_jobs_handler | : if job failed, send method call 'Reset' to broker    
+-|--------------------------+                                                        
  |    +---------------------+                                                        
  |--> | sd_bus_message_read | read id/path/unit/result from msg                      
  |    +---------------------+                                                        
  |                                                                                   
  |--> if result == done || result == skipped                                         
  |    -                                                                              
  |    +--> unref service job                                                         
  |                                                                                   
  +--> else                                                                           
       |                                                                              
       |    +--------------------------+                                              
       +--> | service_reset_activation | thru controller bus, send method call 'Reset'
            +--------------------------+                                              
```

```
src/launch/service.c                                                       
+--------------------------+                                                
| service_reset_activation | : thru controller bus, send method call 'Reset'
+-|------------------------+                                                
  |    +----------------------------+                                       
  |--> | service_discard_activation |                                       
  |    +----------------------------+                                       
  |                                                                         
  |--> prepare string = /org/bus1/DBus/Name/$id                             
  |                                                                         
  |    +--------------------+                                               
  +--> | sd_bus_call_method | Reset                                         
       +--------------------+                                               
```

```
src/launch/service.c                                                                                                
+--------------------+                                                                                               
| service_watch_unit | : send method call (add unit), install callback to add match rules                            
+-|------------------+                                                                                               
  |    +--------------------------+                                                                                  
  +--> | sd_bus_call_method_async | prepare msg (method call), install callback, send msg out                        
       +--------------------------+ +---------------------------------+                                              
                                    | service_watch_unit_load_handler |                                              
                                    +---------------------------------+                                              
                                    read obj_path from msg, send msg (method call, add match), add match rules to bus
```

```
src/launch/service.c                                                                                                                        
+---------------------------------+                                                                                                          
| service_watch_unit_load_handler | : read obj_path from msg, send msg (method call, add match), add match rules to bus                      
+-|-------------------------------+                                                                                                          
  |    +---------------------+                                                                                                               
  |--> | sd_bus_message_read | read obj_path from msg                                                                                        
  |    +---------------------+                                                                                                               
  |    +---------------------------+                                                                                                         
  +--> | sd_bus_match_signal_async | send msg ('add match' method) to bus clients, add match rule to bus                                     
       +---------------------------+ +----------------------------+                                                                          
                                     | service_watch_unit_handler | check msg, if condition met: send method call 'Reset' thru controller bus
                                     +----------------------------+                                                                          
```

```
src/launch/service.c                                                                                     
+----------------------------+                                                                            
| service_watch_unit_handler | : check msg, if condition met: send method call 'Reset' thru controller bus
+-|--------------------------+                                                                            
  |    +---------------------+                                                                            
  |--> | sd_bus_message_read | read interface from msg                                                    
  |    +---------------------+                                                                            
  |                                                                                                       
  |--> while not end of msg                                                                               
  |    |                                                                                                  
  |    |    +---------------------+                                                                       
  |    |--> | sd_bus_message_read | read property from msg                                                
  |    |    +---------------------+                                                                       
  |    |                                                                                                  
  |    |--> if property == 'ActiveState'                                                                  
  |    |    |                                                                                             
  |    |    |    +---------------------+                                                                  
  |    |    +--> | sd_bus_message_read | read value from msg                                              
  |    |         +---------------------+                                                                  
  |    |                                                                                                  
  |    |--> elif property == 'ConditionResult'                                                            
  |    |    |                                                                                             
  |    |    |    +---------------------+                                                                  
  |    |    +--> | sd_bus_message_read | read condition_result from msg                                   
  |    |         +---------------------+                                                                  
  |    |                                                                                                  
  |    +--> else                                                                                          
  |         |                                                                                             
  |         |    +---------------------+                                                                  
  |         +--> | sd_bus_message_skip |                                                                  
  |              +---------------------+                                                                  
  |                                                                                                       
  +--> if value == failed || condition_result == false                                                    
       |                                                                                                  
       |    +--------------------------+                                                                  
       +--> | service_reset_activation | thru controller bus, send method call 'Reset'                    
            +--------------------------+                                                                  
```

```
src/launch/service.c                                                   
+----------------------------+                                          
| service_start_unit_handler | : read 'job' from msg and save in service
+-|--------------------------+                                          
  |    +---------------------+                                          
  |--> | sd_bus_message_read | read 'job' from msg                      
  |    +---------------------+                                          
  |                                                                     
  +--> service->job = job                                               
```

```
src/launch/service.c                                                                                                                
+------------------------------+                                                                                                     
| service_start_transient_unit | : send method call (add unit), send method call (start transient unit), monitor 'job removed' signal
+-|----------------------------+                                                                                                     
  |    +--------------------+                                                                                                        
  |--> | service_watch_jobs | register match-rule to 'JobRemoved' of systemd                                                         
  |    +--------------------+                                                                                                        
  |    +------------------------+                                                                                                    
  |--> | sd_bus_get_unique_name | get unique name of bus                                                                             
  |    +------------------------+                                                                                                    
  |    +---------------------+                                                                                                       
  |--> | systemd_escape_unit | escape the unique name                                                                                
  |    +---------------------+                                                                                                       
  |                                                                                                                                  
  |--> determine unit name                                                                                                           
  |                                                                                                                                  
  |    +--------------------+                                                                                                        
  |--> | service_watch_unit | send method call (add unit), install callback to add match rules                                       
  |    +--------------------+                                                                                                        
  |    +--------------------------------+                                                                                            
  |--> | sd_bus_message_new_method_call | prepare msg of method_call type, append path/member/interface/destination info             
  |    +--------------------------------+ (StartTransientUnit)                                                                       
  |                                                                                                                                  
  |--> append content (as if it's a service file) to msg                                                                             
  |                                                                                                                                  
  |    +-------------------+                                                                                                         
  +--> | sd_bus_call_async | prepare slot (install callback, insert to hashmap/prioq), send msg out                                  
       +-------------------+ +----------------------------+                                                                          
                             | service_start_unit_handler | read 'job' from msg and save in service                                  
                             +----------------------------+                                                                          
```

```
src/launch/launcher.c                                                                                                  
+----------------------------------------+                                                                              
| launcher_on_set_activation_environment | : given in-msg, prepare method call 'set environment' and send out           
+-|--------------------------------------+                                                                              
  |    +--------------------------------+                                                                               
  |--> | sd_bus_message_new_method_call | prepare msg of method_call type, append path/member/interface/destination info
  |    +--------------------------------+ (SetEnvironment)                                                              
  |                                                                                                                     
  |--> while not at end of input msg                                                                                    
  |    |                                                                                                                
  |    |    +-----------------------+                                                                                   
  |    +--> | sd_bus_message_append | append 'key = value' to output msg                                                
  |         +-----------------------+                                                                                   
  |    +-------------------+                                                                                            
  +--> | sd_bus_call_async | prepare slot (install callback, insert to hashmap/prioq), send msg out                     
       +-------------------+ +----------------------------------+                                                       
                             | launcher_set_environment_handler | commit error log if there's any                       
                             +----------------------------------+                                                       
```

```
src/bus/driver.c                                                   
static const DriverInterface interfaces[] = {                      
    { "org.freedesktop.DBus", driver_methods },                       
    { "org.freedesktop.DBus.Monitoring", monitoring_methods },        
    { "org.freedesktop.DBus.Introspectable", introspectable_methods },
    { "org.freedesktop.DBus.Peer", peer_methods },                    
    { "org.freedesktop.DBus.Properties", properties_methods },        
    { "org.freedesktop.DBus.Debug.Stats", debug_stats_methods },      
};                                                                 
```

```
src/bus/driver.c                                                                                              
static const DriverMethod driver_methods[] = {                                                                
    { "Hello",                                  ... driver_method_hello, ...                                  
    { "AddMatch",                               ... driver_method_add_match, ...                              
    { "RemoveMatch",                            ... driver_method_remove_match, ...                           
    { "RequestName",                            ... driver_method_request_name, ...                           
    { "ReleaseName",                            ... driver_method_release_name, ...                           
    { "GetConnectionCredentials",               ... driver_method_get_connection_credentials, ...             
    { "GetConnectionUnixUser",                  ... driver_method_get_connection_unix_user, ...               
    { "GetConnectionUnixProcessID",             ... driver_method_get_connection_unix_process_id, ...         
    { "GetAdtAuditSessionData",                 ... driver_method_get_adt_audit_session_data, ...             
    { "GetConnectionSELinuxSecurityContext",    ... driver_method_get_connection_selinux_security_context, ...
    { "StartServiceByName",                     ... driver_method_start_service_by_name, ...                  
    { "ListQueuedOwners",                       ... driver_method_list_queued_owners, ...                     
    { "ListNames",                              ... driver_method_list_names, ...                             
    { "ListActivatableNames",                   ... driver_method_list_activatable_names, ...                 
    { "NameHasOwner",                           ... driver_method_name_has_owner, ...                         
    { "UpdateActivationEnvironment",            ... driver_method_update_activation_environment, ...          
    { "GetNameOwner",                           ... driver_method_get_name_owner, ...                         
    { "ReloadConfig",                           ... driver_method_reload_config, ...                          
    { "GetId",                                  ... driver_method_get_id, ...                                 
    { },                                                                                                      
};                                                                                                            
```

```
src/bus/driver.c                                                                                                
+------------------------------+                                                                                 
| driver_method_get_name_owner | : given service name, find owner and write its unique id to out data, send reply
+-|----------------------------+                                                                                 
  |    +-------------+                                                                                           
  |--> | c_dvar_read | get argument (service_name)                                                               
  |    +-------------+                                                                                           
  |                                                                                                              
  |--> if service_name == "org.freedesktop.DBus"                                                                 
  |    -                                                                                                         
  |    +--> addr = "org.freedesktop.DBus"                                                                        
  |                                                                                                              
  |--> else                                                                                                      
  |    |                                                                                                         
  |    |    +-----------------------+                                                                            
  |    +--> | bus_find_peer_by_name | given service_name, find owner (peer)                                      
  |         +-----------------------+                                                                            
  |    +--------------+                                                                                          
  |--> | c_dvar_write | write unique id of owner to out data                                                     
  |    +--------------+                                                                                          
  |    +-------------------+                                                                                     
  +--> | driver_send_reply |                                                                                     
       +-------------------+                                                                                     
```

```
src/bus/driver.c                                                                  
+--------------------------+                                                       
| driver_method_list_names | : prepare list of uniq_id and servcie_name, send reply
+-|------------------------+                                                       
  |    +--------------+                                                            
  |--> | c_dvar_write | write "org.freedesktop.DBus" to out data                   
  |    +--------------+                                                            
  |                                                                                
  |--> for each peer in bus                                                        
  |    -                                                                           
  |    +--> write its uniq id to out data                                          
  |                                                                                
  |--> for each service_name in bus                                                
  |    -                                                                           
  |    +--> write the name to out data                                             
  |                                                                                
  |    +-------------------+                                                       
  +--> | driver_send_reply | send reply                                            
       +-------------------+                                                       
```
