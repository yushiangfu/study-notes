### dbus-broker

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

### dbus-broker-launch

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
       +--> | launcher_load_service_file | (skip)                  
            +----------------------------+                         
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
