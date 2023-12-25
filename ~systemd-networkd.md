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
