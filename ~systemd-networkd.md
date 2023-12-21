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

```
