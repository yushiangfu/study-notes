```
src/network/networkd.c                                                                                                     
+-----+                                                                                                                     
| run | send all kinds of requests and add links/qdisk/tclass/addresses/neighbors/nexthop/routes/rules                      
+|----+                                                                                                                     
 |    +--------------------+                                                                                                
 |--> | service_parse_argv | (skip, no argument passed in our scenario)                                                     
 |    +--------------------+                                                                                                
 |    +------------------+                                                                                                  
 |--> | mkdir_safe_label | "/run/systemd/netif"                                                                             
 |    +------------------+                                                                                                  
 |    +------------------+                                                                                                  
 |--> | mkdir_safe_label | "/run/systemd/netif/links"                                                                       
 |    +------------------+                                                                                                  
 |    +------------------+                                                                                                  
 |--> | mkdir_safe_label | "/run/systemd/netif/leases"                                                                      
 |    +------------------+                                                                                                  
 |    +------------------+                                                                                                  
 |--> | mkdir_safe_label | "/run/systemd/netif/lldp"                                                                        
 |    +------------------+                                                                                                  
 |    +-------------+                                                                                                       
 |--> | manager_new | alloc and setup manager                                                                               
 |    +-------------+                                                                                                       
 |    +---------------+                                                                                                     
 |--> | manager_setup | register callback forr post source, connect rtnl, setup dbus, prepare 'resolve', alloc address pools
 |    +---------------+                                                                                                     
 |    +---------------------------+                                                                                         
 |--> | manager_parse_config_file | parse /etc/systemd/networkd.conf                                                        
 |    +---------------------------+                                                                                         
 |    +---------------------+                                                                                               
 |--> | manager_load_config | load *.netdev & *.network                                                                     
 |    +---------------------+                                                                                               
 |    +-------------------+                                                                                                 
 |--> | manager_enumerate | send all kinds of requests and add links/qdisk/tclass/addresses/neighbors/nexthop/routes/rules  
 |    +-------------------+                                                                                                 
 |    +---------------+                                                                                                     
 |--> | manager_start | start speed meter, save manager and links                                                           
 |    +---------------+                                                                                                     
 |    +---------------+                                                                                                     
 +--> | sd_event_loop |                                                                                                     
      +---------------+                                                                                                     
```

```
src/network/networkd-manager.c                                                                                
+-----------------------+                                                                                      
| manager_dirty_handler | : save states of manager to file, save state of dirty links to file                  
+-|---------------------+                                                                                      
  |                                                                                                            
  |--> if manager is dirty                                                                                     
  |    |                                                                                                       
  |    |    +--------------+                                                                                   
  |    +--> | manager_save | save current manager settings to file, update manager settings, clear 'dirty' flag
  |         +--------------+                                                                                   
  |                                                                                                            
  +--> for each dirty link in manager                                                                          
       |                                                                                                       
       |    +---------------------+                                                                            
       +--> | link_save_and_clean | save link states to file, remove link from manager                         
            +---------------------+                                                                            
```

```
src/network/networkd-state-file.c                                                                   
+--------------+                                                                                     
| manager_save | : save current manager settings to file, update manager settings, clear 'dirty' flag
+-|------------+                                                                                     
  |                                                                                                  
  |--> for each link in manager                                                                      
  |    |                                                                                             
  |    |--> if it's loop-back coninue                                                                
  |    |                                                                                             
  |    +--> determine a few settings (dns, ntp, search_domains, ...)                                 
  |                                                                                                  
  |--> get a few strings (operation, carrier, address, ...)                                          
  |                                                                                                  
  |    +-----------------+                                                                           
  |--> | fopen_temporary |                                                                           
  |    +-----------------+                                                                           
  |                                                                                                  
  |--> write strings to the tmp file                                                                 
  |                                                                                                  
  |--> write settings to the tmp file                                                                
  |                                                                                                  
  |    +---------------------+                                                                       
  |--> | conservative_rename | rename file = manager->state_file                                     
  |    +---------------------+                                                                       
  |                                                                                                  
  |--> update manager settings                                                                       
  |                                                                                                  
  +--> manager->dirty = false                                                                        
```

```
src/network/networkd-state-file.c                                            
+---------------------+                                                      
| link_save_and_clean | : save link states to file, remove link from manager 
+-|-------------------+                                                      
  |    +-----------+                                                         
  |--> | link_save | save link states to tmp file, rename to link->state_file
  |    +-----------+                                                         
  |    +------------+                                                        
  +--> | link_clean | remove the dirty link from manager                     
       +------------+                                                        
```

```
src/network/networkd-queue.c                                                                    
+--------------------------+                                                                     
| manager_process_requests | : for each request in manager: process and detatch from manager     
+-|------------------------+                                                                     
  |                                                                                              
  +--> endless loop                                                                              
       |                                                                                         
       |--> for each request in queue of manager                                                 
       |    |                                                                                    
       |    |--> call ->process(), e.g.,                                                         
       |    |    +------------------------------------+                                          
       |    |    | independent_netdev_process_request | prepare msg and send out to create netdev
       |    |    +------------------------------------+                                          
       |    |                                                                                    
       |    |    +----------------+                                                              
       |    +--> | request_detach | detatch request from manager                                 
       |         +----------------+                                                              
       |                                                                                         
       +--> break                                                                                
```

```
src/network/netdev/netdev.c                                                      
+------------------------------------+                                            
| independent_netdev_process_request | : prepare msg and send out to create netdev
+-|----------------------------------+                                            
  |    +---------------------------+                                              
  |--> | netdev_is_ready_to_create | check if netdev is ready to create (what?)   
  |    +---------------------------+                                              
  |    +---------------------------+                                              
  +--> | independent_netdev_create | prepare msg and send out to create netdev    
       +---------------------------+                                              
```

```
src/network/netdev/netdev.c                                             
+---------------------------+                                            
| independent_netdev_create | : prepare msg and send out to create netdev
+-|-------------------------+                                            
  |                                                                      
  |--> if ->create() exists                                              
  |    -                                                                 
  |    +--> call ->create()                                              
  |                                                                      
  |    +--------------------------+                                      
  |--> | sd_rtnl_message_new_link | alloc msg for rtnl                   
  |    +--------------------------+                                      
  |    +-----------------------+                                         
  |--> | netdev_create_message | append info to msg                      
  |    +-----------------------+                                         
  |    +--------------------+                                            
  |--> | netlink_call_async | send out msg (?)                           
  |    +--------------------+                                            
  |                                                                      
  +--> state = creating                                                  
```

```
src/libsystemd/sd-netlink/sd-netlink.c                                          
+-----------------+                                                              
| sd_netlink_open | : open socket, alloc netlink and setup, bind socket to it    
+---------------------+                                                          
| netlink_open_family | : open socket, alloc netlink and setup, bind socket to it
+-|-------------------+                                                          
  |    +-------------+                                                           
  |--> | socket_open | open socket of arg family                                 
  |    +-------------+                                                           
  |    +--------------------+                                                    
  +--> | sd_netlink_open_fd | alloc netlink and setup, bind socket to it         
       +--------------------+                                                    
```

```
src/libsystemd/sd-netlink/sd-netlink.c                                                                        
+-------------------------+                                                                                    
| sd_netlink_attach_event | : setup two sources: "netlink-receive-message" & "netlink-timer", add to event     
+-|-----------------------+                                                                                    
  |                                                                                                            
  |--> ensure event exists                                                                                     
  |                                                                                                            
  |    +-----------------+                                                                                     
  |--> | sd_event_add_io | prepare 'source' and add to arg 'event, register the source's io                    
  |    +-----------------+ +-------------+                                                                     
  |                        | io_callback | (skip)                                                              
  |                        +-------------+                                                                     
  |    +------------------------------+                                                                        
  |--> | sd_event_source_set_priority |                                                                        
  |    +------------------------------+                                                                        
  |    +---------------------------------+                                                                     
  |--> | sd_event_source_set_description | "netlink-receive-message"                                           
  |    +---------------------------------+                                                                     
  |    +-----------------------------+                                                                         
  |--> | sd_event_source_set_prepare | update s->callback, insert to or remove from queue accordingly          
  |    +-----------------------------+ +------------------+                                                    
  |                                    | prepare_callback | (skip)                                             
  |                                    +------------------+                                                    
  |    +-------------------+                                                                                   
  |--> | sd_event_add_time | set up timer fields of event (fd/queues), prepare 'source' and add to those queues
  |    +-------------------+ +---------------+                                                                 
  |                          | time_callback | (skip)                                                          
  |                          +---------------+                                                                 
  |    +------------------------------+                                                                        
  |--> | sd_event_source_set_priority |                                                                        
  |    +------------------------------+                                                                        
  |    +---------------------------------+                                                                     
  +--> | sd_event_source_set_description | "netlink-timer"                                                     
       +---------------------------------+                                                                     
```

```
src/network/netdev/netdev.c                                                     
+--------------------+                                                           
| netdev_set_ifindex | : get iface idx and set to net_dev, net_dev->state = ready
+-|------------------+                                                           
  |    +-----------------------------+                                           
  |--> | sd_netlink_message_get_type | get type (newlink or dellink) from msg    
  |    +-----------------------------+                                           
  |                                                                              
  |--> if type != newlink, return error                                          
  |                                                                              
  |    +----------------------------------+                                      
  |--> | sd_rtnl_message_link_get_ifindex | get iface idx from msg               
  |    +----------------------------------+                                      
  |                                                                              
  |--> if net_dev's iface idx is already set, return                             
  |                                                                              
  |    +--------------------------------+                                        
  |--> | sd_netlink_message_read_string | get iface name from msg                
  |    +--------------------------------+                                        
  |                                                                              
  |--> expent net_dev's name == iface_name from msg                              
  |                                                                              
  |--> net_dev->ifindex = iface idx                                              
  |                                                                              
  |    +--------------------+                                                    
  +--> | netdev_enter_ready | net_dev->state = ready                             
       +--------------------+                                                    
```

```
src/network/networkd-link.c                                                            
+----------+                                                                            
| link_new | : get iface parameters from msg, prepare 'link' accordingly, add to manager
+-|--------+                                                                            
  |    +----------------------------------+                                             
  |--> | sd_rtnl_message_link_get_ifindex | get iface idx from msg                      
  |    +----------------------------------+                                             
  |    +-------------------------------+                                                
  |--> | sd_rtnl_message_link_get_type | get iface type from msg                        
  |    +-------------------------------+                                                
  |    +---------------------------------------+                                        
  |--> | sd_netlink_message_read_string_strdup | get iface name from msg                
  |    +---------------------------------------+                                        
  |                                                                                     
  |--> determine file names "/run/systemd/netif/links/%d"                               
  |                         "/run/systemd/netif/leases/%d"                              
  |                         "/run/systemd/netif/lldp/%d"                                
  |                                                                                     
  |--> alloc 'link' and setup (iface index, iface type, iface name, file names, ...)    
  |                                                                                     
  |--> add 'link' to links_by_index of manager                                          
  |                                                                                     
  +--> add 'link' to links_by_name of manager                                           
```

```
src/network/networkd-link.c                                     
+------------------+                                             
| link_update_name | : given msg, update link's iface name       
+|-----------------+                                             
 |    +--------------------------------+                         
 |--> | sd_netlink_message_read_string | read iface name from msg
 |    +--------------------------------+                         
 |                                                               
 |--> if link->ifname == name from msg, return                   
 |                                                               
 |    +---------------+                                          
 |--> | format_ifname | format a expected iface name             
 |    +---------------+                                          
 |                                                               
 |--> if expeted iface name != name from msg, return             
 |                                                               
 |--> remove 'link' from manager                                 
 |                                                               
 |--> update iface name in 'link'                                
 |                                                               
 |    +--------------------+                                     
 |--> | hashmap_ensure_put | add 'link' to manager               
 |    +--------------------+                                     
 |                                                               
 +--> update names of link's other fields if necessary           
```

```
src/network/networkd-link.c                                        
+--------------------+                                              
| link_update_driver | : ensure driver name is saved in link->driver
+-|------------------+                                              
  |                                                                 
  |--> if already read, return                                      
  |                                                                 
  |--> link->ethtool_driver_read = true                             
  |                                                                 
  |    +--------------------+                                       
  +--> | ethtool_get_driver | ioctl to get driver name              
       +--------------------+                                       
```

```
src/shared/ethtool-util.c                         
+--------------------+                             
| ethtool_get_driver | : ioctl to get driver name  
+-|------------------+                             
  |    +-----------------+                         
  |--> | ethtool_connect | ensure ethtool_fd exists
  |    +-----------------+                         
  |    +-------+                                   
  |--> | ioctl |  SIOCETHTOOL, to query driver info
  |    +-------+                                   
  |                                                
  +--> get driver name and return                  
```

```
src/network/networkd-link.c                                                        
+------------------------------+                                                    
| link_update_hardware_address | : get hw addr from msg, update to link if different
+-|----------------------------+                                                    
  |    +------------------------------+                                             
  |--> | netlink_message_read_hw_addr | read broadcast addr from msg                
  |    +------------------------------+                                             
  |    +------------------------------+                                             
  |--> | netlink_message_read_hw_addr | read hw addr from msg                       
  |    +------------------------------+                                             
  |                                                                                 
  |--> if link->hw_addr == addr from msg, return                                    
  |                                                                                 
  |--> if link->hw_addr exists (old), remove link from manager first                
  |                                                                                 
  |--> link->hw_addr = hw addr                                                      
  |                                                                                 
  |    +--------------------+                                                       
  |--> | hashmap_ensure_put | add link to manager                                   
  |    +--------------------+                                                       
  |                                                                                 
  +--> update mac in ipv4, dhcp4, dhcp6, ...                                        
```

```
src/network/networkd-link.c                                                        
+------------------------------+                                                    
| link_update_hardware_address | : get hw addr from msg, update to link if different
+-|----------------------------+                                                    
  |    +------------------------------+                                             
  |--> | netlink_message_read_hw_addr | read broadcast addr from msg                
  |    +------------------------------+                                             
  |    +------------------------------+                                             
  |--> | netlink_message_read_hw_addr | read hw addr from msg                       
  |    +------------------------------+                                             
  |                                                                                 
  |--> if link->hw_addr == addr from msg, return                                    
  |                                                                                 
  |--> if link->hw_addr exists (old), remove link from manager first                
  |                                                                                 
  |--> link->hw_addr = hw addr                                                      
  |                                                                                 
  |    +--------------------+                                                       
  |--> | hashmap_ensure_put | add link to manager                                   
  |    +--------------------+                                                       
  |                                                                                 
  +--> update mac in ipv4, dhcp4, dhcp6, ...                                        
```

```
src/network/networkd-link.c                                                                                         
+-----------------------+                                                                                            
| link_update_operstate | : determine states and update to link, emit dbus signal or prop change, label link as dirty
+-|---------------------+                                                                                            
  |                                                                                                                  
  |--> determine carrier state                                                                                       
  |                                                                                                                  
  |--> if carrier state >= CARRIER                                                                                   
  |    -                                                                                                             
  |    +--> for each link slave                                                                                      
  |         |                                                                                                        
  |         |    +-----------------------+                                                                           
  |         +--> | link_update_operstate | (recursive)                                                               
  |              +-----------------------+                                                                           
  |                                                                                                                  
  |--> determine address state                                                                                       
  |                                                                                                                  
  |--> determine oper state                                                                                          
  |                                                                                                                  
  |--> determine online state                                                                                        
  |                                                                                                                  
  |--> update these states in link                                                                                   
  |                                                                                                                  
  |--> if at least one state changes                                                                                 
  |    |                                                                                                             
  |    |    +------------------------+                                                                               
  |    |--> | link_send_changed_strv | prepare dbus info, emit signal of property change                             
  |    |    +------------------------+                                                                               
  |    |    +------------+                                                                                           
  |    +--> | link_dirty | label link as dirty, add to dirty list of manager                                         
  |         +------------+                                                                                           
  |                                                                                                                  
  +--> if 'update master' is specified                                                                               
       |                                                                                                             
       |    +-----------------------+                                                                                
       +--> | link_update_operstate | (recursive)                                                                    
            +-----------------------+                                                                                
```

```
src/network/networkd-link-bus.c                                              
+------------------------+                                                    
| link_send_changed_strv | : prepare dbus info, emit signal of property change
+-|----------------------+                                                    
  |    +---------------+                                                      
  |--> | link_bus_path | encode dbus obj path link                            
  |    +---------------+                                                      
  |    +-------------------------------------+                                
  +--> | sd_bus_emit_properties_changed_strv |                                
       +-------------------------------------+                                
       manager service                                                        
       obj path of link                                                       
       iface: "org.freedesktop.network1.Link"                                 
       properties from arg                                                    
```

```
src/network/networkd-link.c                                                                            
+------------------+                                                                                    
| link_get_network | : given link, find matched network from manager                                    
+-|----------------+                                                                                    
  |                                                                                                     
  +--> for each network in manager                                                                      
       |                                                                                                
       |    +------------------+                                                                        
       |--> | net_match_config | given attributes saved in link, check if they match the current network
       |    +------------------+                                                                        
       |                                                                                                
       +--> if not match, continue                                                                      
```

```
src/network/networkd-link.c                                                
+-------------------+                                                       
| link_stop_engines | : stop all kinds of network engines                   
+-|-----------------+                                                       
  |                                                                         
  |--> determine if we keep dhcp                                            
  |                                                                         
  |--> if not                                                               
  |    |                                                                    
  |    |    +---------------------+                                         
  |    +--> | sd_dhcp_client_stop | reset client, set client state = stopped
  |         +---------------------+                                         
  |    +---------------------+                                              
  |--> | sd_dhcp_server_stop | (skip)                                       
  |    +---------------------+                                              
  |    +-----------------+                                                  
  |--> | sd_lldp_rx_stop | (skip)                                           
  |    +-----------------+                                                  
  |    +-----------------+                                                  
  |--> | sd_lldp_tx_stop | (skip)                                           
  |    +-----------------+                                                  
  |    +----------------+                                                   
  |--> | sd_ipv4ll_stop | (skip)                                            
  |    +----------------+                                                   
  |    +--------------+                                                     
  |--> | ipv4acd_stop | (skip)                                              
  |    +--------------+                                                     
  |    +----------------------+                                             
  |--> | sd_dhcp6_client_stop | (skip)                                      
  |    +----------------------+                                             
  |    +----------------+                                                   
  |--> | dhcp_pd_remove | (skip)                                            
  |    +----------------+                                                   
  |    +------------+                                                       
  |--> | ndisc_stop | reset ndisc                                           
  |    +------------+                                                       
  |    +-------------+                                                      
  |--> | ndisc_flush | free ndisc related fields                            
  |    +-------------+                                                      
  |    +--------------+                                                     
  +--> | sd_radv_stop | (skip)                                              
       +--------------+                                                     
```

```
src/network/networkd-link.c                                                                           
+--------------------+                                                                                 
| link_drop_requests | : for each request in manager, if it belongs to arg link, remove it from manager
+-|------------------+                                                                                 
  |                                                                                                    
  +--> for each request in manager                                                                     
       -                                                                                               
       +--> if request belongs to arg link                                                             
            |                                                                                          
            |    +----------------+                                                                    
            +--> | request_detach | remove request from manager and unref                              
                 +----------------+                                                                    
```

```
src/network/networkd-queue.c                             
+----------------+                                        
| request_detach | : remove request from manager and unref
+-|--------------+                                        
  |    +--------------------+                             
  |--> | ordered_set_remove | remove request from manager 
  |    +--------------------+                             
  |    +---------------+                                  
  +--> | request_unref |                                  
       +---------------+                                  
```

```
src/network/networkd-link.c                                                                                          
+---------------------------+                                                                                         
| manager_rtnl_process_link | : given msg, get link/net_dev from manager, perform 'new link' or 'del link' accordingly
+-|-------------------------+                                                                                         
  |    +-----------------------------+                                                                                
  |--> | sd_netlink_message_get_type | get type (newlink or dellink) from msg                                         
  |    +-----------------------------+                                                                                
  |    +----------------------------------+                                                                           
  |--> | sd_rtnl_message_link_get_ifindex | get iface idx from msg                                                    
  |    +----------------------------------+                                                                           
  |    +--------------------------------+                                                                             
  |--> | sd_netlink_message_read_string | get iface name from msg                                                     
  |    +--------------------------------+                                                                             
  |    +-------------------+                                                                                          
  |--> | link_get_by_index | given iface idx, get link from manager                                                   
  |    +-------------------+                                                                                          
  |    +------------+                                                                                                 
  |--> | netdev_get | given iface name, get net_dev from manager                                                      
  |    +------------+                                                                                                 
  |                                                                                                                   
  +--> switch type                                                                                                    
       case newlink                                                                                                   
       |--> if net_dev exists                                                                                         
       |    -    +--------------------+                                                                               
       |    +--> | netdev_set_ifindex | get iface idx and set to net_dev, net_dev->state = ready                      
       |         +--------------------+                                                                               
       |--> if no link found                                                                                          
       |    |    +----------+                                                                                         
       |    |--> | link_new | get iface parameters from msg, prepare 'link' accordingly, add to manager               
       |    |    +----------+                                                                                         
       |    |    +-------------+                                                                                      
       |    |--> | link_update | given msg, update link                                                               
       |    |    +-------------+                                                                                      
       |    |    +------------------------+                                                                           
       |    +--> | link_check_initialized | chec if initialized, if true, configure link                              
       |         +------------------------+                                                                           
       +--> else                                                                                                      
            |    +-------------+                                                                                      
            |--> | link_update | given msg, update link                                                               
            |    +-------------+                                                                                      
            |    +-----------------------+                                                                            
            +--> | link_reconfigure_impl | given iface name, reconfigure link                                         
                 +-----------------------+                                                                            
       case dellink                                                                                                   
            |    +-----------+                                                                                        
            |--> | link_drop |                                                                                        
            |    +-----------+                                                                                        
            |    +-------------+                                                                                      
            +--> | netdev_drop |                                                                                      
                 +-------------+                                                                                      
```

```
src/network/networkd-link.c                                                               
+-------------+                                                                            
| link_update | : given msg, update link                                                   
+-|-----------+                                                                            
  |    +------------------+                                                                
  |--> | link_update_name | given msg, update link's iface name                            
  |    +------------------+                                                                
  |    +-------------------------------+                                                   
  |--> | link_update_alternative_names | update link's alternative name                    
  |    +-------------------------------+                                                   
  |    +-----------------+                                                                 
  |--> | link_update_mtu | get mtu from msg, update link's mtu if different                
  |    +-----------------+                                                                 
  |    +--------------------+                                                              
  |--> | link_update_driver | ensure driver name is saved in link->driver                  
  |    +--------------------+                                                              
  |    +----------------------------------------+                                          
  |--> | link_update_permanent_hardware_address | get permanent addr from msg, save in link
  |    +----------------------------------------+                                          
  |    +------------------------------+                                                    
  |--> | link_update_hardware_address | get hw addr from msg, update to link if different  
  |    +------------------------------+                                                    
  |    +--------------------+                                                              
  |--> | link_update_master | given msg, update link->master_ifindex                       
  |    +--------------------+                                                              
  |    +---------------------------------+                                                 
  |--> | link_update_ipv6ll_addrgen_mode | (skip)                                          
  |    +---------------------------------+                                                 
  |    +-------------------+                                                               
  +--> | link_update_flags | given msg, update link's flags and operstate                  
       +-------------------+                                                               
```

```
src/network/networkd-link.c                                                                                              
+-------------------+                                                                                                     
| link_update_flags | : given msg, update link's flags and operstate                                                      
+-|-----------------+                                                                                                     
  |    +--------------------------------+                                                                                 
  |--> | sd_rtnl_message_link_get_flags | get flags from msg                                                              
  |    +--------------------------------+                                                                                 
  |    +----------------------------+                                                                                     
  |--> | sd_netlink_message_read_u8 | get oper state from msg                                                             
  |    +----------------------------+                                                                                     
  |                                                                                                                       
  |--> if they are same to the ones in link, return                                                                       
  |                                                                                                                       
  |--> if link->flags != flags from msg                                                                                   
  |    -                                                                                                                  
  |    +--> log debug info                                                                                                
  |                                                                                                                       
  |--> link->flags = flags                                                                                                
  |                                                                                                                       
  |--> link->kernel_operstate = operstate                                                                                 
  |                                                                                                                       
  |    +-----------------------+                                                                                          
  |--> | link_update_operstate | determine states and update to link, emit dbus signal or prop change, label link as dirty
  |    +-----------------------+                                                                                          
  |                                                                                                                       
  |--> if up                                                                                                              
  |    |                                                                                                                  
  |    |    +---------------------+                                                                                       
  |    +--> | link_admin_state_up | (skip)                                                                                
  |         +---------------------+                                                                                       
  |                                                                                                                       
  |--> else (down)                                                                                                        
  |    |                                                                                                                  
  |    |    +-----------------------+                                                                                     
  |    +--> | link_admin_state_down | (skip)                                                                              
  |         +-----------------------+                                                                                     
  |                                                                                                                       
  |--> if has carrier                                                                                                     
  |    |                                                                                                                  
  |    |    +---------------------+                                                                                       
  |    +--> | link_carrier_gained | reconfigure link, up or down other links on bound-by list, start engines              
  |         +---------------------+                                                                                       
  |                                                                                                                       
  +--> else (carrier lost)                                                                                                
       |                                                                                                                  
       |    +-------------------+                                                                                         
       +--> | link_carrier_lost | down other links on bound-by list, stop engines, remove configs                         
            +-------------------+                                                                                         
```

```
src/network/networkd-link.c                                                                      
+---------------------+                                                                           
| link_carrier_gained | : reconfigure link, up or down other links on bound-by list, start engines
+-|-------------------+                                                                           
  |    +----------------------+                                                                   
  |--> | event_source_disable | disable source 'carrier_lost_timer'                               
  |    +----------------------+                                                                   
  |    +-----------------------+                                                                  
  |--> | link_reconfigure_impl | given iface name, reconfigure link                               
  |    +-----------------------+                                                                  
  |    +---------------------------+                                                              
  |--> | link_handle_bound_by_list | for each link in bound-by list, send req (up or down)        
  |    +---------------------------+                                                              
  |                                                                                               
  +--> if link state is 'configuring' or 'configured'                                             
       |                                                                                          
       |    +---------------------------+                                                         
       +--> | link_acquire_dynamic_conf | start engines                                           
       |    +---------------------------+                                                         
       |    +-----------------------------+                                                       
       +--> | link_request_static_configs | send requests to get static configs                   
            +-----------------------------+                                                       
```

```
src/network/networkd-link.c                                                                                    
+-----------------------+                                                                                       
| link_reconfigure_impl | : given iface name, reconfigure link                                                  
+-|---------------------+                                                                                       
  |    +------------+                                                                                           
  |--> | netdev_get | given iface name, get net_dev                                                             
  |    +------------+                                                                                           
  |    +------------------+                                                                                     
  |--> | link_get_network | given link, find matched network from manager                                       
  |    +------------------+                                                                                     
  |                                                                                                             
  |--> if link->network == network we just found, return                                                        
  |                                                                                                             
  |    +-------------------+                                                                                    
  |--> | link_stop_engines | stop all kinds of network engines                                                  
  |    +-------------------+                                                                                    
  |    +--------------------+                                                                                   
  |--> | link_drop_requests | for each request in manager, if it belongs to arg link, remove it from manager    
  |    +--------------------+                                                                                   
  |                                                                                                             
  |--> if networrk && !keep_config                                                                              
  |    |                                                                                                        
  |    |    +------------------------+                                                                          
  |    +--> | link_foreignize_config | for each config of link, set their source = NETWORK_CONFIG_SOURCE_FOREIGN
  |         +------------------------+                                                                          
  |                                                                                                             
  |--> else                                                                                                     
  |    |                                                                                                        
  |    |    +--------------------------+                                                                        
  |    +--> | link_drop_managed_config | for each config of link, send req to remove it                         
  |         +--------------------------+                                                                        
  |    +-------------------------+                                                                              
  |--> | link_free_bound_to_list | free bound_by list of link                                                   
  |    +-------------------------+                                                                              
  |    +-------------------+                                                                                    
  |--> | link_free_engines | unref engines in link                                                              
  |    +-------------------+                                                                                    
  |                                                                                                             
  |--> unref network & netdev of link                                                                           
  |                                                                                                             
  |--> link->netdev = netdev found by iface name                                                                
  |                                                                                                             
  |--> link->network = network found by link                                                                    
  |                                                                                                             
  |--> link state = initialized                                                                                 
  |                                                                                                             
  |--> link->activated = false                                                                                  
  |                                                                                                             
  |    +----------------+                                                                                       
  +--> | link_configure | send requests to set link configs                                                     
       +----------------+                                                                                       
```

```
src/network/networkd-link.c                                                                                          
+------------------------+                                                                                            
| link_foreignize_config | : for each config of link, set their source = NETWORK_CONFIG_SOURCE_FOREIGN                
+-|----------------------+                                                                                            
  |    +------------------------+                                                                                     
  |--> | link_foreignize_routes | for each link-related route, ->source = NETWORK_CONFIG_SOURCE_FOREIGN               
  |    +------------------------+                                                                                     
  |    +--------------------------+                                                                                   
  |--> | link_foreignize_nexthops | for each link-related nexthop, ->source = NETWORK_CONFIG_SOURCE_FOREIGN           
  |    +--------------------------+                                                                                   
  |    +---------------------------+                                                                                  
  |--> | link_foreignize_addresses | for each address in link, ->source = NETWORK_CONFIG_SOURCE_FOREIGN               
  |    +---------------------------+                                                                                  
  |    +---------------------------+                                                                                  
  |--> | link_foreignize_neighbors | for each neighbor in link, ->source = NETWORK_CONFIG_SOURCE_FOREIGN              
  |    +---------------------------+                                                                                  
  |    +--------------------------------------+                                                                       
  +--> | link_foreignize_routing_policy_rules | for each rule of target link, ->source = NETWORK_CONFIG_SOURCE_FOREIGN
       +--------------------------------------+                                                                       
```

```
src/network/networkd-route.c                                                                     
+------------------------+                                                                        
| link_foreignize_routes | : for each link-related route, ->source = NETWORK_CONFIG_SOURCE_FOREIGN
+-|----------------------+                                                                        
  |                                                                                               
  |--> for each route in link                                                                     
  |    -                                                                                          
  |    +--> route->source = NETWORK_CONFIG_SOURCE_FOREIGN                                         
  |                                                                                               
  |    +---------------------+                                                                    
  |--> | manager_mark_routes | mark routes of target link                                         
  |    +---------------------+                                                                    
  |                                                                                               
  +--> for each route in manager                                                                  
       |                                                                                          
       |--> if route isn't marked, continue                                                       
       |                                                                                          
       +--> route->source = NETWORK_CONFIG_SOURCE_FOREIGN                                         
```

```
src/network/networkd-route.c                                                           
+---------------------+                                                                 
| manager_mark_routes | : mark routes of target link                                    
+-|-------------------+ (firstly mark all routes, then unmark routes of any other links)
  |                                                                                     
  |--> for each route in manager                                                        
  |    |                                                                                
  |    |    +------------+                                                              
  |    +--> | route_mark | mark route                                                   
  |         +------------+ (we can search function definition by '_mark')               
  |                                                                                     
  +--> for each link in manager                                                         
       |                                                                                
       |--> if link == arg link, continue                                               
       |                                                                                
       |--> if !link->network, continue                                                 
       |                                                                                
       |--> if link isn't configured, continue                                          
       |                                                                                
       +--> for each route in link->network                                             
            |                                                                           
            |    +---------------+                                                      
            |--> | route_convert | given route, prepare converted_route                 
            |    +---------------+                                                      
            |                                                                           
            +--> for each route in converted_route                                      
                 |                                                                      
                 |    +--------------+                                                  
                 +--> | route_unmark |                                                  
                      +--------------+                                                  
```

```
src/network/networkd-route.c                                                                                 
+---------------+                                                                                             
| route_convert | : given route, prepare converted_route                                                      
+-|-------------+                                                                                             
  |                                                                                                           
  |--> if route needs no convert, return                                                                      
  |                                                                                                           
  |--> if route has nexthop id                                                                                
  |    |                                                                                                      
  |    |    +---------------------------+                                                                     
  |    |--> | manager_get_nexthop_by_id | given id, get nexthop from manager                                  
  |    |    +---------------------------+                                                                     
  |    |                                                                                                      
  |    +--> if group in nexthop is empty                                                                      
  |    |    |                                                                                                 
  |    |    |    +----------------------+                                                                     
  |    |    |--> | converted_routes_new | alloc converted_route                                               
  |    |    |    +----------------------+                                                                     
  |    |    |    +-----------+                                                                                
  |    |    |--> | route_dup | duplicate 'src' to get 'dst', save in converted_route                          
  |    |    |    +-----------+                                                                                
  |    |    |    +---------------------+                                                                      
  |    |    |--> | route_apply_nexthop | given nexthop, setup converted_route                                 
  |    |    |    +---------------------+                                                                      
  |    |    +--> return                                                                                       
  |    |                                                                                                      
  |    |    +----------------------+                                                                          
  |    |--> | converted_routes_new | given nexthop group size, alloc converted_route                          
  |    |    +----------------------+                                                                          
  |    |                                                                                                      
  |    |--> for each nexthop in group                                                                         
  |    |    |                                                                                                 
  |    |    |    +---------------------------+                                                                
  |    |    |--> | manager_get_nexthop_by_id | given id, get nexthop from manager                             
  |    |    |    +---------------------------+                                                                
  |    |    |    +-----------+                                                                                
  |    |    |--> | route_dup | duplicate 'src' to get 'dst', save in converted_route                          
  |    |    |    +-----------+                                                                                
  |    |    |    +---------------------+                                                                      
  |    |    +--> | route_apply_nexthop | given nexthop, setup converted_route                                 
  |    |         +---------------------+                                                                      
  |    +--> return                                                                                            
  |                                                                                                           
  |    +----------------------+                                                                               
  |--> | converted_routes_new | alloc converted_route                                                         
  |    +----------------------+                                                                               
  |                                                                                                           
  +--> for path in route                                                                                      
       -                                                                                                      
       +--> setup converted_route                                                                             
```

```
src/network/networkd-routing-policy-rule.c                                                                      
+--------------------------------------+                                                                         
| link_foreignize_routing_policy_rules | : for each rule of target link, ->source = NETWORK_CONFIG_SOURCE_FOREIGN
+-|------------------------------------+                                                                         
  |    +-----------------------------------+                                                                     
  |--> | manager_mark_routing_policy_rules | mark routing_policy_rule of target link                             
  |    +-----------------------------------+                                                                     
  |                                                                                                              
  +--> for each rule in manager                                                                                  
       |                                                                                                         
       |--> if it's not marked, continue                                                                         
       |                                                                                                         
       +--> rule->source = NETWORK_CONFIG_SOURCE_FOREIGN                                                         
```

```
src/network/networkd-routing-policy-rule.c                                    
+-----------------------------------+                                          
| manager_mark_routing_policy_rules | : mark routing_policy_rule of target link
+-|---------------------------------+                                          
  |                                                                            
  |--> for each rule in manager                                                
  |    |                                                                       
  |    |    +--------------------------+                                       
  |    +--> | routing_policy_rule_mark | mark routing_policy_rule              
  |         +--------------------------+                                       
  |                                                                            
  +--> for each link in manager                                                
       |                                                                       
       |--> if link == arg link, continue                                      
       |                                                                       
       +--> for each rule in link network                                      
           |                                                                   
           |    +----------------------------+                                 
           +--> | routing_policy_rule_unmark | unmark routing_policy_rule      
                +----------------------------+                                 
```

```
src/network/networkd-link.c                                                                         
+--------------------------+                                                                         
| link_drop_managed_config | : for each config of link, send req to remove it                        
+-|------------------------+                                                                         
  |    +--------------------------+                                                                  
  |--> | link_drop_managed_routes | for each link-related route, send req to remove it               
  |    +--------------------------+                                                                  
  |    +----------------------------+                                                                
  |--> | link_drop_managed_nexthops | for each link-related nexthop, send req to remove it           
  |    +----------------------------+                                                                
  |    +-----------------------------+                                                               
  |--> | link_drop_managed_addresses | for each address in link, send req to remove it               
  |    +-----------------------------+                                                               
  |    +-----------------------------+                                                               
  |--> | link_drop_managed_neighbors | for each neighbor in link, send req to remove it              
  |    +-----------------------------+                                                               
  |    +----------------------------------------+                                                    
  +--> | link_drop_managed_routing_policy_rules | for each rule of target link, send req to remove it
       +----------------------------------------+                                                    
```

```
src/network/networkd-route.c                                                                    
+--------------------------+                                                                     
| link_drop_managed_routes | : for each link-related route, send req to remove it                
+-|------------------------+                                                                     
  |                                                                                              
  |--> for each route in link                                                                    
  |    |                                                                                         
  |    |    +--------------+                                                                     
  |    +--> | route_remove | alloc req (del route) and setup, send req out                       
  |         +--------------+                                                                     
  |    +---------------------+                                                                   
  |--> | manager_mark_routes | mark routes of target link                                        
  |    +---------------------+                                                                   
  |    +----------------------------+                                                            
  +--> | manager_drop_marked_routes | : for each marked route in manage, send req to remove route
       +----------------------------+                                                            
```

```
src/network/networkd-route.c                                                             
+----------------------------+                                                            
| manager_drop_marked_routes | : for each marked route in manage, send req to remove route
+-|--------------------------+                                                            
  |                                                                                       
  +--> for each route in manager                                                          
       |                                                                                  
       |--> if route isn't marked, continue                                               
       |                                                                                  
       |    +--------------+                                                              
       +--> | route_remove | alloc req (del route) and setup, send req out                
            +--------------+                                                              
```

```
src/network/networkd-route.c                                   
+--------------+                                                
| route_remove | : alloc req (del route) and setup, send req out
+-|------------+                                                
  |                                                             
  |--> get manager from route or link                           
  |                                                             
  |    +---------------------------+                            
  |--> | sd_rtnl_message_new_route | prepare req (del route)    
  |    +---------------------------+                            
  |                                                             
  |--> determine type                                           
  |                                                             
  |    +--------------------------------+                       
  |--> | sd_rtnl_message_route_set_type | set type in req       
  |    +--------------------------------+                       
  |    +---------------------------+                            
  |--> | route_set_netlink_message | given route, setup msg     
  |    +---------------------------+                            
  |    +--------------------+                                   
  +--> | netlink_call_async | send msg out                      
       +--------------------+                                   
```

```
src/network/networkd-route.c                                    
+---------------------------+                                    
| route_set_netlink_message | : given route, setup msg           
+-|-------------------------+                                    
  |                                                              
  |--> if in-addr is set                                         
  |    -                                                         
  |    +--> append gateway or via (?) info to msg                
  |                                                              
  |--> if route has dst_prefix                                   
  |    -                                                         
  |    +--> append dst info to msg                               
  |                                                              
  |--> if route has src_prefix                                   
  |    -                                                         
  |    +--> append src info to msg                               
  |                                                              
  |    +---------------------------------+                       
  |--> | sd_rtnl_message_route_set_scope | set scope             
  |    +---------------------------------+                       
  |    +---------------------------------+                       
  |--> | sd_rtnl_message_route_set_flags | set flags             
  |    +---------------------------------+                       
  |    +---------------------------------+                       
  |--> | sd_rtnl_message_route_set_table |                       
  |    +---------------------------------+                       
  |    +-------------------------------+                         
  |--> | sd_netlink_message_append_u32 | append nexthop id to msg
  |    +-------------------------------+                         
  |    +------------------------------+                          
  |--> | sd_netlink_message_append_u8 | append pref to msg       
  |    +------------------------------+                          
  |    +-------------------------------+                         
  +--> | sd_netlink_message_append_u32 | append priority to msg  
       +-------------------------------+                         
```

```
src/network/networkd-link.c                                                                  
+-------------------------+                                                                   
| link_free_bound_to_list | : free bound_by list of link                                      
+-|-----------------------+                                                                   
  |                                                                                           
  |--> for each bound_by in link                                                              
  |    |                                                                                      
  |    |--> remove it from link                                                               
  |    |                                                                                      
  |    |    +------------+                                                                    
  |    |--> | link_dirty | label link as dirty, add to dirty list of manager                  
  |    |    +------------+                                                                    
  |    |    +---------------------------+                                                     
  |    +--> | link_handle_bound_to_list | prepare req (up or down) and add to queue of manager
  |         +---------------------------+                                                     
  |                                                                                           
  +--> if we processed any bound_by                                                           
       |                                                                                      
       |    +------------+                                                                    
       +--> | link_dirty | label link as dirty, add to dirty list of manager                  
            +------------+                                                                    
```

```
src/network/networkd-link.c                                                        
+---------------------------+                                                       
| link_handle_bound_to_list | : prepare req (up or down) and add to queue of manager
+-|-------------------------+                                                       
  |                                                                                 
  |--> determine if 'link' should be up or down                                     
  |                                                                                 
  |    +----------------------------------+                                         
  +--> | link_request_to_bring_up_or_down | prepare req and add to queue of manager 
       +----------------------------------+                                         
```

```
src/network/networkd-setlink.c                                               
+----------------------------------+                                          
| link_request_to_bring_up_or_down | : prepare req and add to queue of manager
+-|--------------------------------+                                          
  |    +-------------------------+                                            
  +--> | link_queue_request_full | prepare req and add to queue of manager    
       +-------------------------+                                            
```

```
src/network/networkd-queue.c                                        
+-------------------------+                                          
| link_queue_request_full | : prepare req and add to queue of manager
+-------------------------+                                          
| request_new | : prepare req and add to queue of manager            
+-|-----------+                                                      
  |                                                                  
  |--> alloc req and setup                                           
  |                                                                  
  |--> check if there's already one in manager                       
  |                                                                  
  |--> if found, return                                              
  |                                                                  
  |    +------------------------+                                    
  +--> | ordered_set_ensure_put | add req to hash table of manager   
       +------------------------+                                    
```

```
src/network/networkd-link.c                                                                            
+----------------+                                                                                      
| link_configure | : send requests to set link configs                                                  
+-|--------------+                                                                                      
  |                                                                                                     
  |--> link state = configuring                                                                         
  |                                                                                                     
  |    +------------------------+                                                                       
  |--> | link_new_bound_to_list | for each link-related carrier in manager, add to bound-to list of link
  |    +------------------------+                                                                       
  |    +------------------------------+                                                                 
  |--> | link_request_traffic_control | prepare req to ask for qdisc/tclass                             
  |    +------------------------------+                                                                 
  |    +-------------------------+                                                                      
  |--> | link_request_sr_iov_vfs | (hardware virtualization related, skip)                              
  |    +-------------------------+                                                                      
  |    +-----------------+                                                                              
  |--> | link_set_sysctl | (sysctl related, skip)                                                       
  |    +-----------------+                                                                              
  |    +-------------------------+                                                                      
  |--> | link_request_to_set_mac | send req to set link mac                                             
  |    +-------------------------+                                                                      
  |    +---------------------------+                                                                    
  |--> | link_request_to_set_flags | send req to set link flags                                         
  |    +---------------------------+                                                                    
  |    +--------------------+                                                                           
  |--> | link_configure_mtu | send req to set link mtu                                                  
  |    +--------------------+                                                                           
  |    +----------------------------------+                                                             
  |--> | link_request_to_set_addrgen_mode | send req to set link addresss generation mode               
  |    +----------------------------------+                                                             
  |    +----------------------------+                                                                   
  |--> | link_request_to_set_master | send req to set link master                                       
  |    +----------------------------+                                                                   
  |    +------------------------------+                                                                 
  |--> | link_request_stacked_netdevs | send req to config stacked netdevs                              
  |    +------------------------------+                                                                 
  |    +--------------------------+                                                                     
  |--> | link_request_to_set_bond | send req to set link bond                                           
  |    +--------------------------+                                                                     
  |    +----------------------------+                                                                   
  |--> | link_request_to_set_bridge | send req to set bridge                                            
  |    +----------------------------+                                                                   
  |    +---------------------------------+                                                              
  |--> | link_request_to_set_bridge_vlan | send req to set bridge vlan                                  
  |    +---------------------------------+                                                              
  |    +--------------------------+                                                                     
  |--> | link_request_to_activate | send req to set activate link                                       
  |    +--------------------------+                                                                     
  |                                                                                                     
  +--> . . .                                                                                            
```

```
src/network/networkd-link.c                                                                       
+------------------------+                                                                         
| link_new_bound_to_list | : for each link-related carrier in manager, add to bound-to list of link
+-|----------------------+                                                                         
  |                                                                                                
  |--> get manager from link                                                                       
  |                                                                                                
  |--> for each carrier in manager                                                                 
  |    -                                                                                           
  |    +--> if network->bind_carrier == current carrier                                            
  |         |                                                                                      
  |         |    +------------------+                                                              
  |         +--> | link_put_carrier | ensure carrier is in link (bound-to list)                    
  |              +------------------+                                                              
  |                                                                                                
  +--> for each carrier in link                                                                    
       |                                                                                           
       |    +------------------+                                                                   
       +--> | link_put_carrier | ensure link is in carrier (bound-by list)                         
            +------------------+                                                                   
```

```
src/network/tc/tc.c                                                               
+------------------------------+                                                   
| link_request_traffic_control | : prepare req to ask for qdisc/tclass             
+-|----------------------------+                                                   
  |                                                                                
  |--> for each qdisc in link->network                                             
  |    |                                                                           
  |    |    +--------------------+                                                 
  |    +--> | link_request_qdisc | prepare req to ask for qdisc (traffic control)  
  |         +--------------------+                                                 
  |                                                                                
  +--> for each tclass in link->network                                            
       |                                                                           
       |    +---------------------+                                                
       +--> | link_request_tclass | prepare req to ask for tclass (traffic control)
            +---------------------+                                                
```

```
src/network/tc/qdisc.c                                                   
+--------------------+                                                    
| link_request_qdisc | : prepare req to ask for qdisc (traffic control)   
+-|------------------+                                                    
  |    +-----------+                                                      
  |--> | qdisc_get | get qdisc from link                                  
  |    +-----------+                                                      
  |    +-------------------------+                                        
  +--> | link_queue_request_safe | prepare req and add to queue of manager
       +-------------------------+                                        
```

```
src/network/networkd-setlink.c                                         
+-------------------------+                                             
| link_request_to_set_mac | : send req to set link mac                  
+-|-----------------------+                                             
  |    +-----------------------------+                                  
  |--> | net_verify_hardware_address |                                  
  |    +-----------------------------+                                  
  |    +-----------------------+                                        
  +--> | link_request_set_link | prepare req and add to queue of manager
       +-----------------------+                                        
```

```
src/network/networkd-link.c                                                                  
+---------------------------+                                                                 
| link_handle_bound_by_list | : for each link in bound-by list, send req (up or down)         
+-|-------------------------+                                                                 
  |                                                                                           
  +--> for each link in bound-by list of arg link                                             
       |                                                                                      
       |    +---------------------------+                                                     
       +--> | link_handle_bound_to_list | prepare req (up or down) and add to queue of manager
            +---------------------------+                                                     
```

```
src/network/networkd-link.c                                                              
+-------------------+                                                                     
| link_carrier_lost | : down other links on bound-by list, stop engines, remove configs   
+-|-----------------+                                                                     
  |    +---------------------------+                                                      
  |--> | link_handle_bound_by_list | for each link in bound-by list, send req (up or down)
  |    +---------------------------+                                                      
  |    +------------------------+                                                         
  +--> | link_carrier_lost_impl | stop engines and remove configs                         
       +------------------------+                                                         
```

```
src/network/networkd-link.c                                                      
+------------------------+                                                        
| link_carrier_lost_impl | : stop engines and remove configs                      
+-|----------------------+                                                        
  |    +-------------------+                                                      
  |--> | link_stop_engines | stop all kinds of network engines                    
  |    +-------------------+                                                      
  |    +--------------------------+                                               
  +--> | link_drop_managed_config | for each config of link, send req to remove it
       +--------------------------+                                               
```

```
src/network/networkd-link.c                                                     
+------------------------+                                                       
| link_check_initialized | : chec if initialized, if true, configure link        
+-|----------------------+                                                       
  |    +----------------------------+                                            
  |--> | sd_device_new_from_ifindex | given iface idx, alloc sd_device           
  |    +----------------------------+                                            
  |    +-----------------------------+                                           
  |--> | sd_device_get_is_initialized| given sd_device, check if it's initialized
  |    +-----------------------------+                                           
  |                                                                              
  |--> if link is dhcp client                                                    
  |    |                                                                         
  |    |    +------------------------------+                                     
  |    +--> | sd_dhcp_client_attach_device |                                     
  |         +------------------------------+                                     
  |                                                                              
  |--> if link is dhcp6 client                                                   
  |    |                                                                         
  |    |    +-------------------------------+                                    
  |    +--> | sd_dhcp6_client_attach_device |                                    
  |         +-------------------------------+                                    
  |    +---------------------------+                                             
  |--> | link_set_sr_iov_ifindices | (hardware virtualization related, skip)     
  |    +---------------------------+                                             
  |                                                                              
  |--> if link state isn't pending                                               
  |    |                                                                         
  |    |    +------------------+                                                 
  |    |--> | link_reconfigure | send req to get link, reconfigure it            
  |    |    +------------------+                                                 
  |    +--> return                                                               
  |                                                                              
  |    +-------------------+                                                     
  +--> | link_call_getlink | send req to get link, reconfigure it                
       +-------------------+                                                     
```

```
src/network/networkd-link.c                                                               
+------------------+                                                                       
| link_reconfigure | : send req to get link, reconfigure it                                
+-|----------------+                                                                       
  |                                                                                        
  |--> if link is pending or initialized, return                                           
  |                                                                                        
  |    +-------------------+                                                               
  +--> | link_call_getlink | send req to get link                                          
       +-------------------+ +-------------------------+                                   
                             | link_reconfigure_handler| given iface name, reconfigure link
                             +-------------------------+                                   
```

```
src/network/networkd-manager.c                                                                                                 
+---------------+                                                                                                               
| manager_setup | : register callback forr post source, connect rtnl, setup dbus, prepare 'resolve', alloc address pools        
+-|-------------+                                                                                                               
  |    +------------------+                                                                                                     
  |--> | sd_event_default |                                                                                                     
  |    +------------------+                                                                                                     
  |    +-----------------------+                                                                                                
  |--> | sd_event_set_watchdog | given arg b, arm or unarm watchdog of event                                                    
  |    +-----------------------+                                                                                                
  |                                                                                                                             
  |--> install signal handlers                                                                                                  
  |                                                                                                                             
  |    +-------------------+                                                                                                    
  |--> | sd_event_add_post | prepare 'source' for post and add to event's hash table                                            
  |    +-------------------+ +-----------------------+                                                                          
  |                          | manager_dirty_handler | save states of manager to file, save state of dirty links to file        
  |                          +-----------------------+                                                                          
  |    +-------------------+                                                                                                    
  |--> | sd_event_add_post | prepare 'source' for post and add to event's hash table                                            
  |    +-------------------+ +--------------------------+                                                                       
  |                          | manager_process_requests | for each request in manager: process and detatch from manager         
  |                          +--------------------------+                                                                       
  |                                                                                                                             
  |    +--------------------+                                                                                                   
  |--> | manager_listen_fds | look for rtnl_fd                                                                                  
  |    +--------------------+                                                                                                   
  |    +----------------------+                                                                                                 
  |--> | manager_connect_rtnl | add match fules, attach filter table                                                            
  |    +----------------------+                                                                                                 
  |    +---------------------+                                                                                                  
  +--> | manager_connect_bus | register vtables, request name "org.freedesktop.network1", register callbacks for match rules    
  |    +---------------------+                                                                                                  
  |    +----------------------+                                                                                                 
  |--> | manager_connect_udev | prepare monitor, add net/ieee80211/rfkill to it, regisster call (setup sd_device) to 'io' source
  |    +----------------------+                                                                                                 
  |    +--------------------+                                                                                                   
  |--> | sd_resolve_default | prepare default 'resolve'                                                                         
  |    +--------------------+                                                                                                   
  |    +-------------------------+                                                                                              
  |--> | sd_resolve_attach_event | attach event to resolve, register callback of 'io' source                                    
  |    +-------------------------+                                                                                              
  |    +----------------------------+                                                                                           
  |--> | address_pool_setup_default | alloc well-known address pools and add to manager                                         
  |    +----------------------------+                                                                                           
  |                                                                                                                             
  +--> state file = "/run/systemd/netif/state"                                                                                  
```

```
src/network/networkd-manager.c                                                                                                                               
+----------------------+                                                                                                                                      
| manager_connect_rtnl | : add match fules, attach filter table                                                                                               
+-|--------------------+                                                                                                                                      
  |                                                                                                                                                           
  |--> if fd isn't valid                                                                                                                                      
  |    -    +-----------------+                                                                                                                               
  |    +--> | sd_netlink_open | open socket, alloc netlink and setup, bind socket to it                                                                       
  |         +-----------------+                                                                                                                               
  |--> else                                                                                                                                                   
  |    -    +--------------------+                                                                                                                            
  |    +--> | sd_netlink_open_fd | alloc netlink and setup, bind socket to it                                                                                 
  |         +--------------------+                                                                                                                            
  |    +-------------------------+                                                                                                                            
  |--> | sd_netlink_attach_event | setup two sources: "netlink-receive-message" & "netlink-timer", add to event                                               
  |    +-------------------------+                                                                                                                            
  |    +-------------------+                                                                                                                                  
  |--> | netlink_add_match | register callback for match newlink/dellink                                                                                      
  |    +-------------------+ +---------------------------+                                                                                                    
  |                          | manager_rtnl_process_link | given msg, get link/net_dev from manager, perform 'new link' or 'del link' accordingly             
  |    +-------------------+ +---------------------------+                                                                                                    
  |--> | netlink_add_match | register callback for match newqdisc/delqdisc                                                                                    
  |    +-------------------+ +----------------------------+                                                                                                   
  |                          | manager_rtnl_process_qdisc | given msg, get link from manager, alloc qdisc, perform 'add qdisc' or 'del qdisc' accordingly     
  |    +-------------------+ +----------------------------+                                                                                                   
  |--> | netlink_add_match | register callback for match newtclass/deltclass                                                                                  
  |    +-------------------+ +-----------------------------+                                                                                                  
  |                          | manager_rtnl_process_tclass | given msg, get link from manager, alloc tclass, perform 'add tclass' or 'del tclass' accordingly 
  |    +-------------------+ +-----------------------------+                                                                                                  
  |--> | netlink_add_match | register callback for match new_addr / del_addr                                                                                  
  |    +-------------------+ +------------------------------+                                                                                                 
  |                          | manager_rtnl_process_address | given msg, get link from manager, alloc address, perform 'add addr' or 'del addr' accordingly   
  |    +-------------------+ +------------------------------+                                                                                                 
  |--> | netlink_add_match | register callback for match new_neigh / del_neigh                                                                                
  |    +-------------------+ +-------------------------------+                                                                                                
  |                          | manager_rtnl_process_neighbor | given msg, get link from manager, alloc neigh, perform 'add addr' or 'del addr' accordingly    
  |    +-------------------+ +-------------------------------+                                                                                                
  |--> | netlink_add_match | register callback for match new_route / del_route                                                                                
  |    +-------------------+ +----------------------------+                                                                                                   
  |                          | manager_rtnl_process_route | given msg, get link from manager, alloc route, perform 'add route' or 'del route' accordingly     
  |    +-------------------+ +----------------------------+                                                                                                   
  |--> | netlink_add_match | register callback for match new_rule / del_rule                                                                                  
  |    +-------------------+ +---------------------------+                                                                                                    
  |                          | manager_rtnl_process_rule |  given msg, get link from manager, alloc rule, perform 'add rule' or 'del rule' accordingly        
  |    +-------------------+ +---------------------------+                                                                                                    
  |--> | netlink_add_match | register callback for match new_nexthop / del_nexthop                                                                            
  |    +-------------------+ +------------------------------+                                                                                                 
  |                          | manager_rtnl_process_nexthop | given msg, get link from manager, alloc rule, perform 'add nexthop' or 'del nexthop' accordingly
  |                          +------------------------------+                                                                                                 
  |    +---------------------------+                                                                                                                          
  +--> | manager_setup_rtnl_filter | set socket option (attach filter table)                                                                                  
       +---------------------------+                                                                                                                          
```

```
src/network/tc/qdisc.c                                                                                                       
+----------------------------+                                                                                                
| manager_rtnl_process_qdisc | : given msg, get link from manager, alloc qdisc, perform 'add qdisc' or 'del qdisc' accordingly
+-|--------------------------+                                                                                                
  |    +-----------------------------+                                                                                        
  |--> | sd_netlink_message_get_type | get type (newqdisc or delqdisc) from msg                                               
  |    +-----------------------------+                                                                                        
  |    +---------------------------------------------+                                                                        
  |--> | sd_rtnl_message_traffic_control_get_ifindex | get iface idx                                                          
  |    +---------------------------------------------+                                                                        
  |    +-------------------+                                                                                                  
  |--> | link_get_by_index | giiven iface idx, get link from manager                                                          
  |    +-------------------+                                                                                                  
  |    +-----------+                                                                                                          
  |--> | qdisc_new | alloc qdisc                                                                                              
  |    +-----------+                                                                                                          
  |    +--------------------------------------------+                                                                         
  |--> | sd_rtnl_message_traffic_control_get_handle | get handle from msg                                                     
  |    +--------------------------------------------+                                                                         
  |    +--------------------------------------------+                                                                         
  |--> | sd_rtnl_message_traffic_control_get_parent | get parent from msg                                                     
  |    +--------------------------------------------+                                                                         
  |    +---------------------------------------+                                                                              
  |--> | sd_netlink_message_read_string_strdup | get tca_kind from msg                                                        
  |    +---------------------------------------+                                                                              
  |                                                                                                                           
  +--> switch type                                                                                                            
       case newqdisc                                                                                                          
       -    +-----------+                                                                                                     
       +--> | qdisc_add | ensure qdisc is added to link                                                                       
            +-----------+                                                                                                     
       case delqdisc                                                                                                          
       -     +------------+                                                                                                   
       +-->  | qdisc_free | ensure qdisc is removed from link and released                                                    
             +------------+                                                                                                   
```

```
src/network/networkd-address.c                                                                                                 
+------------------------------+                                                                                                
| manager_rtnl_process_address | : given msg, get link from manager, alloc address, perform 'add addr' or 'del addr' accordingly
+-|----------------------------+                                                                                                
  |    +-----------------------------+                                                                                          
  |--> | sd_netlink_message_get_type | get type (new_addr / del_addr) from msg                                                  
  |    +-----------------------------+                                                                                          
  |    +----------------------------------+                                                                                     
  |--> | sd_rtnl_message_addr_get_ifindex | get iface idx from msg                                                              
  |    +----------------------------------+                                                                                     
  |    +-------------------+                                                                                                    
  |--> | link_get_by_index | given iface idx, get link from manager                                                             
  |    +-------------------+                                                                                                    
  |    +-------------+                                                                                                          
  |--> | address_new | alloc and setup address                                                                                  
  |    +-------------+                                                                                                          
  |    +---------------------------------+                                                                                      
  |--> | sd_rtnl_message_addr_get_family | get family                                                                           
  |    +---------------------------------+                                                                                      
  |    +------------------------------------+                                                                                   
  |--> | sd_rtnl_message_addr_get_prefixlen | get prefix len                                                                    
  |    +------------------------------------+                                                                                   
  |    +--------------------------------+                                                                                       
  |--> | sd_rtnl_message_addr_get_scope | get scope                                                                             
  |    +--------------------------------+                                                                                       
  |    +-----------------------------+                                                                                          
  |--> | sd_netlink_message_read_u32 | get flags                                                                                
  |    +-----------------------------+                                                                                          
  |                                                                                                                             
  |--> switch familty                                                                                                           
  |    case inet                                                                                                                
  |    ---> get local/address/broadcast/label from msg                                                                          
  |    case inet6                                                                                                               
  |    ---> get local/address from msg                                                                                          
  |                                                                                                                             
  |    +-------------+                                                                                                          
  |--> | address_get | get address from link                                                                                    
  |    +-------------+                                                                                                          
  |                                                                                                                             
  +--> switch type                                                                                                              
       case new_addr                                                                                                            
       |--> if address isn't in link yet                                                                                        
       |    -    +-------------+                                                                                                
       |    +--> | address_add | add newly allocated address to link                                                            
       |         +-------------+                                                                                                
       |    +----------------+                                                                                                  
       +--> | address_update |                                                                                                  
            +----------------+                                                                                                  
       case del_addr                                                                                                            
       -    +--------------+                                                                                                    
       +--> | address_drop |                                                                                                    
            +--------------+                                                                                                    
```

```
src/network/networkd-manager.c                                                                                               
+----------------------------+                                                                                                
| manager_rtnl_process_route | : given msg, get link from manager, alloc route, perform 'add route' or 'del route' accordingly
+|---------------------------+                                                                                                
 |    +-----------------------------+                                                                                         
 |--> | sd_netlink_message_get_type | get type (add_route / del_route) from msg                                               
 |    +-----------------------------+                                                                                         
 |    +-----------------------------+                                                                                         
 |--> | sd_netlink_message_read_u32 | get iface idx from msg                                                                  
 |    +-----------------------------+                                                                                         
 |    +-------------------+                                                                                                   
 |--> | link_get_by_index | given iface idx, get link from manager                                                            
 |    +-------------------+                                                                                                   
 |    +-----------+                                                                                                           
 |--> | route_new | alloc and setup route                                                                                     
 |    +-----------+                                                                                                           
 |                                                                                                                            
 |--> from msg, get family/protocol/flags/dst/gateway/src/scope/type/table/priority/multi_path/cache_info                     
 |                                                                                                                            
 |    +-------------------+                                                                                                   
 +--> | process_route_one | give type, perform add_route / del_route accordingly                                              
      +-------------------+                                                                                                   
```

```
src/network/networkd-manager.c                                            
+---------------------------+                                              
| manager_setup_rtnl_filter | : set socket option (attach filter table)    
+-|-------------------------+                                              
  |                                                                        
  |--> prepare filter table                                                
  |                                                                        
  |    +--------------------------+                                        
  +--> | sd_netlink_attach_filter | set socket option (attach filter table)
       +--------------------------+                                        
```

```
src/network/networkd-manager.c                                                                                                             
+---------------------+                                                                                                                     
| manager_connect_bus | : register vtables, request name "org.freedesktop.network1", register callbacks for match rules                     
+-|-------------------+                                                                                                                     
  |    +---------------------------------------------+                                                                                      
  |--> | bus_open_system_watch_bind_with_description | get a bus, configure it and start                                                    
  |    +---------------------------------------------+                                                                                      
  |    +------------------------+                                                                                                           
  |--> | bus_add_implementation | register manager_object and vtable to dbus                                                                
  |    +------------------------+                                                                                                           
  |    +------------------------------+                                                                                                     
  |--> | bus_log_control_api_register | register log_control_object and vtable to dbus                                                      
  |    +------------------------------+                                                                                                     
  |    +---------------------------+                                                                                                        
  |--> | sd_bus_request_name_async | "org.freedesktop.network1"                                                                             
  |    +---------------------------+                                                                                                        
  |    +---------------------+                                                                                                              
  |--> | sd_bus_attach_event |                                                                                                              
  |    +---------------------+                                                                                                              
  |    +---------------------------+                                                                                                        
  |--> | sd_bus_match_signal_async | register callback for match rule ("Connected", when a service starts running bus, it emits such signal)
  |    +---------------------------+ +--------------+                                                                                       
  |                                  | on_connected | set hostname, set timezone, request product uuid                                      
  |                                  +--------------+                                                                                       
  |    +---------------------------+                                                                                                        
  +--> | sd_bus_match_signal_async | register callback for match rule ("PrepareForSleep")                                                   
       +---------------------------+ +-------------------------+                                                                            
                                     | match_prepare_for_sleep | configure each link in manager                                             
                                     +-------------------------+                                                                            
```

```
src/network/networkd-manager.c                                                                                            
+----------------------+                                                                                                   
| manager_connect_udev | : prepare monitor, add net/ieee80211/rfkill to it, regisster call (setup sd_device) to 'io' source
+-|--------------------+                                                                                                   
  |    +-----------------------+                                                                                           
  |--> | sd_device_monitor_new | alloc and setup sd_device_monitor                                                         
  |    +-----------------------+                                                                                           
  |    +-------------------------------------------+                                                                       
  |--> | sd_device_monitor_set_receive_buffer_size | set receive buffer size                                               
  |    +-------------------------------------------+                                                                       
  |    +------------------------------------------------------+                                                            
  |--> | sd_device_monitor_filter_add_match_subsystem_devtype | add (key, val) = ('net', null) to hashmap in monitor       
  |    +------------------------------------------------------+                                                            
  |    +------------------------------------------------------+                                                            
  |--> | sd_device_monitor_filter_add_match_subsystem_devtype | add (key, val) = ('ieee80211', null) to hashmap in monitor 
  |    +------------------------------------------------------+                                                            
  |    +------------------------------------------------------+                                                            
  |--> | sd_device_monitor_filter_add_match_subsystem_devtype | add (key, val) = ('rfkill', null) to hashmap in monitor    
  |    +------------------------------------------------------+                                                            
  |    +--------------------------------+                                                                                  
  |--> | sd_device_monitor_attach_event | ensure monitor has event                                                         
  |    +--------------------------------+                                                                                  
  |    +-------------------------+                                                                                         
  +--> | sd_device_monitor_start | attach event to monitor, register callback (setup sd_device) to 'io' source             
       +-------------------------+                                                                                         
```

```
src/network/networkd-manager.c                                                                          
+------------------------+                                                                               
| manager_process_uevent | : process uevent                                                              
+-|----------------------+                                                                               
  |    +----------------------+                                                                          
  |--> | sd_device_get_action | get action from device                                                   
  |    +----------------------+                                                                          
  |    +-------------------------+                                                                       
  |--> | sd_device_get_subsystem | get action from subsystem                                             
  |    +-------------------------+                                                                       
  |                                                                                                      
  |--> if subsystem is 'net'                                                                             
  |    |                                                                                                 
  |    |    +---------------------------+                                                                
  |    +--> | manager_udev_process_link | given device, get link frrom monitor and send packet (get link)
  |         +---------------------------+                                                                
  |                                                                                                      
  |--> elif subsystem is "ieee80211"                                                                     
  |    -                                                                                                 
  |    +--> (skip)                                                                                       
  |                                                                                                      
  +--> elif subsystem is "rfkill"                                                                        
       -                                                                                                 
       +--> (skip)                                                                                       
```

```
src/network/networkd-link.c                                                                   
+---------------------------+                                                                  
| manager_udev_process_link | : given device, get link frrom monitor and send packet (get link)
+-|-------------------------+                                                                  
  |    +-----------------------+                                                               
  |--> | sd_device_get_ifindex | get iface-index from device                                   
  |    +-----------------------+                                                               
  |    +-------------------+                                                                   
  |--> | link_get_by_index | given iface-index, get link from monitor                          
  |    +-------------------+                                                                   
  |    +------------------+                                                                    
  +--> | link_initialized | send rtnl packet (get link)                                        
       +------------------+                                                                    
```

```
src/network/networkd-link.c                                        
+------------------+                                                
| link_initialized | : send rtnl packet (get link)                  
+-|----------------+                                                
  |                                                                 
  |--> if link is dhcp client                                       
  |    |                                                            
  |    |    +------------------------------+                        
  |    +--> | sd_dhcp_client_attach_device | replace client's device
  |         +------------------------------+                        
  |    +---------------------------+                                
  |--> | link_set_sr_iov_ifindices | (???)                          
  |    +---------------------------+                                
  |    +-------------------+                                        
  +--> | link_call_getlink | send rtnl packet (get-link)            
       +-------------------+                                        
```

```
src/network/networkd-conf.c                                                               
+---------------------------+                                                              
| manager_parse_config_file | : parse /etc/systemd/networkd.conf                           
+|--------------------------+                                                              
 |    +--------------------------+                                                         
 +--> | config_parse_many_nulstr | collect related configs, parse them, add info to hashmap
      +--------------------------+ /etc/systemd/networkd.conf                              
```

```
src/network/networkd-manager.c                                                    
+---------------------+                                                            
| manager_load_config | : load *.netdev & *.network                                
+-|-------------------+                                                            
  |    +-------------+                                                             
  |--> | netdev_load | (probably not our case, I found no *.netdev file)           
  |    +-------------+                                                             
  |    +--------------+                                                            
  +--> | network_load | for each *.network, alloc 'network' and add to arg networks
       +--------------+                                                            
```

```
src/network/networkd-network.c                                                                                 
+--------------+                                                                                                
| network_load | : for each *.network, alloc 'network' and add to arg networks                                  
+-|------------+                                                                                                
  |    +----------------------+                                                                                 
  |--> | conf_files_list_strv | find *.network from predefined paths                                            
  |    +----------------------+                                                                                 
  |                                                                                                             
  +--> for each found file                                                                                      
       |                                                                                                        
       |    +------------------+                                                                                
       +--> | network_load_one | alloc 'network', parse '*.network', add default static route, add to 'networks'
            +------------------+                                                                                
```

```
src/network/networkd-network.c                                                                       
+------------------+                                                                                  
| network_load_one | : alloc 'network', parse '*.network', add default static route, add to 'networks'
+-|----------------+                                                                                  
  |                                                                                                   
  |--> alloc 'network'                                                                                
  |                                                                                                   
  |    +-------------------+                                                                          
  |--> | config_parse_many | parse *.network                                                          
  |    +-------------------+                                                                          
  |    +-------------------------------------+                                                        
  |--> | network_add_default_route_on_device | add default static route                               
  |    +-------------------------------------+                                                        
  |    +----------------------------+                                                                 
  +--> | ordered_hashmap_ensure_put | put 'network' in 'networks'                                     
       +----------------------------+                                                                 
```

```
src/network/networkd-manager.c                                                                                       
+-------------------+                                                                                                 
| manager_enumerate | : send all kinds of requests and add links/qdisk/tclass/addresses/neighbors/nexthop/routes/rules
+-|-----------------+                                                                                                 
  |    +-------------------------+                                                                                    
  |--> | manager_enumerate_links | send requet (get link), receive reply and perform 'new link'                       
  |    +-------------------------+                                                                                    
  |    +-------------------------+                                                                                    
  |--> | manager_enumerate_qdisc | send requet (get qdisc), receive reply and perform 'add qdisc'                     
  |    +-------------------------+                                                                                    
  |    +--------------------------+                                                                                   
  |--> | manager_enumerate_tclass | send requet (get class), receive reply and perform 'add tclass'                   
  |    +--------------------------+                                                                                   
  |    +-----------------------------+                                                                                
  |--> | manager_enumerate_addresses | send requet (get addr), receive reply and perform 'add addr'                   
  |    +-----------------------------+                                                                                
  |    +-----------------------------+                                                                                
  |--> | manager_enumerate_neighbors | send requet (get neigh), receive reply and perform 'add neigh'                 
  |    +-----------------------------+                                                                                
  |    +---------------------------+                                                                                  
  |--> | manager_enumerate_nexthop | send requet (get nexthop), receive reply and perform 'add nexthop'               
  |    +---------------------------+                                                                                  
  |    +--------------------------+                                                                                   
  |--> | manager_enumerate_routes | send requet (get route), receive reply and perform 'add route'                    
  |    +--------------------------+                                                                                   
  |    +-------------------------+                                                                                    
  |--> | manager_enumerate_rules | send requet (get rule), receive reply and perform 'add rule'                       
  |    +-------------------------+                                                                                    
  |    +---------------------------------+                                                                            
  |--> | manager_enumerate_nl80211_wiphy | (skip)                                                                     
  |    +---------------------------------+                                                                            
  |    +----------------------------------+                                                                           
  |--> | manager_enumerate_nl80211_config | (skip)                                                                    
  |    +----------------------------------+                                                                           
  |    +--------------------------------+                                                                             
  +--> | manager_enumerate_nl80211_mlme | (skip)                                                                      
       +--------------------------------+                                                                             
```

```
src/network/networkd-manager.c                                                                                   
+-------------------------+                                                                                       
| manager_enumerate_links | : send requet (get link), receive reply and perform 'new link'                        
+-|-----------------------+                                                                                       
  |    +--------------------------+                                                                               
  |--> | sd_rtnl_message_new_link | prepare msg (get link)                                                        
  |    +--------------------------+                                                                               
  |    +----------------------------+                                                                             
  +--> | manager_enumerate_internal | send request & receive replies, perform 'new link' or 'del link' accordingly
       +----------------------------+                                                                             
```

```
src/network/networkd-manager.c                                                                                                 
+----------------------------+                                                                                                  
| manager_enumerate_internal | : send request & receive replies, perform 'new link' or 'del link' accordingly                   
+-|--------------------------+                                                                                                  
  |    +-------------------------------------+                                                                                  
  |--> | sd_netlink_message_set_request_dump | set 'dump' flag                                                                  
  |    +-------------------------------------+                                                                                  
  |    +-----------------+                                                                                                      
  |--> | sd_netlink_call | send packet and read reply                                                                           
  |    +-----------------+                                                                                                      
  |                                                                                                                             
  +--> for each reply                                                                                                           
       -                                                                                                                        
       +--> call arg proces, e.g.,                                                                                              
            +---------------------------+                                                                                       
            | manager_rtnl_process_link | given msg, get link/net_dev from manager, perform 'new link' or 'del link' accordingly
            +---------------------------+                                                                                       
```

```
src/network/networkd-manager.c                                                                           
+---------------+                                                                                         
| manager_start | : start speed meter, save manager and links                                             
+-|-------------+                                                                                         
  |    +---------------------------+                                                                      
  |--> | manager_start_speed_meter | add speed meter (register callback for 'time' source)                
  |    +---------------------------+                                                                      
  |    +--------------+                                                                                   
  |--> | manager_save | save current manager settings to file, update manager settings, clear 'dirty' flag
  |    +--------------+                                                                                   
  |                                                                                                       
  +--> for each link                                                                                      
       |                                                                                                  
       |    +-----------+                                                                                 
       +--> | link_save | save link states to tmp file, rename to link->state_file                        
            +-----------+                                                                                 
```

```
src/libsystemd-network/sd-dhcp-client.c                                           
+--------------------+                                                             
| sd_dhcp_client_new | : alloc and setup dhcp_client, add default options to client
+-|------------------+                                                             
  |                                                                                
  |--> alloc dhcp_client                                                           
  |                                                                                
  |--> get default options                                                         
  |                                                                                
  +--> for each option                                                             
       |                                                                           
       |    +-----------------------------------+                                  
       +--> | sd_dhcp_client_set_request_option | ensure option is in client's set 
            +-----------------------------------+                                  
```

```
src/network/networkd-dhcp4.c                                                                            
+------------------+                                                                                     
| dhcp4_lease_lost | : remove addr/routes, reset mtu/hostname, queue requests of nexthop/route to manager
+-|----------------+                                                                                     
  |    +---------------------------------+                                                               
  |--> | dhcp4_remove_address_and_routes | remove each route and address from link                       
  |    +---------------------------------+                                                               
  |    +----------------+                                                                                
  |--> | dhcp_reset_mtu | prepare req (set link mtu) and add to queue of manager                         
  |    +----------------+                                                                                
  |    +---------------------+                                                                           
  |--> | dhcp_reset_hostname | determine hostname, call method to set hostname                           
  |    +---------------------+                                                                           
  |    +------------+                                                                                    
  |--> | link_dirty | label link as dirty, add to dirty list of manager                                  
  |    +------------+                                                                                    
  |    +------------------------------+                                                                  
  |--> | link_request_static_nexthops | for each nexthop in network: queue a request to manager          
  |    +------------------------------+                                                                  
  |    +----------------------------+                                                                    
  +--> | link_request_static_routes | for each route in network: queue a route request to manager        
       +----------------------------+                                                                    
```

```
src/network/networkd-dhcp4.c                                                                    
+---------------------------------+                                                              
| dhcp4_remove_address_and_routes | : remove each route and address from link                    
+-|-------------------------------+                                                              
  |                                                                                              
  |--> for each route in link                                                                    
  |    |                                                                                         
  |    |    +--------------+                                                                     
  |    |--> | route_remove | prepare rtnl msg (del route), setup and send out                    
  |    |    +--------------+                                                                     
  |    |    +----------------------+                                                             
  |    +--> | route_cancel_request | remove route-type request from manager                      
  |         +----------------------+                                                             
  |                                                                                              
  +--> for each address in link                                                                  
       |                                                                                         
       |    +----------------+                                                                   
       |--> | address_remove | prepare rtnl msg (del route), setup and send out, update operstate
       |    +----------------+                                                                   
       |    +------------------------+                                                           
       +--> | address_cancel_request | remove address-type request from manager                  
            +------------------------+                                                           
```

```
src/network/networkd-route.c                                      
+--------------+                                                   
| route_remove | : prepare rtnl msg (del route), setup and send out
+-|------------+                                                   
  |                                                                
  |--> get link from route                                         
  |                                                                
  |    +---------------------------+                               
  |--> | sd_rtnl_message_new_route | prepare rtnl msg (del route)  
  |    +---------------------------+                               
  |    +--------------------------------+                          
  |--> | sd_rtnl_message_route_set_type |                          
  |    +--------------------------------+                          
  |    +---------------------------+                               
  |--> | route_set_netlink_message | given route, setup msg        
  |    +---------------------------+                               
  |    +--------------------+                                      
  +--> | netlink_call_async | send msg out                         
       +--------------------+                                      
```

```
src/network/networkd-dhcp4.c                                                          
+----------------+                                                                     
| dhcp_reset_mtu | : prepare req (set link mtu) and add to queue of manager            
+-------------------------+                                                            
| link_request_to_set_mtu | : prepare req (set link mtu) and add to queue of manager   
+-|-----------------------+                                                            
  |                                                                                    
  |--> determine mtu                                                                   
  |                                                                                    
  |    +-----------------------+                                                       
  +--> | link_request_set_link | prepare req (set link mtu) and add to queue of manager
       +-----------------------+                                                       
```

```
src/network/networkd-dhcp4.c                                                   
+---------------------+                                                         
| dhcp_reset_hostname | : determine hostname, call method to set hostname       
+-|-------------------+                                                         
  |                                                                             
  |--> try to get hostname from 'dhcp_hostname'                                 
  |                                                                             
  |--> if got nothing                                                           
  |    |                                                                        
  |    |    +----------------------------+                                      
  |    +--> | sd_dhcp_lease_get_hostname | try to get hostname from 'dhcp_lease'
  |         +----------------------------+                                      
  |                                                                             
  |--> if got nothing, return                                                   
  |                                                                             
  |    +----------------------+                                                 
  +--> | manager_set_hostname | call method to set hostname                     
       +----------------------+                                                 
```

```
src/network/networkd-nexthop.c                                                              
+------------------------------+                                                             
| link_request_static_nexthops | : for each nexthop in network: queue a request to manager   
+-|----------------------------+                                                             
  |                                                                                          
  |--> for each nexthop in network                                                           
  |    |                                                                                     
  |    |    +----------------------+                                                         
  |    +--> | link_request_nexthop | ensure target nexthop exists, queue a request to manager
  |         +----------------------+                                                         
  |                                                                                          
  +--> set link state to 'configured' or 'configuring'                                       
```

```
src/network/networkd-nexthop.c                                                    
+----------------------+                                                           
| link_request_nexthop | : ensure target nexthop exists, queue a request to manager
+-|--------------------+                                                           
  |    +-------------+                                                             
  |--> | nexthop_get | given arg 'in', get target netxhop                          
  |    +-------------+                                                             
  |                                                                                
  |--> if got nothing                                                              
  |    |                                                                           
  |    |    +-------------+                                                        
  |    |--> | nexthop_dup | duplicate 'src' to 'dest'                              
  |    |    +-------------+                                                        
  |    |    +--------------------+                                                 
  |    |--> | nexthop_acquire_id | find an available id and save in nexthop        
  |    |    +--------------------+                                                 
  |    |    +-------------+                                                        
  |    +--> | nexthop_add | add nexthop to either link or manager                  
  |         +-------------+                                                        
  |    +-------------------------+                                                 
  +--> | link_queue_request_safe | prepare req and add to queue of manager         
       +-------------------------+                                                 
```

```
src/network/networkd-nexthop.c                                   
+-------------+                                                   
| nexthop_get | : given arg 'in', get target netxhop              
+-|-----------+                                                   
  |                                                               
  |--> get set 'nexthops' from link or manager                    
  |                                                               
  |    +---------+                                                
  |--> | set_get | given arg 'in', get nexthop from set 'nexthops'
  |    +---------+                                                
  |                                                               
  |--> if found, return                                           
  |                                                               
  +--> for each nexthop in nexthops                               
       |                                                          
       |--> compare nexthop and arg 'in' without id               
       |                                                          
       +--> if found, return                                      
```

```
src/network/networkd-nexthop.c            
+-------------+                            
| nexthop_dup | : duplicate 'src' to 'dest'
+-|-----------+                            
  |                                        
  |--> duplicate 'src' nexthop to 'dest'   
  |                                        
  +--> for each nexthop_grp in 'src'       
       -                                   
       +--> duplicate and add to 'dest'    
```

```
src/network/networkd-nexthop.c                                  
+--------------------+                                           
| nexthop_acquire_id | : find an available id and save in nexthop
+-|------------------+                                           
  |                                                              
  |--> for each network in manager                               
  |    -                                                         
  |    +--> for each nexthop in network                          
  |         -                                                    
  |         +--> put nexthop->id in local 'ids'                  
  |                                                              
  |--> find an available (unused) id                             
  |                                                              
  +--> save in nexthop                                           
```

```
src/network/networkd-nexthop.c                        
+-------------+                                        
| nexthop_add | : add nexthop to either link or manager
+-|-----------+                                        
  |                                                    
  |--> if nexthop should be owned by link              
  |    |                                               
  |    |    +----------------+                         
  |    +--> | set_ensure_put | add nexthop to link     
  |         +----------------+                         
  |                                                    
  |--> else                                            
  |    |                                               
  |    |    +----------------+                         
  |    +--> | set_ensure_put | add link to manager     
  |         +----------------+                         
  |                                                    
  +--> add (id, nexthop) to manager                    
```

```
src/network/networkd-route.c                                                               
+----------------------------+                                                              
| link_request_static_routes | : for each route in network: queue a route request to manager
+-|--------------------------+                                                              
  |                                                                                         
  |--> for each route in network                                                            
  |    |                                                                                    
  |    |    +---------------------------+                                                   
  |    +--> | link_request_static_route | queue a route request to manager                  
  |         +---------------------------+                                                   
  |    +-------------------------------+                                                    
  |--> | link_request_wireguard_routes | (skip)                                             
  |    +-------------------------------+                                                    
  |                                                                                         
  +--> set link state to 'configured' or 'configuring'                                      
```

```
src/network/networkd-route.c                                                                                 
+---------------------------+                                                                                 
| link_request_static_route | : queue a route request to manager                                              
+-|-------------------------+                                                                                 
  |                                                                                                           
  |--> if route needs no conversion                                                                           
  |    |                                                                                                      
  |    |    +--------------------+                                                                            
  |    |--> | link_request_route | ensure route exists, setup existing accordingly, queue a request to manager
  |    |    +--------------------+                                                                            
  |    +--> return                                                                                            
  |                                                                                                           
  |    +-------------------------+                                                                            
  +--> | link_queue_request_safe | prepare req and add to queue of manager                                    
       +-------------------------+                                                                            
```

```
src/network/networkd-route.c                                                                       
+--------------------+                                                                              
| link_request_route | : ensure route exists, setup existing accordingly, queue a request to manager
+-|------------------+                                                                              
  |    +-----------+                                                                                
  |--> | route_get | given arg 'in', get route from manager or link                                 
  |    +-----------+                                                                                
  |                                                                                                 
  |--> if route is outdated, remove it and return                                                   
  |                                                                                                 
  |--> if route doesn't exist                                                                       
  |    |                                                                                            
  |    |    +-----------+                                                                           
  |    |--> | route_dup | duplicate arg 'route'                                                     
  |    |    +-----------+                                                                           
  |    |    +-----------+                                                                           
  |    +--> | route_add | add route to manager or link                                              
  |         +-----------+                                                                           
  |                                                                                                 
  |--> else (exist)                                                                                 
  |    -                                                                                            
  |    +--> setup 'existing' based on route                                                         
  |                                                                                                 
  |    +-------------------------+                                                                  
  +--> | link_queue_request_safe | prepare req and add to queue of manager                          
       +-------------------------+                                                                  
```

```
src/network/networkd-dhcp4.c                                                           
+-----------------+                                                                     
| dhcp4_configure | : alloc client, configure and install 'dhcp4_handler'               
+-|---------------+                                                                     
  |    +--------------------+                                                           
  |--> | sd_dhcp_client_new | alloc and setup dhcp_client, add default options to client
  |    +--------------------+                                                           
  |    +-----------------------------+                                                  
  |--> | sd_dhcp_client_attach_event | attach event to dhcp_client                      
  |    +-----------------------------+                                                  
  |    +------------------------------+                                                 
  |--> | sd_dhcp_client_attach_device | attach link dev to dhcp_client                  
  |    +------------------------------+                                                 
  |    +------------------------+                                                       
  |--> | sd_dhcp_client_set_mac | set hw mac in dhcp_client                             
  |    +------------------------+                                                       
  |    +----------------------------+                                                   
  |--> | sd_dhcp_client_set_ifindex | set iface index in dhcp_client                    
  |    +----------------------------+                                                   
  |    +-----------------------------+                                                  
  |--> | sd_dhcp_client_set_callback | install callback in dhcp_client                  
  |    +-----------------------------+ +---------------+                                
  |                                    | dhcp4_handler | handle dhcp events             
  |                                    +---------------+                                
  |    +------------------------+                                                       
  |--> | sd_dhcp_client_set_mtu | set mtu of client                                     
  |    +------------------------+                                                       
  |                                                                                     
  |--> if not anonymize                                                                 
  |    |                                                                                
  |    |--> set options (mtu, routes, domains, ntp, timezone)                           
  |    |                                                                                
  |    |    +--------------------+                                                      
  |    +--> | dhcp4_set_hostname |                                                      
  |         +--------------------+                                                      
  |                                                                                     
  |--> set options (client port, max attempts, service type, socket priority, )         
  |                                                                                     
  |    +---------------------------+                                                    
  |--> | dhcp4_set_request_address | find dynamic addr from link, save it client        
  |    +---------------------------+                                                    
  |    +-----------------------------+                                                  
  +--> | dhcp4_set_client_identifier | set client id                                    
       +-----------------------------+                                                  
```

```
src/network/networkd-dhcp4.c                                                                                           
+---------------+                                                                                                       
| dhcp4_handler | : handle dhcp events                                                                                  
+-|-------------+                                                                                                       
  |                                                                                                                     
  +--> switch event                                                                                                     
       case stop                                                                                                        
       ---> (skip ipv4ll)                                                                                               
       +--> if link->dhcp_lease                                                                                         
            |--> if ->dhcp_send_release is set                                                                          
            |    -    +-----------------------------+                                                                   
            |    +--> | sd_dhcp_client_send_release | prepare client msg (dhcp release), send out                       
            |         +-----------------------------+                                                                   
            |    +------------------+                                                                                   
            +--> | dhcp4_lease_lost | remove addr/routes, reset mtu/hostname, queue requests of nexthop/route to manager
                 +------------------+                                                                                   
       case expired                                                                                                     
       +--> if link->dhcp_lease                                                                                         
            -    +------------------+                                                                                   
            +--> | dhcp4_lease_lost | remove addr/routes, reset mtu/hostname, queue requests of nexthop/route to manager
                 +------------------+                                                                                   
       case ip_change                                                                                                   
       -    +----------------------+                                                                                    
       +--> | dhcp_lease_ip_change | get lease, mark dirty on link, set hostname/timezone, request addrs/routes         
            +----------------------+                                                                                    
       case renew                                                                                                       
       -    +------------------+                                                                                        
       +--> | dhcp_lease_renew | get lease, mark dirty on link, request addrs/routes                                    
            +------------------+                                                                                        
       case ip_acquire                                                                                                  
       -    +---------------------+                                                                                     
       +--> | dhcp_lease_acquired | get lease, mark dirty on link, set hostname/timezone, request addrs/routes          
            +---------------------+                                                                                     
       case selecting                                                                                                   
       -    +-------------------------+                                                                                 
       +--> | dhcp_server_is_filtered | get lease and get server addr                                                   
            +-------------------------+                                                                                 
       case transient_failure                                                                                           
```

```
src/network/networkd-dhcp4.c                                                                        
+----------------------+                                                                             
| dhcp_lease_ip_change | : get lease, mark dirty on link, set hostname/timezone, request addrs/routes
+---------------------++                                                                             
| dhcp_lease_acquired | : get lease, mark dirty on link, set hostname/timezone, request addrs/routes 
+-|-------------------+                                                                              
  |    +--------------------------+                                                                  
  |--> | sd_dhcp_client_get_lease | get lease                                                        
  |    +--------------------------+                                                                  
  |    +------------+                                                                                
  |--> | link_dirty | label link as dirty, add to dirty list of manager                              
  |    +------------+                                                                                
  |                                                                                                  
  |--> if ->dhcp_use_mtu is true                                                                     
  |    |                                                                                             
  |    |    +-----------------------+                                                                
  |    |--> | sd_dhcp_lease_get_mtu | get mtu from lease                                             
  |    |    +-----------------------+                                                                
  |    |    +-------------------------+                                                              
  |    +--> | link_request_to_set_mtu | set rtnl request to set mtu                                  
  |         +-------------------------+                                                              
  |                                                                                                  
  |--> if ->dhcp_use_hostname is true                                                                
  |    |                                                                                             
  |    |--> get hostname from network or lease                                                       
  |    |                                                                                             
  |    +--> if got hostname                                                                          
  |         |                                                                                        
  |         |    +----------------------+                                                            
  |         +--> | manager_set_hostname | call bus method to set hostname                            
  |              +----------------------+                                                            
  |                                                                                                  
  |--> if ->dhcp_use_timezone is true                                                                
  |    |                                                                                             
  |    |    +----------------------------+                                                           
  |    |--> | sd_dhcp_lease_get_timezone | get timezone from lease                                   
  |    |    +----------------------------+                                                           
  |    |    +----------------------+                                                                 
  |    +--> | manager_set_timezone | call bus method to set timezone                                 
  |         +----------------------+                                                                 
  |    +----------------------------------+                                                          
  +--> | dhcp4_request_address_and_routes | mark 'dhcp' on addrs/routes, request addrs/routes        
       +----------------------------------+                                                          
```

```
src/network/networkd-dhcp4.c                                                                                         
+----------------------------------+                                                                                  
| dhcp4_request_address_and_routes | : mark 'dhcp' on addrs/routes, request addrs/routes                              
+-|--------------------------------+                                                                                  
  |    +---------------------+                                                                                        
  |--> | link_mark_addresses | dhcp4                                                                                  
  |    +---------------------+                                                                                        
  |    +------------------+                                                                                           
  |--> | link_mark_routes | dhcp4                                                                                     
  |    +------------------+                                                                                           
  |    +-----------------------+                                                                                      
  |--> | dhcp4_request_address | get params from lease, setup 'address', add to link, queue a req (address) to manager
  |    +-----------------------+                                                                                      
  |    +----------------------+                                                                                       
  +--> | dhcp4_request_routes | request all kinds of rroutes (prefix/static/dns/ntp)                                  
       +----------------------+                                                                                       
```

```
src/network/networkd-dhcp4.c                                                                                     
+-----------------------+                                                                                         
| dhcp4_request_address |  : get params from lease, setup 'address', add to link, queue a req (address) to manager
+-|---------------------+                                                                                         
  |    +---------------------------+                                                                              
  |--> | sd_dhcp_lease_get_address | get address from lease                                                       
  |    +---------------------------+                                                                              
  |    +---------------------------+                                                                              
  |--> | sd_dhcp_lease_get_netmask | get netmask from lease                                                       
  |    +---------------------------+                                                                              
  |    +-------------------------------------+                                                                    
  |--> | sd_dhcp_lease_get_server_identifier | get server addr from lease                                         
  |    +-------------------------------------+                                                                    
  |                                                                                                               
  |--> if arg 'announce' is true                                                                                  
  |    |                                                                                                          
  |    |    +--------------------------+                                                                          
  |    |--> | sd_dhcp_lease_get_router | get router addr from lease                                               
  |    |    +--------------------------+                                                                          
  |    |                                                                                                          
  |    +--> log                                                                                                   
  |                                                                                                               
  |    +-------------+                                                                                            
  |--> | address_new | alloc 'address'                                                                            
  |    +-------------+                                                                                            
  |                                                                                                               
  |--> setup 'address'                                                                                            
  |                                                                                                               
  |    +----------------------+                                                                                   
  +--> | link_request_address | prepare 'address', add it to link, queue a request (address) to manager           
       +----------------------+                                                                                   
```

```
src/network/networkd-address.c                                                                     
+----------------------+                                                                            
| link_request_address | : prepare 'address', add it to link, queue a request (address) to manager  
+-|--------------------+                                                                            
  |    +-----------------+                                                                          
  |--> | address_acquire | prepare 'address' from arg 'original', with new prefix & ipaddr          
  |    +-----------------+                                                                          
  |                                                                                                 
  |--> if address needs to set broadcast                                                            
  |    |                                                                                            
  |    |    +-----------------------+                                                               
  |    +--> | address_set_broadcast | prepare bcast addr, save in arg address                       
  |         +-----------------------+                                                               
  |    +-------------+                                                                              
  |--> | address_get | given 'address', get existing one from link                                  
  |    +-------------+                                                                              
  |                                                                                                 
  |--> if there's no existing one                                                                   
  |    |                                                                                            
  |    |    +-------------+                                                                         
  |    |--> | address_dup | duplicate 'address' to 'tmp'                                            
  |    |    +-------------+                                                                         
  |    |    +-------------+                                                                         
  |    +--> | address_add | add newly allocated address to link                                     
  |         +-------------+                                                                         
  |                                                                                                 
  |--> else (there's existing one)                                                                  
  |    -                                                                                            
  |    +--> setup 'existing' based on 'address'                                                     
  |                                                                                                 
  |    +-------------------+                                                                        
  |--> | ipv4acd_configure | (skip)                                                                 
  |    +-------------------+                                                                        
  |    +-------------------------+                                                                  
  +--> | link_queue_request_safe | prepare req and add to queue of manager                          
       +-------------------------+ +-------------------------+                                      
                                   | address_process_request | prepare rtnl msg (new addr), send out
                                   +-------------------------+                                      
                                                                                                    
```

```
src/network/networkd-address.c                                                      
+-----------------+                                                                  
| address_acquire | : prepare 'address' from arg 'original', with new prefix & ipaddr
+-|---------------+                                                                  
  |    +----------------------+                                                      
  |--> | address_pool_acquire | get a random prefix from manager's pool              
  |    +----------------------+                                                      
  |                                                                                  
  |--> pick the first ip addr in range                                               
  |                                                                                  
  |    +-------------+                                                               
  |--> | address_dup | duplicate address from 'original'                             
  |    +-------------+                                                               
  |                                                                                  
  +--> save the first ip addr in duplicated address                                  
```

```
src/network/networkd-address-pool.c                              
+----------------------+                                          
| address_pool_acquire | : get a random prefix from manager's pool
+-|--------------------+                                          
  |                                                               
  +--> for each pool in manager                                   
       |                                                          
       |    +--------------------------+                          
       |--> | address_pool_acquire_one | get a random prefix      
       |    +--------------------------+                          
       |                                                          
       +--> if got, return                                        
```

```
src/network/networkd-dhcp4.c                                                                       
+----------------------+                                                                            
| dhcp4_request_routes | : request all kinds of rroutes (prefix/static/dns/ntp)                     
+-|--------------------+                                                                            
  |    +----------------------------+                                                               
  |--> | dhcp4_request_prefix_route | prepare 'route', queue a request to manager                   
  |    +----------------------------+                                                               
  |    +-----------------------------+                                                              
  |--> | dhcp4_request_static_routes | setup rout3es, queue a request to manager                    
  |    +-----------------------------+                                                              
  |    +----------------------------------+                                                         
  |--> | dhcp4_request_semi_static_routes | for each route, setup route & queue a request to manager
  |    +----------------------------------+                                                         
  |    +-----------------------------+                                                              
  |--> | dhcp4_request_routes_to_dns | request routes to dns servers                                
  |    +-----------------------------+                                                              
  |    +-----------------------------+                                                              
  +--> | dhcp4_request_routes_to_ntp | request routes to ntp servers                                
       +-----------------------------+                                                              
```

```
src/network/networkd-dhcp4.c                                                                             
+----------------------------+                                                                            
| dhcp4_request_prefix_route | : prepare 'route', queue a request to manager                              
+-|--------------------------+                                                                            
  |    +---------------------------+                                                                      
  |--> | sd_dhcp_lease_get_address | get addr from lease                                                  
  |    +---------------------------+                                                                      
  |    +---------------------------+                                                                      
  |--> | sd_dhcp_lease_get_netmask | get netmask from lease                                               
  |    +---------------------------+                                                                      
  |    +-----------+                                                                                      
  |--> | route_new | alloc route                                                                          
  |    +-----------+                                                                                      
  |                                                                                                       
  |--> setup route                                                                                        
  |                                                                                                       
  |    +---------------------+                                                                            
  +--> | dhcp4_request_route | ensure route exists, setup existing accordingly, queue a request to manager
       +---------------------+                                                                            
```

```
src/network/networkd-dhcp4.c                                                                            
+---------------------+                                                                                  
| dhcp4_request_route | : ensure route exists, setup existing accordingly, queue a request to manager    
+-|-------------------+                                                                                  
  |    +-------------------------------------+                                                           
  |--> | sd_dhcp_lease_get_server_identifier | get server addr from lease                                
  |    +-------------------------------------+                                                           
  |                                                                                                      
  |--> ensure 'route' is setup                                                                           
  |                                                                                                      
  |    +-----------+                                                                                     
  |--> | route_get | get route from manager or link                                                      
  |    +-----------+                                                                                     
  |    +--------------------+                                                                            
  +--> | link_request_route | ensure route exists, setup existing accordingly, queue a request to manager
       +--------------------+                                                                            
```

```
src/network/networkd-dhcp4.c                                                            
+-----------------------------+                                                          
| dhcp4_request_static_routes | : setup rout3es, queue a request to manager              
+-|---------------------------+                                                          
  |    +---------------------------------+                                               
  |--> | sd_dhcp_lease_get_static_routes | alloc buf to save static route addresses      
  |    +---------------------------------+                                               
  |    +------------------------------------+                                            
  |--> | sd_dhcp_lease_get_classless_routes | alloc buf to save classless route addresses
  |    +------------------------------------+                                            
  |                                                                                      
  |--> if ->dhcp_use_routes isn't set                                                    
  |    |                                                                                 
  |    |--> try to find a valid route                                                    
  |    |                                                                                 
  |    +--> return                                                                       
  |                                                                                      
  |--> determine routes (static or classless)                                            
  |                                                                                      
  +--> for each route                                                                    
       |                                                                                 
       |--> alloc route                                                                  
       |                                                                                 
       |--> get gateway/prefixLen/destination from route                                 
       |                                                                                 
       |    +--------------------------+                                                 
       +--> | dhcp4_request_route_auto | setup route, queue a request to manager         
            +--------------------------+                                                 
```

```
src/libsystemd-network/sd-dhcp-lease.c                                
+---------------------------------+                                    
| sd_dhcp_lease_get_static_routes | : alloc buf to save route addresses
+-----------------------+---------+                                    
| dhcp_lease_get_routes | : alloc buf to save route addresses          
+-|---------------------+                                              
  |                                                                    
  |--> alloc buf for route_ptr[]                                       
  |                                                                    
  +--> save addresses of each route from arg                           
```

```
src/network/networkd-dhcp4.c                                                                             
+--------------------------+                                                                              
| dhcp4_request_route_auto | : setup route, queue a request to manager                                    
+-|------------------------+                                                                              
  |                                                                                                       
  |--> get address & route from lease                                                                     
  |                                                                                                       
  +--> determine scope/family/gw of route                                                                 
  |                                                                                                       
  |    +---------------------+                                                                            
  +--> | dhcp4_request_route | ensure route exists, setup existing accordingly, queue a request to manager
       +---------------------+                                                                            
```

```
src/network/networkd-dhcp4.c                                                                                  
+----------------------------------+                                                                           
| dhcp4_request_semi_static_routes | : for each route, setup route & queue a request to manager                
+-|--------------------------------+                                                                           
  |                                                                                                            
  +--> for each route in network                                                                               
       |                                                                                                       
       |    +--------------------------------+                                                                 
       |--> | dhcp4_request_route_to_gateway | setup route, queue a request to manager                         
       |    +--------------------------------+                                                                 
       |    +-----------+                                                                                      
       |--> | route_dup | duplicate route                                                                      
       |    +-----------+                                                                                      
       |    +---------------------+                                                                            
       +--> | dhcp4_request_route | ensure route exists, setup existing accordingly, queue a request to manager
            +---------------------+                                                                            
```

```
src/network/networkd-dhcp4.c                                                                
+------------------+                                                                         
| dhcp_lease_renew | : get lease, mark dirty on link, request addrs/routes                   
+-|----------------+                                                                         
  |    +--------------------------+                                                          
  |--> | sd_dhcp_client_get_lease | get lease                                                
  |    +--------------------------+                                                          
  |    +------------+                                                                        
  |--> | link_dirty | label link as dirty, add to dirty list of manager                      
  |    +------------+                                                                        
  |    +----------------------------------+                                                  
  +--> | dhcp4_request_address_and_routes | mark 'dhcp' on addrs/routes, request addrs/routes
       +----------------------------------+                                                  
```

```
src/network/networkd-dhcp4.c                                              
+---------------------------+                                              
| dhcp4_set_request_address | : find dynamic addr from link, save it client
+-|-------------------------+                                              
  |                                                                        
  |--> find dynamic addr from link                                         
  |                                                                        
  |    +------------------------------------+                              
  +--> | sd_dhcp_client_set_request_address | set addr in client           
       +------------------------------------+                              
```

```
src/network/networkd-dhcp4.c                                                                         
+---------------------------+                                                                         
| link_request_dhcp4_client | : queue a request to start dhcp4 client                                 
+-|-------------------------+                                                                         
  |                                                                                                   
  |--> if dhcp4 isn't enabled yet, return                                                             
  |                                                                                                   
  |--> if link has dhcp_client already, return                                                        
  |                                                                                                   
  |    +--------------------+                                                                         
  +--> | link_queue_request | prepare req and add to queue of manager                                 
       +--------------------+ +-----------------------+                                               
                              | dhcp4_process_request | request dev_uid, configure client and start it
                              +-----------------------+                                               
```

```
src/network/networkd-dhcp4.c                                                                    
+-----------------------+                                                                        
| dhcp4_process_request | : request dev_uid, configure client and start it                       
+-|---------------------+                                                                        
  |    +----------------------+                                                                  
  |--> | dhcp4_configure_duid | request dev uid                                                  
  |    +----------------------+                                                                  
  |    +-----------------+                                                                       
  |--> | dhcp4_configure | alloc client, configure and install 'dhcp4_handler'                   
  |    +-----------------+                                                                       
  |    +-------------+                                                                           
  +--> | dhcp4_start | setup client, prepare raw socket for client, install callback for io event
       +-------------+                                                                           
```

```
src/network/networkd-dhcp4.c                                                                        
+-------------+                                                                                      
| dhcp4_start | : setup client, prepare raw socket for client, install callback for io event         
+----------------------+                                                                             
| sd_dhcp_client_start | : setup client, prepare raw socket for client, install callback for io event
+-|--------------------+                                                                             
  |    +-------------------+                                                                         
  |--> | client_initialize |                                                                         
  |    +-------------------+                                                                         
  |    +--------------+                                                                              
  +--> | client_start | setup raw socket and bind it, install callback for io event                  
       +--------------+                                                                              
```

```
src/libsystemd-network/sd-dhcp-client.c                                                             
+--------------+                                                                                     
| client_start | : setup raw socket and bind it, install callback for io event                       
+----------------------+                                                                             
| client_start_delayed | : setup raw socket and bind it, install callback for io event               
+-|--------------------+                                                                             
  |    +------------------------------+                                                              
  |--> | dhcp_network_bind_raw_socket | given arp_type, setup raw socket and bind it                 
  |    +------------------------------+                                                              
  |                                                                                                  
  |--> save socket fd in client                                                                      
  |                                                                                                  
  |    +--------------------------+                                                                  
  +--> | client_initialize_events | install io_callback to event                                     
       +--------------------------+ +----------------------------+                                   
                                    | client_receive_message_raw | receive msg, handle it accordingly
                                    +----------------------------+                                   
```

```
src/libsystemd-network/dhcp-network.c                                         
+------------------------------+                                               
| dhcp_network_bind_raw_socket | : given arp_type, setup raw socket and bind it
+-|----------------------------+                                               
  |                                                                            
  +--> switch arp_type                                                         
                                                                               
       case ether                                                              
       |                                                                       
       |    +------------------+                                               
       +--> | _bind_raw_socket | setup raw socket and bind it                  
            +------------------+                                               
                                                                               
       case infiniband                                                         
       |                                                                       
       |    +------------------+                                               
       +--> | _bind_raw_socket | setup raw socket and bind it                  
            +------------------+                                               
```

```
src/libsystemd-network/dhcp-network.c             
+------------------+                               
| _bind_raw_socket | : setup raw socket and bind it
+-|----------------+                               
  |                                                
  |--> setup filter                                
  |                                                
  |    +--------+                                  
  |--> | socket |                                  
  |    +--------+                                  
  |    +------------+                              
  |--> | setsockopt | attach filter to socket      
  |    +------------+                              
  |    +------+                                    
  +--> | bind | bind socket to addr                
       +------+                                    
```

```
src/libsystemd-network/sd-dhcp-client.c                                   
+----------------------------+                                             
| client_receive_message_raw | : receive msg, handle it accordingly        
+-|--------------------------+                                             
  |                                                                        
  |--> get packet size and alloc buffer                                    
  |                                                                        
  |    +--------------+                                                    
  |--> | recvmsg_safe |                                                    
  |    +--------------+                                                    
  |    +-----------------------+                                           
  +--> | client_handle_message | given client state, handle msg accordingly
       +-----------------------+                                           
```

```
src/libsystemd-network/sd-dhcp-client.c                                                  
+-----------------------+                                                                 
| client_handle_message | : given client state, handle msg accordingly                    
+-|---------------------+                                                                 
  |                                                                                       
  +--> switch client state                                                                
       case selecting                                                                     
       |    +---------------------+                                                       
       |--> | client_handle_offer | parse options and save in lease, save lease in clientt
       |    +---------------------+                                                       
       +--> client state = requesting                                                     
       case rebooting                                                                     
       case requesting                                                                    
       case renewing                                                                      
       case rebinding                                                                     
       |    +-------------------+                                                         
       |--> | client_handle_ack | alloc lease, parse 'ack', save lease in client          
       |    +-------------------+                                                         
       |                                                                                  
       |--> close client->fd                                                              
       |                                                                                  
       |--> client state = bound                                                          
       |                                                                                  
       |    +---------------------------+                                                 
       |--> | client_set_lease_timeouts |                                                 
       |    +---------------------------+                                                 
       |    +------------------------------+                                              
       |--> | dhcp_network_bind_udp_socket | prepare socket, bind to src addr             
       |    +------------------------------+                                              
       |                                                                                  
       |--> save fd in client                                                             
       |                                                                                  
       +--> install callback for io event                                                 
       case bound                                                                         
       -    +--------------------------+                                                  
       +--> | client_handle_forcerenew | parse options but do nothing then                
            +--------------------------+                                                  
```

```
src/libsystemd-network/sd-dhcp-client.c                                                                     
+---------------------+                                                                                      
| client_handle_offer | : parse options and save in lease, save lease in clientt                             
+-|-------------------+                                                                                      
  |                                                                                                          
  |--> alloc 'lease'                                                                                         
  |                                                                                                          
  |    +-------------------+                                                                                 
  |--> | dhcp_option_parse | parse options and save in lease, return msg_type                                
  |    +-------------------+ +--------------------------+                                                    
  |                          | dhcp_lease_parse_options | given code, parse option accordingly, save in lease
  |                          +--------------------------+                                                    
  |                                                                                                          
  |--> save lease in client                                                                                  
  |                                                                                                          
  |    +---------------+                                                                                     
  +--> | client_notify | notify (event = selecting)                                                          
       +---------------+                                                                                     
```

```
src/libsystemd-network/dhcp-option.c                                 
+-------------------+                                                 
| dhcp_option_parse | parse options and save in lease, return msg_type
+-|-----------------+                                                 
  |    +---------------+                                              
  +--> | parse_options | parse options and save in lease              
       +---------------+                                              
```

```
src/libsystemd-network/dhcp-option.c              
+---------------+                                  
| parse_options | : parse options and save in lease
+-|-------------+                                  
  |                                                
  +--> while traversing options                    
       |                                           
       |--> get (code, len, option) from options   
       |                                           
       |--> if end, return                         
       |                                           
       +--> switch code                            
            -                                      
            +--> parse option accordingly          
```

```
src/libsystemd-network/sd-dhcp-client.c                                     
+-------------------+                                                        
| client_handle_ack | : alloc lease, parse 'ack', save lease in client       
+-|-----------------+                                                        
  |    +----------------+                                                    
  |--> | dhcp_lease_new | alloc lease                                        
  |    +----------------+                                                    
  |    +-------------------+                                                 
  |--> | dhcp_option_parse | parse options and save in lease, return msg_type
  |    +-------------------+                                                 
  |                                                                          
  |--> get server/addr from ack                                              
  |                                                                          
  |--> determine event (ip_acquire / renew / ip_change)                      
  |                                                                          
  +--> save lease in client                                                  
```

```
src/libsystemd-network/dhcp-network.c                                     
+------------------------------+                                           
| dhcp_network_bind_udp_socket | : prepare socket, bind to src addr        
+-|----------------------------+                                           
  |    +--------+                                                          
  |--> | socket |                                                          
  |    +--------+                                                          
  |                                                                        
  |--> set socket options                                                  
  |                                                                        
  |--> if ifindex is provided                                              
  |    |                                                                   
  |    |    +------------------------+                                     
  |    +--> | socket_bind_to_ifindex | set socket option to bind to ifindex
  |         +------------------------+                                     
  |    +------+                                                            
  +--> | bind | bind socket to src_addr                                    
       +------+                                                            
```

```
src/libsystemd/sd-netlink/sd-netlink.c                                                          
+-------------+                                                                                  
| io_callback | : dispatch 1st msg from rqueue, find matched entry from netlink and run it       
+--------------------+                                                                           
| sd_netlink_process | : dispatch 1st msg from rqueue, find matched entry from netlink and run it
+-----------------+--+                                                                           
| process_running |                                                                              
+-|---------------+                                                                              
  |    +-----------------+                                                                       
  |--> | process_timeout | run the first reply_callback in queue if it's timeout                 
  |    +-----------------+                                                                       
  |    +-----------------+                                                                       
  |--> | dispatch_rqueue | dispatch first msg from rqueue                                        
  |    +-----------------+                                                                       
  |                                                                                              
  |--> if msg is broadcast                                                                       
  |    |                                                                                         
  |    |    +---------------+                                                                    
  |    +--> | process_match | find type/cmd matched entry in netlink, call ->callback()          
  |         +---------------+                                                                    
  |                                                                                              
  +--> else                                                                                      
       |                                                                                         
       |    +---------------+                                                                    
       +--> | process_reply | given serial from msg, remove target entry from netlink, run it    
            +---------------+                                                                    
```

```
src/libsystemd/sd-netlink/sd-netlink.c                                    
+-----------------+                                                        
| process_timeout | : run the first reply_callback in queue if it's timeout
+-|---------------+                                                        
  |    +------------+                                                      
  |--> | prioq_peek | peek the 1st reply_callback in queue                 
  |    +------------+                                                      
  |                                                                        
  |-->  if its not timeout yet, return                                     
  |                                                                        
  |    +-----------------------------+                                     
  |    | message_new_synthetic_error | prepare msg of error code           
  |    +-----------------------------+                                     
  |    +----------------+                                                  
  |--> | hashmap_remove | remove that reply_callback from queue            
  |    +----------------+                                                  
  |    +--------------+                                                    
  |--> | container_of | get outer slot                                     
  |    +--------------+                                                    
  |                                                                        
  +--> call ->callback()                                                   
```

```
src/libsystemd/sd-netlink/sd-netlink.c                                                                                         
+---------------+                                                                                                               
| process_match | : find type/cmd matched entry in netlink, call ->callback()                                                   
+-|-------------+                                                                                                               
  |    +-----------------------------+                                                                                          
  |--> | sd_netlink_message_get_type | get type (e.g., new link) from msg                                                       
  |    +-----------------------------+                                                                                          
  |                                                                                                                             
  |--> if msg protocol is 'generic'                                                                                             
  |    |                                                                                                                        
  |    |    +-----------------------------+                                                                                     
  |    +--> | sd_genl_message_get_command | get cmd (?) from msg                                                                
  |         +-----------------------------+                                                                                     
  |                                                                                                                             
  +--> for each callback in netlink                                                                                             
       |                                                                                                                        
       |--> if type or cmd isn't matched, continue                                                                              
       |                                                                                                                        
       |--> check if msg aims for any group in callback                                                                         
       |                                                                                                                        
       +--> cal ->callback(), e.g.,                                                                                             
            +---------------------------+                                                                                       
            | manager_rtnl_process_link | given msg, get link/net_dev from manager, perform 'new link' or 'del link' accordingly
            +---------------------------+                                                                                       
```

```
src/libsystemd/sd-netlink/sd-netlink.c                                                                                    
+---------------+                                                                                                          
| process_reply | : given serial from msg, remove target entry from netlink, run it                                        
+-|-------------+                                                                                                          
  |    +--------------------+                                                                                              
  |--> | message_get_serial | get serial from msg                                                                          
  |    +--------------------+                                                                                              
  |    +----------------+                                                                                                  
  |--> | hashmap_remove | given serial, remove target reply_callback from netlink                                          
  |    +----------------+                                                                                                  
  |    +-----------------------------+                                                                                     
  |--> | sd_netlink_message_get_type | get type (e.g., new link) from msg                                                  
  |    +-----------------------------+                                                                                     
  |                                                                                                                        
  +--> cal ->callback(), e.g.,                                                                                             
       +---------------------------+                                                                                       
       | manager_rtnl_process_link | given msg, get link/net_dev from manager, perform 'new link' or 'del link' accordingly
       +---------------------------+                                                                                       
```
