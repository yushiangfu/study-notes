```
src/resolve/resolved.c                                                      
+-----+                                                                      
| run |                                                                      
+-|---+                                                                      
  |    +--------------------+                                                
  |--> | service_parse_argv |                                                
  |    +--------------------+                                                
  |    +------------------+                                                  
  |--> | mkdir_safe_label | "/run/systemd/resolve"                           
  |    +------------------+                                                  
  |    +-------------+                                                       
  |--> | manager_new | setup manager, register callbacks for inotify and rtnl
  |    +-------------+                                                       
  |    +---------------+                                                     
  |--> | manager_start | (skip)                                              
  |    +---------------+                                                     
  |    +---------------------------+                                         
  |--> | manager_write_resolv_conf | update resolv.conf                      
  |    +---------------------------+                                         
  |    +---------------------------+                                         
  |--> | manager_check_resolv_conf | (skip)                                  
  |    +---------------------------+                                         
  |    +------------------------------+                                      
  |--> | capability_bounding_set_drop | (skip)                               
  |    +------------------------------+                                      
  |    +--------------+                                                      
  |--> | notify_start |                                                      
  |    +--------------+                                                      
  |    +---------------+                                                     
  +--> | sd_event_loop |                                                     
       +---------------+                                                     
```

```
src/resolve/resolved-manager.c                                                                                          
+-------------+                                                                                                          
| manager_new | : setup manager, register callbacks for inotify and rtnl                                                 
+-|-----------+                                                                                                          
  |                                                                                                                      
  |--> alloc and setup manager                                                                                           
  |                                                                                                                      
  |    +---------------------------+                                                                                     
  |--> | manager_parse_config_file | parse /etc/systemd/resolved.conf                                                    
  |    +---------------------------+                                                                                     
  |    +------------------+                                                                                              
  |--> | sd_event_default |                                                                                              
  |    +------------------+                                                                                              
  |    +-----------------------+                                                                                         
  |--> | sd_event_set_watchdog | given arg b, arm or unarm watchdog of event                                             
  |    +-----------------------+                                                                                         
  |    +------------------------+                                                                                        
  |--> | manager_watch_hostname | setup monitor for "/proc/sys/kernel/hostname", get hostname thru uname                 
  |    +------------------------+                                                                                        
  |    +------------+                                                                                                    
  |--> | dnssd_load | (find no *.dnssd)                                                                                  
  |    +------------+                                                                                                    
  |    +---------------+                                                                                                 
  |--> | dns_scope_new | alloc scope, add to manager                                                                     
  |    +---------------+                                                                                                 
  |    +--------------------------------+                                                                                
  |--> | manager_network_monitor_listen | setup monitor for /run/systemd/netif/links/                                    
  |    +--------------------------------+                                                                                
  |    +---------------------+                                                                                           
  |--> | manager_rtnl_listen | open socket, register matches, get all links/addresses and process them                   
  |    +---------------------+                                                                                           
  |    +---------------------+                                                                                           
  +--> | manager_connect_bus | register vtables, request name "org.freedesktop.login1", register callbacks for match rule
       +---------------------+                                                                                           
```

```
src/resolve/resolved-manager.c                                                                    
+------------------------+                                                                         
| manager_watch_hostname | : setup monitor for "/proc/sys/kernel/hostname", get hostname thru uname
+-|----------------------+                                                                         
  |    +-----------------+                                                                         
  |--> | sd_event_add_io |                                                                         
  |    +-----------------+ +--------------------+                                                  
  |                        | on_hostname_change |                                                  
  |                        +--------------------+                                                  
  |                                                                                                
  |    +---------------------+                                                                     
  +--> | determine_hostnames | get hostname thru uname                                             
       +---------------------+                                                                     
```

```
src/resolve/resolved-manager.c                          
+---------------------+                                  
| determine_hostnames | : get hostname thru uname        
+|--------------------+                                  
 |    +-------------------------+                        
 +--> | resolve_system_hostname | get hostname thru uname
      +-------------------------+                        
```

```
src/resolve/resolved-dns-scope.c                              
+---------------+                                              
| dns_scope_new | : alloc scope, add to manager                
+-|-------------+                                              
  |                                                            
  |--> alloc scope                                             
  |                                                            
  |--> add scope to manager                                    
  |                                                            
  |    +----------------------------+                          
  |--> | dns_scope_llmnr_membership | (do nothing in our case) 
  |    +----------------------------+  (do nothing in our case)
  |    +---------------------------+                           
  +--> | dns_scope_mdns_membership |                           
       +---------------------------+                           
```

```
src/resolve/resolved-manager.c                                                                                  
+--------------------------------+                                                                               
| manager_network_monitor_listen | : setup monitor for /run/systemd/netif/links/                                 
+-|------------------------------+                                                                               
  |    +------------------------+                                                                                
  |--> | sd_network_monitor_new | setup inotify watching "/run/systemd/netif/links/"                             
  |    +------------------------+                                                                                
  |    +---------------------------+                                                                             
  |--> | sd_network_monitor_get_fd | get inotify fd                                                              
  |    +---------------------------+                                                                             
  |    +-------------------------------+                                                                         
  |--> | sd_network_monitor_get_events | return 'pollin'                                                         
  |    +-------------------------------+                                                                         
  |    +-----------------+                                                                                       
  +--> | sd_event_add_io |                                                                                       
       +-----------------+ +------------------+                                                                  
                           | on_network_event | update links in manager, update resolv.conf, emit property change
                           +------------------+                                                                  
```

```
src/resolve/resolved-manager.c                                                         
+------------------+                                                                    
| on_network_event | : update links in manager, update resolv.conf, emit property change
+-|----------------+                                                                    
  |    +--------------------------+                                                     
  |--> | sd_network_monitor_flush | given inotify events, add or remove watch           
  |    +--------------------------+                                                     
  |                                                                                     
  |--> for each link in manager                                                         
  |    |                                                                                
  |    |    +-------------+                                                             
  |    +--> | link_update | given msg, update link                                      
  |         +-------------+                                                             
  |    +---------------------------+                                                    
  |--> | manager_write_resolv_conf | update resolv.conf                                 
  |    +---------------------------+                                                    
  |    +----------------------+                                                         
  +--> | manager_send_changed | emit property change of 'DNS'                           
       +----------------------+                                                         
```

```
src/resolve/resolved-resolv-conf.c                                                                
+---------------------------+                                                                      
| manager_write_resolv_conf | : update resolv.conf                                                 
+-|-------------------------+                                                                      
  |    +--------------------------+                                                                
  |--> | manager_read_resolv_conf | parse /etc/resolv.conf, alloc dns server/domain, add to manager
  |    +--------------------------+                                                                
  |    +-----------------------------+                                                             
  |--> | manager_compile_dns_servers | find servers from manager/links, add to arg dns             
  |    +-----------------------------+                                                             
  |    +--------------------------------+                                                          
  |--> | manager_compile_search_domains | find domains from manager/links, add to arg dns          
  |    +--------------------------------+                                                          
  |    +-----------------------+                                                                   
  |--> | fopen_temporary_label | open a temporary file                                             
  |    +-----------------------+                                                                   
  |    +-----------------------------------+                                                       
  |--> | write_uplink_resolv_conf_contents | write resolve content to the file                     
  |    +-----------------------------------+                                                       
  |    +---------------------+                                                                     
  +--> | conservative_rename | rename the file to "/run/systemd/resolve/resolv.conf"               
       +---------------------+                                                                     
```

```
src/resolve/resolved-resolv-conf.c                                                                   
+--------------------------+                                                                          
| manager_read_resolv_conf | : parse /etc/resolv.conf, alloc dns server/domain, add to manager        
+-|------------------------+                                                                          
  |                                                                                                   
  |--> open "/etc/resolv.conf"                                                                        
  |                                                                                                   
  |--> endless loop (break when nothing to read)                                                      
  |    |                                                                                              
  |    |    +-----------+                                                                             
  |    |--> | read_line |                                                                             
  |    |    +-----------+                                                                             
  |    |                                                                                              
  |    |--> get value of "nameserver", alloc 'dns server' and add to manager                          
  |    |                                                                                              
  |    +--> get value of "domain", alloc 'dns search domain' and add to manager                       
  |                                                                                                   
  |    +------------------------+                                                                     
  |--> | manager_set_dns_server | appoint the first server as current dns server, emit property change
  |    +------------------------+                                                                     
  |    +-------------------------------+                                                              
  +--> | dns_server_reset_features_all | reset all servers                                            
       +-------------------------------+                                                              
```

```
src/resolve/resolved-manager.c                                                                                
+---------------------+                                                                                        
| manager_rtnl_listen | : open socket, register matches, get all links/addresses and process them              
+-|-------------------+                                                                                        
  |    +-----------------+                                                                                     
  |--> | sd_netlink_open | open netlink socket                                                                 
  |    +-----------------+                                                                                     
  |    +-------------------------+                                                                             
  |--> | sd_netlink_attach_event | setup two sources: "netlink-receive-message" & "netlink-timer", add to event
  |    +-------------------------+                                                                             
  |    +----------------------+                                                                                
  +--> | sd_netlink_add_match | 'new link' & 'del link'                                                        
  |    +----------------------+ +----------------------+                                                       
  |                             | manager_process_link |                                                       
  |                             +----------------------+                                                       
  |    +----------------------+                                                                                
  |--> | sd_netlink_add_match | 'new addr' & 'del addr'                                                        
  |    +----------------------+ +-------------------------+                                                    
  |                             | manager_process_address |                                                    
  |                             +-------------------------+                                                    
  |                                                                                                            
  |--> send rtnl request to get all links                                                                      
  |                                                                                                            
  |--> for each reply (link)                                                                                   
  |    |                                                                                                       
  |    |    +----------------------+                                                                           
  |    +--> | manager_process_link | handle 'new link' or 'del link'                                           
  |         +----------------------+                                                                           
  |                                                                                                            
  |--> send rtnl request to get all addresses                                                                  
  |                                                                                                            
  +--> for each reply (address)                                                                                
       |                                                                                                       
       |    +-------------------------+                                                                        
       +--> | manager_process_address | handle 'new addr' or 'del addr'                                        
            +-------------------------+                                                                        
```

```
src/resolve/resolved-bus.c                                                                                         
+---------------------+                                                                                             
| manager_connect_bus | : register vtables, request name "org.freedesktop.login1", register callbacks for match rule
+-|-------------------+                                                                                             
  |    +---------------------------------------------+                                                              
  |--> | bus_open_system_watch_bind_with_description |                                                              
  |    +---------------------------------------------+                                                              
  |    +------------------------+                                                                                   
  |--> | bus_add_implementation | register manager_object and vtable to dbus                                        
  |    +------------------------+                                                                                   
  |    +---------------------------+                                                                                
  |--> | sd_bus_request_name_async | "org.freedesktop.resolve1"                                                     
  |    +---------------------------+                                                                                
  |    +---------------------+                                                                                      
  |--> | sd_bus_attach_event |                                                                                      
  |    +---------------------+                                                                                      
  |    +---------------------------+                                                                                
  +--> | sd_bus_match_signal_async | register match callback for org.freedesktop.login1                             
       +---------------------------+                                                                                
```
