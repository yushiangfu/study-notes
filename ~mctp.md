### mctpd from Code Construct

```
src/mctpd.c                                                              
+------+                                                                  
| main |                                                                  
+-|----+                                                                  
  |    +--------------+                                                   
  |--> | setup_config | init ctx, set self-defined uuid                   
  |    +--------------+                                                   
  |    +------------+                                                     
  |--> | parse_args |                                                     
  |    +------------+                                                     
  |    +-------------+                                                    
  |--> | mctp_nl_new | prepare netlink socket, set up entries and save eid
  |    +-------------+                                                    
  |    +-------------+                                                    
  |--> | mctp_nl_new | create another one for query                       
  |    +-------------+                                                    
  |    +-----------+                                                      
  |--> | setup_bus | set up dbus paths and ifaces                         
  |    +-----------+                                                      
  |    +----------------+                                                 
  |--> | listen_monitor | monitor netlink and refresh linkmap if necessary
  |    +----------------+                                                 
  |    +------------+                                                     
  |--> | setup_nets | for each iface idx in ctx, add net/peer to ctx      
  |    +------------+                                                     
  |    +--------------------+                                             
  |--> | listen_control_msg | register handler for incoming request       
  |    +--------------------+                                             
  |    +---------------+                                                  
  |--> | setup_testing | (skip)                                           
  |    +---------------+                                                  
  |    +--------------+                                                   
  |--> | request_dbus | request service name "xyz.openbmc_project.MCTP"   
  |    +--------------+                                                   
  |    +---------------+                                                  
  +--> | sd_event_loop |                                                  
       +---------------+                                                  
```

```
src/mctp-netlink.c                                                  
+-------------+                                                      
| mctp_nl_new | : prepare netlink socket, set up entries and save eid
+-|-----------+                                                      
  |                                                                  
  |--> alloc 'mctp_nl' & set up                                      
  |                                                                  
  |    +----------------+                                            
  |--> | open_nl_socket | open and set up netlink socket             
  |    +----------------+                                            
  |    +--------------+                                              
  +--> | fill_linkmap | set up entries and save eid                  
       +--------------+                                              
```

```
src/mctp-netlink.c                                                                 
+--------------+                                                                    
| fill_linkmap | : set up entries and save eid                                      
+-|------------+ (each entry represents an iface and contains a few eid)            
  |                                                                                 
  |--> set up msg of 'get link'                                                     
  |                                                                                 
  |    +--------------+                                                             
  |--> | mctp_nl_send | send packet of netlink family                               
  |    +--------------+                                                             
  |                                                                                 
  |--> endless loop                                                                 
  |    |                                                                            
  |    |    +----------+                                                            
  |    |--> | recvfrom | (peek)                                                     
  |    |    +----------+                                                            
  |    |                                                                            
  |    |--> prepare larger buffer is necessary                                      
  |    |                                                                            
  |    |    +----------+                                                            
  |    |--> | recvfrom |                                                            
  |    |    +----------+                                                            
  |    |    +--------------------+                                                  
  |    +--> | parse_getlink_dump | set up entry for each msg, add to map            
  |         +--------------------+                                                  
  |    +------------------+                                                         
  |--> | fill_local_addrs | send packet to get all eid, save them all in one 'entry'
  |    +------------------+                                                         
  |    +--------------+                                                             
  +--> | sort_linkmap | sort linkmap and eid                                        
       +--------------+                                                             
```

```
src/mctp-netlink.c                                                 
+--------------+                                                    
| mctp_nl_send | : send packet of netlink family                    
+-|------------+                                                    
  |                                                                 
  |--> set up 'addr' and send                                       
  |                                                                 
  +--> if 'ack' is labled in flags                                  
       |                                                            
       |    +------------------+                                    
       +--> | handle_nlmsg_ack | handle error or dump unexpected msg
            +------------------+                                    
```

```
src/mctp-netlink.c                                                       
+--------------------+                                                    
| parse_getlink_dump | : set up entry for each msg, add to map            
+-|------------------+                                                    
  |                                                                       
  +--> for each msg in list                                               
       |                                                                  
       |    +-----------------------+                                     
       |--> | mctp_get_rtnlmsg_attr | get attr of 'af spec'               
       |    +-----------------------+                                     
       |    +-----------------------+                                     
       |--> | mctp_get_rtnlmsg_attr | get attr of 'mctp'                  
       |    +-----------------------+                                     
       |    +---------------------------+                                 
       |--> | mctp_get_rtnlmsg_attr_u32 | get attr of 'mctp net'          
       |    +---------------------------+                                 
       |    +-----------------------+                                     
       |--> | mctp_get_rtnlmsg_attr | get attr of 'interface name'        
       |    +-----------------------+                                     
       |    +-------------------+                                         
       +--> | linkmap_add_entry | extend array and set up entry info there
            +-------------------+                                         
```

```
src/mctp-netlink.c                                                            
+------------------+                                                           
| fill_local_addrs | : send packet to get all eid, save them all in one 'entry'
+-|----------------+                                                           
  |                                                                            
  |--> set up msg of 'get addr'                                                
  |                                                                            
  |    +---------------+                                                       
  |--> | mctp_nl_query | send packet, assemble all responses into one buffer   
  |    +---------------+                                                       
  |                                                                            
  +--> for each msg                                                            
       |                                                                       
       |    +--------------------------+                                       
       |--> | mctp_get_rtnlmsg_attr_u8 | get attr of 'local (eid)'             
       |    +--------------------------+                                       
       |    +---------------+                                                  
       +--> | entry_byindex | given idx, get target entry                      
            +---------------+                                                  
```

```
src/mctp-netlink.c                                                    
+---------------+                                                      
| mctp_nl_query | : send packet, assemble all responses into one buffer
+-|-------------+                                                      
  |    +--------------+                                                
  |--> | mctp_nl_send | send packet of netlink family                  
  |    +--------------+                                                
  |    +------------------+                                            
  +--> | mctp_nl_recv_all | receive all responses into one buffer      
       +------------------+                                            
```

```
src/mctp-netlink.c                                                
+------------------+                                               
| mctp_nl_recv_all | : receive all responses into one buffer       
+-|----------------+                                               
  |                                                                
  +--> while not done                                              
       |                                                           
       |    +----------+                                           
       |--> | recvfrom | (peek)                                    
       |    +----------+                                           
       |                                                           
       |--> prepare larger buffer if necessary                     
       |                                                           
       |    +----------+                                           
       |--> | recvfrom |                                           
       |    +----------+                                           
       |    +-----------------+                                    
       +--> | nlmsgs_are_done | scan msg list to check if it's done
            +-----------------+                                    
```

```
src/mctpd.c                                                         
+----------------+                                                   
| listen_monitor | : monitor netlink and refresh linkmap if necessary
+-|--------------+                                                   
  |    +-----------------+                                           
  |--> | mctp_nl_monitor | enable monitor                            
  |    +-----------------+                                           
  |    +-----------------+                                           
  +--> | sd_event_add_io | register callback of monitor              
       +-----------------+ +-------------------+                     
                           | cb_listen_monitor | refresh linkmap     
                           +-------------------+                     
```

```
src/mctp-netlink.c                                                  
+-----------------+                                                  
| mctp_nl_monitor | : enable or disable monitor                      
+-|---------------+                                                  
  |                                                                  
  |--> if arg 'enable' is set                                        
  |    |                                                             
  |    |--> if monitor already exists, return it                     
  |    |                                                             
  |    |    +----------------+                                       
  |    +--> | open_nl_socket | open netlink socket, use it as monitor
  |         +----------------+                                       
  |                                                                  
  +--> else                                                          
       -                                                             
       +--> close monitor (socket)                                   
```

```
src/mctpd.c                                                                             
+-------------------+                                                                    
| cb_listen_monitor | : refresh linkmap                                                  
+-|-----------------+                                                                    
  |    +------------------------+                                                        
  +--> | mctp_nl_handle_monitor | reset and re-fill new linkmap, compare with the old one
  |    +------------------------+                                                        
  |                                                                                      
  +--> for each 'change'                                                                 
       |                                                                                 
       |--> swtich op                                                                    
       |--> case 'add link'                                                              
       |    -    +---------------------+                                                 
       |    +--> | add_interface_local | add net/peer into ctx                           
       |         +---------------------+                                                 
       |--> case 'del link'                                                              
       |    -    +---------------+                                                       
       |    +--> | del_interface | remove peers of specified iface idx, prune unused nets
       |         +---------------+                                                       
       |--> case 'change net'                                                            
       |    |    +---------------------+                                                 
       |    |--> | add_interface_local | add net/peer into ctx                           
       |    |    +---------------------+                                                 
       |    |     +----------------------+                                               
       |    +-->  | change_net_interface | move related peers from old net to new one    
       |          +----------------------+                                               
       |--> case 'add eid'                                                               
       |    -    +---------------+                                                       
       |    +--> | add_local_eid | ensure peer exists in ctx                             
       |         +---------------+                                                       
       |--> case 'del eid'                                                               
       |    -    +---------------+                                                       
       |    +--> | del_local_eid | peer refcnt--, if 0, remove it                        
       |         +---------------+                                                       
       +--> case 'change up'                                                             
            ---> (do nothing)                                                            
```

```
src/mctp-netlink.c                                                                 
+------------------------+                                                          
| mctp_nl_handle_monitor | : reset and re-fill new linkmap, compare with the old one
+-|----------------------+                                                          
  |                                                                                 
  |--> drain the socket (continuously receive data till it's empty)                 
  |                                                                                 
  |--> reset 'nl'                                                                   
  |                                                                                 
  |    +--------------+                                                             
  |--> | fill_linkmap | set up entries and save eid                                 
  |    +--------------+                                                             
  |    +-------------------+                                                        
  |--> | fill_link_changes | compare old/new difference and fill up changes         
  |    +-------------------+                                                        
  |                                                                                 
  +--> free old link map                                                            
```

```
src/mctp-netlink.c                                                                    
+-------------------+                                                                  
| fill_link_changes | : compare old/new difference and fill up changes                 
+-|-----------------+                                                                  
  |                                                                                    
  +--> while there's still something to compare                                        
       |                                                                               
       |--> get old/new entries separately                                             
       |                                                                               
       |--> if they share the same iface                                               
       |    |                                                                          
       |    |--> if they share the same net                                            
       |    |    |                                                                     
       |    |    |    +------------------+                                             
       |    |    +--> | fill_eid_changes | compare old/new eid, save the change details
       |    |         +------------------+                                             
       |    |                                                                          
       |    |--> else ('net' changes)                                                  
       |    |    |                                                                     
       |    |    |    +------------------+                                             
       |    |    |--> | fill_eid_changes | bc of trick, here it saves all the old eid  
       |    |    |    +------------------+                                             
       |    |    |    +-------------+                                                  
       |    |    |--> | push_change | extend array, return the last one                
       |    |    |    +-------------+                                                  
       |    |    |                                                                     
       |    |    +--> specify 'op = change_net' in the last element                    
       |    |                                                                          
       |    +--> if old-up != new-up                                                   
       |         |                                                                     
       |         |    +-------------+                                                  
       |         +--> | push_change | extend array, return the last one                
       |         |    +-------------+                                                  
       |         |                                                                     
       |         +--> specify 'op = change_up' in the last element                     
       |                                                                               
       |--> if no old entry || new entry has smaller iface idx (case 'add link')       
       |    |                                                                          
       |    |    +-------------+                                                       
       |    +--> | push_change | extend array, return the last one                     
       |    |    +-------------+                                                       
       |    |                                                                          
       |    +--> specify 'op = add link' in the last element                           
       |                                                                               
       +--> if no new entry || old entry has smaller iface idx  (case 'del link')      
            |                                                                          
            |    +-------------+                                                       
            +--> | push_change | extend array, return the last one                     
            |    +-------------+                                                       
            |                                                                          
            +--> specify 'op = del link' in the last element                           
```

```
src/mctp-netlink.c                                                
+------------------+                                               
| fill_eid_changes | : compare old/new eid, save the change details
+-|----------------+                                               
  |                                                                
  |--> get old/new eid from pool separately                        
  |                                                                
  |--> if new eid is smaller (case 'add')                          
  |    |                                                           
  |    |    +-------------+                                        
  |    |--> | push_change | extend array, return the last one      
  |    |    +-------------+                                        
  |    |                                                           
  |    +--> save info                                              
  |                                                                
  +--> if old eid is smaller (case 'del')                          
       |                                                           
       |    +-------------+                                        
       |--> | push_change | extend array, return the last one      
       |    +-------------+                                        
       |                                                           
       +--> save info                                              
```

```
src/mctpd.c                                                                
+---------------------+                                                     
| add_interface_local | : add net/peer into ctx                             
+-|-------------------+                                                     
  |    +---------------------+                                              
  |--> | mctp_nl_net_byindex | given idx, get entry net                     
  |    +---------------------+                                              
  |                                                                         
  |--> if it's not in ctx yet                                               
  |    |                                                                    
  |    |    +---------+                                                     
  |    +--> | add_net | add to ctx and send dbus signal                     
  |         +---------+                                                     
  |    +-----------------------+                                            
  |--> | mctp_nl_addrs_byindex | given idx, get duplicated eid pool of entry
  |    +-----------------------+                                            
  |                                                                         
  |--> for each eid                                                         
  |    |                                                                    
  |    |    +---------------+                                               
  |    +--> | add_local_eid | ensure peer exists in ctx                     
  |         +---------------+                                               
  |                                                                         
  +--> free the duplicated eid pool                                         
```

```
src/mctpd.c                                                   
+---------------+                                              
| add_local_eid | : ensure peer exists in ctx                  
+-|-------------+                                              
  |                                                            
  |--> given net/eid, if peer exists already, ref cnt++, return
  |                                                            
  |    +----------+                                            
  |--> | add_peer | ensure peer exists in ctx, return it       
  |    +----------+                                            
  |                                                            
  |--> further set up peer                                     
  |                                                            
  |    +--------------+                                        
  +--> | publish_peer | send dbus signal of 'endpoint added'   
       +--------------+                                        
```

```
src/mctpd.c                                       
+----------+                                       
| add_peer | : ensure peer exists in ctx, return it
+-|--------+                                       
  |                                                
  |--> get net from ctx                            
  |                                                
  |--> given eid, get peer idx from net            
  |                                                
  |--> if peer idx >= 0                            
  |    |                                           
  |    |--> get peer from ctx                      
  |    |                                           
  |    +--> return this unused peer                
  |                                                
  |    (idx < 0, empty slot)                       
  |                                                
  |--> find an unused peer from ctx                
  |                                                
  |--> set up the peer from ctx                    
  |                                                
  +--> save idx in network                         
```

```
src/mctpd.c                                                                  
+---------------+                                                             
| del_interface | : remove peers of specified iface idx, prune unused nets    
+-|-------------+                                                             
  |                                                                           
  |--> for each peer in ctx                                                   
  |    -                                                                      
  |    +--> if it shares the specified iface idx                              
  |         |                                                                 
  |         |    +-------------+                                              
  |         +--> | remove_peer | send signal of 'endpoint removed', reset peer
  |              +-------------+                                              
  |    +----------------+                                                     
  +--> | prune_old_nets | remove unused nets in ctx                           
       +----------------+                                                     
```

```
src/mctpd.c                                                      
+----------------+                                                
| prune_old_nets | : remove unused nets in ctx                    
+-|--------------+                                                
  |    +------------------+                                       
  |--> | mctp_nl_net_list | get (active) net list from ctx        
  |    +------------------+                                       
  |                                                               
  +--> for each net in ctx (active + inactive)                    
       |                                                          
       |--> if it's active, adjust it's position in ctx           
       |                                                          
       +--> else                                                  
            |                                                     
            |    +------------------+                             
            +--> | emit_net_removed | send signal of 'net removed'
                 +------------------+                             
```

```
src/mctpd.c                                                                         
+----------------------+                                                             
| change_net_interface | : move related peers from old net to new one                
+-|--------------------+                                                             
  |                                                                                  
  |--> get old/new nets separately                                                   
  |                                                                                  
  |--> for each peer in ctx                                                          
  |    |                                                                             
  |    |--> if it's unrelated to new net (iface), continue                           
  |    |                                                                             
  |    |    +----------------+                                                       
  |    |--> | unpublish_peer | delete the peer's neighbor and route, send dbus signal
  |    |    +----------------+                                                       
  |    |                                                                             
  |    |--> copy peer idx from old to new, clear old                                 
  |    |                                                                             
  |    |--> save new net in peer                                                     
  |    |                                                                             
  |    |    +--------------+                                                         
  |    +--> | publish_peer | add the peer's neighbor and route, send dbus signal     
  |         +--------------+                                                         
  |    +----------------+                                                            
  +--> | prune_old_nets | remove unused nets in ctx                                  
       +----------------+                                                            
```

```
src/mctpd.c                                                               
+----------------+                                                         
| unpublish_peer | : delete the peer's neighbor and route, send dbus signal
+-|--------------+                                                         
  |                                                                        
  |--> if the peer has neighbor                                            
  |    |                                                                   
  |    |    +-------------------+                                          
  |    +--> | peer_neigh_update | send netlink packet of 'del neighbor'    
  |         +-------------------+                                          
  |                                                                        
  |--> if the peer has route                                               
  |    |                                                                   
  |    |    +-------------------+                                          
  |    +--> | peer_route_update | send netlink packet of 'del route'       
  |         +-------------------+                                          
  |                                                                        
  +--> if the peer was published                                           
       |                                                                   
       |    +-----------------------+                                      
       +--> | emit_endpoint_removed | send signal of 'endpoints removed'   
            +-----------------------+                                      
```

```
src/mctpd.c                                                    
+------------+                                                  
| setup_nets | : for each iface idx in ctx, add net/peer to ctx 
+-|----------+                                                  
  |    +-----------------+                                      
  |--> | mctp_nl_if_list | duplicate array of iface idx from ctx
  |    +-----------------+                                      
  |                                                             
  +--> for each iface idx                                       
       |                                                        
       |    +---------------------+                             
       +--> | add_interface_local | add net/peer into ctx       
            +---------------------+                             
```

```
src/mctpd.c                                                                      
+--------------------+                                                            
| listen_control_msg | : register handler for incoming request                    
+-|------------------+                                                            
  |                                                                               
  |--> prepare a socker for receiving                                             
  |                                                                               
  |    +------+                                                                   
  |--> | bind |                                                                   
  |    +------+                                                                   
  |    +-----------------+                                                        
  +--> | sd_event_add_io | register callback for incoming data                    
       +-----------------+ +-----------------------+                              
                           | cb_listen_control_msg | receive request and handle it
                           +-----------------------+                              
```

```
src/mctpd.c                                             
+-----------------------+                                
| cb_listen_control_msg | : receive request and handle it
+-|---------------------+                                
  |    +--------------+                                  
  |--> | read_message | alloc buffer, read in data       
  |    +--------------+                                  
  |                                                      
  +--> switch cmd_code                                   
       case 'get version support'                        
       -    +------------------------------------+       
       +--> | handle_control_get_version_support |       
            +------------------------------------+       
       case 'set eid'                                    
       -    +--------------------------------+           
       +--> | handle_control_set_endpoint_id |           
            +--------------------------------+           
       case 'get eid'                                    
       -    +--------------------------------+           
       +--> | handle_control_get_endpoint_id |           
            +--------------------------------+           
       case 'get ep uuid'                                
       -    +----------------------------------+         
       +--> | handle_control_get_endpoint_uuid |         
            +----------------------------------+         
       case 'get msg type support'                       
       -    +-----------------------------------------+  
       +--> | handle_control_get_message_type_support |  
            +-----------------------------------------+  
       case 'resolve eid'                                
       -    +------------------------------------+       
       +--> | handle_control_resolve_endpoint_id |       
            +------------------------------------+       
```

```
src/mctpd.c                                 
+--------------+                             
| read_message | : alloc buffer, read in data
+-|------------+                             
  |    +----------+                          
  |--> | recvfrom | (peek)                   
  |    +----------+                          
  |                                          
  |--> alloc buffer                          
  |                                          
  |    +----------+                          
  |--> | recvfrom |                          
  |    +----------+                          
  |                                          
  +--> return buf and size to caller         
```

### mctp-demux-daemon

```
utils/mctp-demux-daemon.c                                                                
+------+                                                                                  
| main |                                                                                  
+-|----+                                                                                  
  |                                                                                       
  |--> handle command options (binding_path, socket_path, verbose, local_eid)             
  |                                                                                       
  |--> prepare context buffer                                                             
  |                                                                                       
  |--> handle binding_path and socket_path (pcap related, skip)                           
  |                                                                                       
  |    +--------------------+                                                             
  |--> | mctp_set_log_stdio | set log level                                               
  |    +--------------------+                                                             
  |    +-----------+                                                                      
  |--> | mctp_init | prepare 'mctp' struct                                                
  |    +-----------+                                                                      
  |    +--------------+                                                                   
  |--> | binding_init | look up binding and call its init (prepare structs, map mmio, ...)
  |    +--------------+                                                                   
  |    +---------------+                                                                  
  |--> | sd_listen_fds |                                                                  
  |    +---------------+                                                                  
  |    +------------+                                                                     
  +--> | run_daemon | endlessly handle incoming events                                    
       +------------+                                                                     
```

```
utils/mctp-demux-daemon.c                                                              
+--------------+                                                                        
| binding_init | : look up binding and call its init (prepare structs, map mmio, ...)   
+-|------------+                                                                        
  |    +----------------+                                                               
  |--> | binding_lookup | given name, look up binding                                   
  |    +----------------+                                                               
  |                                                                                     
  +--> call ->init()                                                                    
       +---------------------+                                                          
       | binding_astlpc_init | prepare structs, map mmio, write something and set status
       +---------------------+                                                          
```

```
utils/mctp-demux-daemon.c                                                                             
+---------------------+                                                                                
| binding_astlpc_init | : prepare structs, map mmio, write something and set status                    
+-|-------------------+                                                                                
  |    +-------------------------+                                                                     
  |--> | mctp_astlpc_init_fileio | prepare ast_lpc struct, open lpc file( to map memory), open kcs file
  |    +-------------------------+                                                                     
  |    +-------------------+                                                                           
  +--> | mctp_register_bus | prepare mctp bus, determine tx/rx, build header, write and set status     
       +-------------------+                                                                           
```

```
astlpc.c                                                                                         
+-------------------------+                                                                       
| mctp_astlpc_init_fileio | : prepare ast_lpc struct, open lpc file( to map memory), open kcs file
+-|-----------------------+                                                                       
  |    +--------------------+                                                                     
  |--> | __mctp_astlpc_init | alloc and set up ast_lpc                                            
  |    +--------------------+                                                                     
  |                +-------------------------------+                                              
  |--> install ops | __mctp_astlpc_fileio_kcs_read |                                              
  |                +--------------------------------+                                             
  |                | __mctp_astlpc_fileio_kcs_write |                                             
  |                +--------------------------------+                                             
  |    +-----------------------------+                                                            
  |--> | mctp_astlpc_init_fileio_lpc | open "/dev/aspeed-lpc-ctrl" to map reserved memory         
  |    +-----------------------------+                                                            
  |    +-----------------------------+                                                            
  +--> | mctp_astlpc_init_fileio_kcs | open "/dev/mctp0"                                          
       +-----------------------------+                                                            
```

```
astlpc.c                                                                           
+-----------------------------+                                                     
| mctp_astlpc_init_fileio_lpc | : open "/dev/aspeed-lpc-ctrl" to map reserved memory
+-|---------------------------+                                                     
  |                                                                                 
  |--> open "/dev/aspeed-lpc-ctrl"                                                  
  |                                                                                 
  |--> ioctl(get_size)                                                              
  |                                                                                 
  |--> ioctl(map)                                                                   
  |                                                                                 
  |--> map reserved memory into address space                                       
  |                                                                                 
  +--> close                                                                        
```

```
core.c                                                                                          
+-------------------+                                                                            
| mctp_register_bus | : prepare mctp bus, determine tx/rx, build header, write and set status    
+-|-----------------+                                                                            
  |                                                                                              
  |--> alloc and set up mctp->busses                                                             
  |                                                                                              
  +--> if ->start() exists                                                                       
       -                                                                                         
       +--> call it, e.g.,                                                                       
            +-------------------------------+                                                    
            | mctp_binding_astlpc_start_bmc | determine tx/rx, build header, write and set status
            +-------------------------------+                                                    
```

```
core.c                                                                                
+-------------------------------+                                                      
| mctp_binding_astlpc_start_bmc | : determine tx/rx, build header, write and set status
+-|-----------------------------+                                                      
  |                                                                                    
  +--> get version 3 protocol                                                          
  |                                                                                    
  |    +----------------------+                                                        
  +--> | mctp_astlpc_init_bmc | determine tx/rx, build header, write and set status    
       +----------------------+                                                        
```

```
astlpc.c                                                                       
+-----------------------+                                                       
|  mctp_astlpc_init_bmc | : determine tx/rx, build header, write and set status 
+--|--------------------+                                                       
   |                                                                            
   |--> determine layout tx/rx attributes                                       
   |                                                                            
   |    +-----------------------------+                                         
   |--> | mctp_astlpc_layout_validate |                                         
   |    +-----------------------------+                                         
   |                                                                            
   |--> build up header                                                         
   |                                                                            
   |    +-----------------------+                                               
   |--> | mctp_astlpc_lpc_write | try indirect access or direct mapping         
   |    +-----------------------+                                               
   |    +----------------------------+                                          
   +--> | mctp_astlpc_kcs_set_status | write register through mmio to set status
        +----------------------------+                                          
```

```
utils/mctp-demux-daemon.c                                                                                  
+------------+                                                                                              
| run_daemon | : endlessly handle incoming events                                                           
+-|----------+                                                                                              
  |                                                                                                         
  |--> config poll_fd and signal                                                                            
  |                                                                                                         
  |    +-----------------+                                                                                  
  |--> | mctp_set_rx_all | install callback to rx                                                           
  |    +-----------------+ +------------+                                                                   
  |                        | rx_message | for each client registering this type, send the received msg to it
  |                        +------------+                                                                   
  +--> loop                                                                                                 
       |                                                                                                    
       |--> if client changes                                                                               
       |    -                                                                                               
       |    +--> realloc pollfds and update their content                                                   
       |                                                                                                    
       |    if binding has ->init_pollfs()                                                                  
       |    -                                                                                               
       |    +--> call it, e.g.,                                                                             
       |         +----------------------------+                                                             
       |         | binding_astlpc_init_pollfd | init poll fd                                                
       |         +----------------------------+                                                             
       |    +------+                                                                                        
       |--> | poll | wait for event                                                                         
       |    +------+                                                                                        
       |                                                                                                    
       |--> if event is about signal, read and break loop                                                   
       |                                                                                                    
       |--> if event is about binding                                                                       
       |    -                                                                                               
       |    +--> call ->process(), e.g.,                                                                    
       |         +------------------------+                                                                 
       |         | binding_astlpc_process | read cmd from data reg and process it accordingly               
       |         +------------------------+                                                                 
       |                                                                                                    
       |--> for each client                                                                                 
       |    -                                                                                               
       |    +--> if no event from it, continue                                                              
       |         |                                                                                          
       |         |    +---------------------+                                                               
       |         +--> | client_process_recv | forward the received msg (to local client or remote host)     
       |              +---------------------+                                                               
       |                                                                                                    
       +--> if event is about socket                                                                        
            |                                                                                               
            |    +----------------+                                                                         
            +--> | socket_process | accept connection, update client info in ctx                            
                 +----------------+                                                                         
```

```
utils/mctp-demux-daemon.c                                                         
+------------+                                                                     
| rx_message | : for each client registering this type, send the received msg to it
+-|----------+                                                                     
  |                                                                                
  |--> given arg msg, set up iov                                                   
  |                                                                                
  +--> for each context client                                                     
       |                                                                           
       |--> if this msg isn't its type, continue                                   
       |                                                                           
       |    +---------+                                                            
       +--> | sendmsg | send to client through socket                              
            +---------+                                                            
```

```
utils/mctp-demux-daemon.c                                                     
+------------------------+                                                     
| binding_astlpc_process | : read cmd from data reg and process it accordingly 
+-|----------------------+                                                     
  |    +------------------+                                                    
  +--> | mctp_astlpc_poll | : read cmd from data reg and process it accordingly
       +-|----------------+                                                    
         |    +----------------------+                                         
         |--> | mctp_astlpc_kcs_read | mmio-read status reg                    
         |    +----------------------+                                         
         |    +----------------------+                                         
         |--> | mctp_astlpc_kcs_read | mmio-read data reg                      
         |    +----------------------+                                         
         |                                                                     
         |--> switch data                                                      
         |--> case cmd_initialise                                              
         |    -    +--------------------------+                                
         |    +--> | mctp_astlpc_init_channel |                                
         |         +--------------------------+                                
         |--> case cmd_tx_begin                                                
         |    -    +----------------------+                                    
         |    +--> | mctp_astlpc_rx_start |                                    
         |         +----------------------+                                    
         +--> case cmd_rx_complete                                             
              -    +-------------------------+                                 
              +--> | mctp_astlpc_tx_complete |                                 
                   +-------------------------+                                 
```

```
utils/mctp-demux-daemon.c                                                           
+---------------------+                                                              
| client_process_recv | : forward the received msg (to local client or remote host)  
+-|-------------------+                                                              
  |                                                                                  
  |--> encure client has type                                                        
  |                                                                                  
  |    +------+                                                                      
  |--> | recv |                                                                      
  |    +------+                                                                      
  |                                                                                  
  |--> ensure buffer can accommodate incoming data                                   
  |                                                                                  
  |    +------+                                                                      
  |--> | recv |                                                                      
  |    +------+                                                                      
  |                                                                                  
  |--> get eid from buffer                                                           
  |                                                                                  
  |--> if it matches local eid                                                       
  |    |                                                                             
  |    |    +------------+                                                           
  |    +--> | rx_message | forward to interested clients                             
  |         +------------+                                                           
  |                                                                                  
  +--> else                                                                          
       |                                                                             
       |    +------------+                                                           
       +--> | tx_message | prepare packet(s) to accommodate msg, send out through kcs
            +------------+                                                           
```

```
utils/mctp-demux-daemon.c                                                                         
+------------+                                                                                     
| tx_message | : prepare packet(s) to accommodate msg, send out through kcs                        
+-|----------+                                                                                     
  |    +-----------------+                                                                         
  +--> | mctp_message_tx | : prepare packet(s) to accommodate msg, send out through kcs            
       +-|---------------+                                                                         
         |    +------------------+                                                                 
         |--> | find_bus_for_eid | currently it simply returns first bus                           
         |    +------------------+                                                                 
         |    +------------------------+                                                           
         +--> | mctp_message_tx_on_bus | prepare packet(s) to accommodate msg, send out through kcs
              +------------------------+                                                           
```

```
core.c                                                                                
+------------------------+                                                             
| mctp_message_tx_on_bus | : prepare packet(s) to accommodate msg, send out through kcs
+-|----------------------+                                                             
  |                                                                                    
  |--> while we haven't packed all data in msg                                         
  |    |                                                                               
  |    |    +-------------------+                                                      
  |    |--> | mctp_pktbuf_alloc | alloc packet buffer                                  
  |    |    +-------------------+                                                      
  |    |                                                                               
  |    |--> build header                                                               
  |    |                                                                               
  |    |--> copy msg to buffer                                                         
  |    |                                                                               
  |    +--> add packet to list in bus                                                  
  |                                                                                    
  |    +--------------------+                                                          
  +--> | mctp_send_tx_queue | send out all packets in bus through kcs                  
       +--------------------+                                                          
```

```
core.c                                                         
+--------------------+                                          
| mctp_send_tx_queue | : send out all packets in bus through kcs
+-|------------------+                                          
  |                                                             
  +--> while bus still has packet in list                       
       |                                                        
       |    +----------------+                                  
       +--> | mctp_packet_tx | send data out through kcs        
            +----------------+                                  
```

```
core.c                                                    
+----------------+                                         
| mctp_packet_tx | : send data out through kcs             
+-|--------------+                                         
  |                                                        
  +--> call ->tx(), e.g.,                                  
       +------------------------+                          
       | mctp_binding_astlpc_tx | send data out through kcs
       +------------------------+                          
```

### mctp_astpcie_register_type (intel openbmc libmctp)

```
utils/mctp-astpcie-register-type.c                                                                      
+------+                                                                                                 
| main |                                                                                                 
+-|----+                                                                                                 
  |    +-----------+                                                                                     
  |--> | mctp_init | prepare 'mctp' struct                                                               
  |    +-----------+                                                                                     
  |    +-------------------+                                                                             
  |--> | mctp_astpcie_init | alloc ast_pcie and set up (install ops)
  |    +-------------------+                                                                             
  |    +-------------------+                                                                             
  |--> | mctp_astpcie_core | get binding                                                                 
  |    +-------------------+                                                                             
  |    +-------------------------------+                                                                 
  |--> | mctp_register_bus_dynamic_eid | set up bus, link 'mctp' and 'binding', call binding->start (open file, ...)
  |    +-------------------------------+                                                                 
  |    +-----------------+                                                                               
  |--> | mctp_set_rx_all | install rx function for msg
  |    +-----------------+ +------------+                                                                
  |                        | rx_message | given cmd from response, handle it (e.g., print log)
  |                        +------------+                                                                
  |                                                                                                      
  |    +------------------+                                                                              
  |--> | mctp_set_rx_ctrl | install rx function for ctrl
  |    +------------------+ +--------------------+                                                       
  |                         | rx_control_message | given cmd, prepare response and send out
  |                         +--------------------+                                                       
  |    +------------------------------------+                                                            
  |--> | mctp_astpcie_register_type_handler | set up params (mctp_ctrl) and ioctl (register_type_handler)
  |    +------------------------------------+                                                            
  |    +------------------+                                                                              
  |--> | wait_for_message | wait for msg arrival                                                         
  |    +------------------+                                                                              
  |    +--------------------------------------+                                                          
  |--> | mctp_astpcie_unregister_type_handler | set up params and ioctl (unregister_type_handler)        
  |    +--------------------------------------+                                                          
  |    +------------------------------------+                                                            
  |--> | mctp_astpcie_register_type_handler | set up params (vdpci) and ioctl (register_type_handler)    
  |    +------------------------------------+                                                            
  |    +------------------+                                                                              
  |--> | wait_for_message | wait for msg arrival                                                         
  |    +------------------+                                                                              
  |    +--------------------------------------+                                                          
  |--> | mctp_astpcie_unregister_type_handler | set up params and ioctl (unregister_type_handler)        
  |    +--------------------------------------+                                                          
  |    +-------------------+                                                                             
  |--> | mctp_astpcie_free | close file and free ast_pcie                                                
  |    +-------------------+                                                                             
  |    +--------------+                                                                                  
  +--> | mctp_destroy | free mctp and related struct                                                     
       +--------------+                                                                                  
```

### mctp_astpcie_discovery (intel openbmc libmctp)

```
utils/mctp-astpcie-discovery.c
+------+
| main | : init struct (ops), set up bus/binding, ioctl (default handler), install rx, send packet (discovery)
+-|----+
  |    +-----------+
  |--> | mctp_init | prepare 'mctp' struct
  |    +-----------+
  |    +-------------------+
  |--> | mctp_astpcie_init | alloc ast_pcie and set up (install ops)
  |    +-------------------+
  |    +-------------------------------+
  |--> | mctp_register_bus_dynamic_eid | set up bus, link 'mctp' and 'binding', call binding->start (open file, ...)
  |    +-------------------------------+
  |    +---------------------------------------+
  |--> | mctp_astpcie_register_default_handler | ioctl (register default handler)
  |    +---------------------------------------+
  |    +-----------------+
  |--> | mctp_set_rx_all | install rx function for msg
  |    +-----------------+ +------------+
  |                        | rx_message | given cmd from response, handle it (e.g., print log)
  |                        +------------+
  |
  |    +------------------+
  |--> | mctp_set_rx_ctrl | install rx function for ctrl
  |    +------------------+ +--------------------+
  |                         | rx_control_message | given cmd, prepare response and send out
  |                         +--------------------+
  |    +----------------------------+
  +--> | discovery_with_notify_flow | build packet (cmd = discovery_notify) and send out
       +----------------------------+
```

```
astpcie.c                                                                                
+-------------------+                                                                     
| mctp_astpcie_init | : alloc ast_pcie and set up (install ops)                           
+-|-----------------+                                                                     
  |                                                                                       
  |--> alloc ast_pcie and set up its binding                                              
  |                                                                                       
  +--> install ops                                                                        
           +-----------------+                                                            
           | mctp_astpcie_tx | build header, write packet to fd (send out)                
           +-----------------+                                                            
           +--------------------+                                                         
           | mctp_astpcie_start | open "/dev/aspeed-mctp" and save fd, get bdf & medium_id
           +--------------------+                                                         
```

```
astpcie.c                                                                       
+--------------------+                                                           
| mctp_astpcie_start | : open "/dev/aspeed-mctp" and save fd, get bdf & medium_id
+-|------------------+                                                           
  |    +-------------------+                                                     
  |--> | mctp_astpcie_open | open "/dev/aspeed-mctp" and save fd                 
  |    +-------------------+                                                     
  |    +----------------------------+                                            
  |--> | mctp_astpcie_get_bdf_ioctl | ioctl (get_bdf)                            
  |    +----------------------------+                                            
  |    +----------------------------------+                                      
  +--> | mctp_astpcie_get_medium_id_ioctl | ioctl (get_medium_id)                
       +----------------------------------+                                      
```

```
utils/mctp-astpcie-discovery.c                                                                                     
+--------------------+                                                                                              
| rx_control_message | : given cmd, prepare response and send out                                                   
+-|------------------+                                                                                              
  |                                                                                                                 
  |--> get cmd code from request                                                                                    
  |                                                                                                                 
  |--> switch (cmd)                                                                                                 
  |--> case prepare_ep_discovery                                                                                    
  |    -    +----------------------------------+                                                                    
  |    +--> | discovery_prepare_broadcast_resp | set completion in response                                         
  |         +----------------------------------+                                                                    
  |--> case ep_discovery                                                                                            
  |    -    +----------------------------------+                                                                    
  |    +--> | discovery_prepare_broadcast_resp | set completion in response                                         
  |         +----------------------------------+                                                                    
  |--> case get_ep_id                                                                                               
  |    -    +----------------------------------------+                                                              
  |    +--> | discovery_prepare_get_endpoint_id_resp | set eid/type/completion in response                          
  |         +----------------------------------------+                                                              
  |--> case set_ep_id                                                                                               
  |    -    +----------------------------------------+                                                              
  |    +--> | discovery_prepare_set_endpoint_id_resp | save eid from request in bus, set completion code in response
  |         +----------------------------------------+                                                              
  |    +-----------------------------+                                                                              
  |--> | mctp_binding_set_tx_enabled | enable tx, if there's pending packets: send out through binding, e.g., pcie  
  |    +-----------------------------+                                                                              
  |    +-----------------+                                                                                          
  +--> | mctp_message_tx | prepare packet(s) to accommodate msg, send out through binding (e.g., pcie)              
       +-----------------+                                                                                          
```

```
utils/mctp-astpcie-discovery.c                                                                           
+----------------------------------------+                                                                
| discovery_prepare_set_endpoint_id_resp | : save eid from request in bus, set completion code in response
+-|--------------------------------------+                                                                
  |    +-------------------------------+                                                                  
  +--> | mctp_ctrl_cmd_set_endpoint_id | : save eid from request in bus, set completion code in response  
       +-|-----------------------------+                                                                  
         |                                                                                                
         |--> switch request_op                                                                           
         |                                                                                                
         |--> case set_eid                                                                                
         |    -                                                                                           
         |    +--> save eid from request in bus                                                           
         |                                                                                                
         +--> case force_eid                                                                              
              -                                                                                           
              +--> save eid from request in bus                                                           
```

```
utils/mctp-astpcie-discovery.c                                                                                   
+----------------------------+                                                                                    
| discovery_with_notify_flow | : build packet (cmd = discovery_notify) and send out                               
+-|--------------------------+                                                                                    
  |                                                                                                               
  |--> set up request and packet (cmd = discovery_notify)                                                         
  |                                                                                                               
  |    +-----------------------------+                                                                            
  |--> | mctp_binding_set_tx_enabled | enable tx, if there's pending packets: send out through binding, e.g., pcie
  |    +-----------------------------+                                                                            
  |    +-----------------+                                                                                        
  |--> | mctp_message_tx | prepare packet(s) to accommodate msg, send out through binding (e.g., pcie)            
  |    +-----------------+                                                                                        
  |    +------------------------+                                                                                 
  +--> | discovery_regular_flow | poll, read data and build packet, handle or forward it                          
       +------------------------+                                                                                 
```

```
utils/mctp-astpcie-discovery.c                                                                                   
+----------------------------+                                                                                    
| discovery_with_notify_flow | : build packet (cmd = discovery_notify) and send out                               
+-|--------------------------+                                                                                    
  |                                                                                                               
  |--> set up request and packet (cmd = discovery_notify)                                                         
  |                                                                                                               
  |    +-----------------------------+                                                                            
  |--> | mctp_binding_set_tx_enabled | enable tx, if there's pending packets: send out through binding, e.g., pcie
  |    +-----------------------------+                                                                            
  |    +-----------------+                                                                                        
  |--> | mctp_message_tx | prepare packet(s) to accommodate msg, send out through binding (e.g., pcie)            
  |    +-----------------+                                                                                        
  |    +------------------------+                                                                                 
  +--> | discovery_regular_flow | poll, read data and build packet, handle or forward it                          
       +------------------------+                                                                                 
```

```
utils/mctp-astpcie-discovery.c                                                                           
+------------------------+                                                                                
| discovery_regular_flow | : poll, read data and build packet, handle or forward it                       
+-|----------------------+                                                                                
  |                                                                                                       
  +--> while it's not yet discovered                                                                      
       |                                                                                                  
       |    +-------------------+                                                                         
       |--> | mctp_astpcie_poll | poll fd                                                                 
       |    +-------------------+                                                                         
       |                                                                                                  
       +--> if poll_in is set                                                                             
            |                                                                                             
            |    +-----------------+                                                                      
            +--> | mctp_astpcie_rx | read data and build packet, handle it locally or forward to elsewhere
                 +-----------------+                                                                      
```

```
astpcie.c                                                                                 
+-----------------+                                                                        
| mctp_astpcie_rx | : read data and build packet, handle it locally or forward to elsewhere
+-|---------------+                                                                        
  |                                                                                        
  |--> read data from fd                                                                   
  |                                                                                        
  |    +-----------------------------------+                                               
  |--> | mctp_astpcie_is_routing_supported | check ifi routing is supported                
  |    +-----------------------------------+                                               
  |    +-------------------+                                                               
  |--> | mctp_pktbuf_alloc | alloc packet buffer                                           
  |    +-------------------+                                                               
  |    +------------------+                                                                
  |--> | mctp_pktbuf_push | push (copy) data to proper position in packet buffer           
  |    +------------------+                                                                
  |    +-------------+                                                                     
  +--> | mctp_bus_rx | parse som/eom, receive packet accordingly                           
       +-------------+                                                                     
```

```
core.c                                                                              
+-------------+                                                                      
| mctp_bus_rx | : parse som/eom, receive packet accordingly                          
+-|-----------+                                                                      
  |                                                                                  
  |--> get header from packet                                                        
  |                                                                                  
  |--> get eom/som flags                                                             
  |                                                                                  
  |--> switch flags                                                                  
  |--> case eom and som                                                              
  |    -    +---------+                                                              
  |    +--> | mctp_rx | handle packet locally or forward to other bus                
  |         +---------+                                                              
  |--> case som                                                                      
  |    |    +---------------------+                                                  
  |    |--> | mctp_msg_ctx_lookup | given src/dst/..., look up context               
  |    |    +---------------------+                                                  
  |    |--> if found, reset it, otherwise create a new context                       
  |    |    +----------------------+                                                 
  |    +--> | mctp_msg_ctx_add_pkt | create buffer for context, copy data from packet
  |         +----------------------+                                                 
  |--> case eom                                                                      
  |    -    +---------------------+                                                  
  |    +--> | mctp_msg_ctx_lookup | given src/dst/..., look up context               
  |    |    +---------------------+                                                  
  |    |    +----------------------+                                                 
  |    +--> | mctp_msg_ctx_add_pkt | create buffer for context, copy data from packet
  |         +----------------------+                                                 
  +--> case not som and not eom                                                      
       |    +---------------------+                                                  
       |--> | mctp_msg_ctx_lookup | given src/dst/..., look up context               
       |    +---------------------+                                                  
       |--> update seq#                                                              
       |    +----------------------+                                                 
       +--> | mctp_msg_ctx_add_pkt | create buffer for context, copy data from packet
            +----------------------+                                                 
```

```
core.c                                                                                                       
+---------+                                                                                                   
| mctp_rx | : handle packet locally or forward to other bus                                                   
+-|-------+                                                                                                   
  |                                                                                                           
  |--> if policy is 'route_endpoint'                                                                          
  |    |                                                                                                      
  |    |--> if it's a control request                                                                         
  |    |    |                                                                                                 
  |    |    |    +----------------------+                                                                     
  |    |    +--> | mctp_ctrl_handle_msg | call binding->control_rx() or mctp->control_rx()                    
  |    |         +----------------------+                                                                     
  |    |                                                                                                      
  |    +--> if ->message_rx() exists                                                                          
  |         -                                                                                                 
  |         +--> call it, e.g.,                                                                               
  |              +------------+                                                                               
  |              | rx_message | given cmd from response, handle it (e.g., print log)                          
  |              +------------+                                                                               
  |                                                                                                           
  +--> if policy is 'route_bridge'                                                                            
       -                                                                                                      
       +--> for each bus other than the argument one                                                          
            -    +------------------------+                                                                   
            +--> | mctp_message_tx_on_bus | prepare packet(s) to accommodate msg, send out through, e.g., pcie
                 +------------------------+                                                                   
```

### linux driver

```
drivers/soc/aspeed/aspeed-mctp.c                                                   
+-------------------+                                                               
| aspeed_mctp_probe |                                                               
+-|-----------------+                                                               
  |                                                                                 
  |--> alloc priv                                                                   
  |                                                                                 
  |--> read dt properties (e.g., "pcie_rc", ...)                                    
  |                                                                                 
  |    +----------------------+                                                     
  |--> | aspeed_mctp_drv_init | : init priv struct/work/tasklets                    
  |    +----------------------+                                                     
  |    +----------------------------+                                               
  |--> | aspeed_mctp_resources_init | map iomem                                     
  |    +----------------------------+                                               
  |    +---------------------------+                                                
  |--> | aspeed_mctp_channels_init | init rx/tx channels                            
  |    +---------------------------+                                                
  |    +---------------+                                                            
  |--> | misc_register | determine dev#, add arg 'misc' to list (misc_list)         
  |    +---------------+                                                            
  |    +----------------------+                                                     
  |--> | aspeed_mctp_irq_init | register two isr for mctp/pcie interrupts separately
  |    +----------------------+                                                     
  |    +------------------------+                                                   
  |--> | aspeed_mctp_pcie_setup | reset, flush tx queues, enable irq, trigger rx    
  |    +------------------------+                                                   
  |    +------------------------+                                                   
  +--> | aspeed_mctp_rx_trigger | hw-level trigger rx                               
       +------------------------+                                                   
```

```
drivers/soc/aspeed/aspeed-mctp.c                                                             
+----------------------+                                                                      
| aspeed_mctp_drv_init | : init priv struct/work/tasklets                                     
+-|--------------------+                                                                      
  |    +-------------------+ +------------------------+                                       
  |--> | INIT_DELAYED_WORK | | aspeed_mctp_reset_work |                                       
  |    +-------------------+ +------------------------+                                       
  |                          reset, flush tx queues, enable irq, trigger rx                   
  |                                                                                           
  |    +--------------+ +------------------------+                                            
  |--> | tasklet_init | | aspeed_mctp_tx_tasklet |                                            
  |    +--------------+ +------------------------+                                            
  |                     set up tx cmd for each client packets, trigger hardware               
  |                                                                                           
  |    +--------------+ +------------------------+                                            
  +--> | tasklet_init | | aspeed_mctp_rx_tasklet |                                            
       +--------------+ +------------------------+                                            
                        for each valid header, prepare rx_packet and dispatch to target client
```

```
drivers/soc/aspeed/aspeed-mctp.c                                          
+------------------------+                                                 
| aspeed_mctp_reset_work | : reset, flush tx queues, enable irq, trigger rx
+-|----------------------+                                                 
  |    +------------------------+                                          
  |--> | aspeed_mctp_pcie_setup | scheudle a reset if necessary            
  |    +------------------------+                                          
  |                                                                        
  +--> if (bus, dev, func) tuple isn't 0                                   
       |                                                                   
       |    +---------------------------------+                            
       |--> | aspeed_mctp_flush_all_tx_queues |                            
       |    +---------------------------------+                            
       |    +------------------------+                                     
       |--> | aspeed_mctp_irq_enable | hw-level enable                     
       |    +------------------------+                                     
       |     +------------------------+                                    
       +-->  | aspeed_mctp_rx_trigger |                                    
             +------------------------+                                    
```

```
drivers/soc/aspeed/aspeed-mctp.c                                                   
+------------------------+                                                          
| aspeed_mctp_tx_tasklet | : set up tx cmd for each client packets, trigger hardware
+-|----------------------+                                                          
  |                                                                                 
  |--> for each client on priv list                                                 
  |    -                                                                            
  |    +--> while tx isn't full                                                     
  |         |                                                                       
  |         |    +------------------+                                               
  |         |--> | ptr_ring_consume | get next packet to send                       
  |         |    +------------------+                                               
  |         |    +-------------------------+                                        
  |         |--> | aspeed_mctp_emit_tx_cmd | set up tx command and update write ptr 
  |         |    +-------------------------+                                        
  |         |    +-------------------------+                                        
  |         +--> | aspeed_mctp_packet_free | free packet to kmem                    
  |              +-------------------------+                                        
  |                                                                                 
  +--> if we've set up any tx command                                               
       |                                                                            
       |    +------------------------+                                              
       +--> | aspeed_mctp_tx_trigger | hw-level trigger tx                          
            +------------------------+                                              
```

```
drivers/soc/aspeed/aspeed-mctp.c                                                                  
+------------------------+                                                                         
| aspeed_mctp_rx_tasklet | : for each valid header, prepare rx_packet and dispatch to target client
+-|----------------------+                                                                         
  |                                                                                                
  |--> if vdm_header_direct_transfer                                                               
  |    |                                                                                           
  |    |--> trigger hw read-ptr update                                                             
  |    |                                                                                           
  |    |--> maintain our own state of buffer                                                       
  |    |                                                                                           
  |    +--> while the header is valid                                                              
  |         |                                                                                      
  |         |--> alloc rx packet                                                                   
  |         |                                                                                      
  |         |    +-----------------------------+                                                   
  |         |--> | aspeed_mctp_dispatch_packet | dispatch packet to target client                  
  |         |    +-----------------------------+                                                   
  |         |                                                                                      
  |         +--> get next header                                                                   
  |                                                                                                
  |--> else                                                                                        
  |    -                                                                                           
  |    +--> while the header is valid                                                              
  |         |                                                                                      
  |         |--> alloc rx packet                                                                   
  |         |                                                                                      
  |         |    +-----------------------------+                                                   
  |         |--> | aspeed_mctp_dispatch_packet | dispatch packet to target client                  
  |         |    +-----------------------------+                                                   
  |         |                                                                                      
  |         +--> get next header                                                                   
  |                                                                                                
  +--> trigger rx if it stops (when it was full)                                                   
```

```
drivers/soc/aspeed/aspeed-mctp.c                                            
+-----------------------------+                                              
| aspeed_mctp_dispatch_packet | : dispatch packet to target client           
+-|---------------------------+                                              
  |    +--------------------------+                                          
  |--> | aspeed_mctp_find_handler | find a (mctp, vendor, vdm) matched client
  |    +--------------------------+                                          
  |                                                                          
  +--> if client found                                                       
       |                                                                     
       |    +------------------+                                             
       |--> | ptr_ring_produce | let pointer point to valid data?            
       |    +------------------+                                             
       |    +-------------+                                                  
       +--> | wake_up_all | wakt up waiting clients                          
            +-------------+                                                  
```

```
drivers/soc/aspeed/aspeed-mctp.c                                       
+--------------------------+                                            
| aspeed_mctp_find_handler | : find a (mctp, vendor, vdm) matched client
+-|------------------------+                                            
  |                                                                     
  |--> determine mctp/vendor/vdm of our target                          
  |                                                                     
  +--> for each registered type handler (e.g., peci)                    
       -                                                                
       +--> if (mctp, vendor, vdm) of handler match our specification   
            -                                                           
            +--> return found client                                    
```

```
drivers/soc/aspeed/aspeed-mctp.c                           
+----------------------------+                              
| aspeed_mctp_resources_init | : map iomem                  
+-|--------------------------+                              
  |    +--------------------------------+                   
  |--> | devm_platform_ioremap_resource | get iomem resource
  |    +--------------------------------+                   
  |    +-----------------------+                            
  |--> | devm_regmap_init_mmio | map iomem                  
  |    +-----------------------+                            
  |    +---------------------------------+                  
  +--> | syscon_regmap_lookup_by_phandle | get syscon iomem 
       +---------------------------------+                  
```

```
drivers/soc/aspeed/aspeed-mctp.c                                                                                                        
+----------------------+                                                                                                                 
| aspeed_mctp_irq_init | : register two isr for mctp/pcie interrupts separately                                                          
+-|--------------------+                                                                                                                 
  |                                                                                                                                      
  |--> get irq# of 'mctp'                                                                                                                
  |                                                                                                                                      
  |    +------------------+                                                                                                              
  |--> | devm_request_irq | prepare 'action' (handler, thread_fn, ...) and install to irq desc                                           
  |    +------------------+ +-------------------------+                                                                                  
  |                         | aspeed_mctp_irq_handler | handle tx/rx interrupts                                                          
  |                         +-------------------------+                                                                                  
  |                                                                                                                                      
  |    +------------------------+                                                                                                        
  |--> | aspeed_mctp_irq_enable | hw-level enable                                                                                        
  |    +------------------------+                                                                                                        
  |                                                                                                                                      
  |--> get irq# of 'pcie'                                                                                                                
  |                                                                                                                                      
  |    +------------------+                                                                                                              
  +--> | devm_request_irq | prepare 'action' (handler, thread_fn, ...) and install to irq desc                                           
       +------------------+ +----------------------------------+                                                                         
                            | aspeed_mctp_pcie_rst_irq_handler | init rx/tx channels, reset mctp, flush tx queues, enable irq, trigger rx
                            +----------------------------------+                                                                         
```

```
drivers/soc/aspeed/aspeed-mctp.c                                                                         
+-------------------------+                                                                               
| aspeed_mctp_irq_handler | : handle tx/rx interrupts                                                     
+-|-----------------------+                                                                               
  |                                                                                                       
  |--> read interrupt status                                                                              
  |                                                                                                       
  |--> if it's tx_sent                                                                                    
  |    |                                                                                                  
  |    |    +---------------------+ +------------------------+                                            
  |    +--> | tasklet_hi_schedule | | aspeed_mctp_tx_tasklet |                                            
  |         +---------------------+ +------------------------+                                            
  |                                 set up tx cmd for each client packets, trigger hardware               
  +--> if it's rx_receive                                                                                 
       |                                                                                                  
       |    +---------------------+ +------------------------+                                            
       +--> | tasklet_hi_schedule | | aspeed_mctp_rx_tasklet |                                            
            +---------------------+ +------------------------+                                            
                                    for each valid header, prepare rx_packet and dispatch to target client
```

```
drivers/soc/aspeed/aspeed-mctp.c
static const struct file_operations aspeed_mctp_fops = {
    .owner = THIS_MODULE,
    .open = aspeed_mctp_open,
    .release = aspeed_mctp_release,
    .read = aspeed_mctp_read,
    .write = aspeed_mctp_write,
    .unlocked_ioctl = aspeed_mctp_ioctl,
    .poll = aspeed_mctp_poll,
};
```

```
drivers/soc/aspeed/aspeed-mctp.c                                                                      
+------------------+                                                                                   
| aspeed_mctp_open | : prepare client and add to mctp_priv                                             
+-|----------------+                                                                                   
  |    +---------------------------+                                                                   
  +--> | aspeed_mctp_create_client | prepare client (init its tx/rx queues), add to list in 'mctp_priv'
  |    +---------------------------+                                                                   
  |                                                                                                    
  +--> save client in file                                                                             
```

```
drivers/soc/aspeed/aspeed-mctp.c                     
+------------------+                                  
| aspeed_mctp_read |                                  
+-|----------------+                                  
  |                                                   
  |--> consume client rx_queue                        
  |                                                   
  |    +---------------------+                        
  |--> | ptr_ring_consume_bh | consume rx queue       
  |    +---------------------+                        
  |    +--------------+                               
  +--> | copy_to_user | copy packet data to user space
       +--------------+                               
```

```
drivers/soc/aspeed/aspeed-mctp.c                                                             
+-------------------+                                                                         
| aspeed_mctp_write | : prepare tx packet, add to tx queue, trigger hw to send out            
+-|-----------------+                                                                         
  |    +--------------------------+                                                           
  |--> | aspeed_mctp_packet_alloc | alloc tx packet                                           
  |    +--------------------------+                                                           
  |    +----------------+                                                                     
  |--> | copy_from_user | copy packet data from user                                          
  |    +----------------+                                                                     
  |    +-------------------------+                                                            
  +--> | aspeed_mctp_send_packet | fill header, add packet to tx queue, trigger hw to send out
       +-------------------------+                                                            
```

```
drivers/soc/aspeed/aspeed-mctp.c                                                        
+-------------------------+                                                              
| aspeed_mctp_send_packet | : fill header, add packet to tx queue, trigger hw to send out
+-|-----------------------+                                                              
  |                                                                                      
  |--> fill (bus, dev, func) into header                                                 
  |                                                                                      
  |--> fill eid (except the case of ep discovery                                         
  |                                                                                      
  |    +---------------------+                                                           
  |--> | ptr_ring_produce_bh | add packet to tx queue                                    
  |    +---------------------+                                                           
  |    +---------------------+ +------------------------+                                
  +--> | tasklet_hi_schedule | | aspeed_mctp_tx_tasklet |                                
       +---------------------+ +------------------------+                                
                               set up tx cmd for each client packets, trigger hardware   
```

```
drivers/soc/aspeed/aspeed-mctp.c                                                                           
+-------------------+                                                                                       
| aspeed_mctp_ioctl | : mctp ioctl                                                                          
+-|-----------------+                                                                                       
  |--> switch cmd                                                                                           
  |--> case filter_eid                                                                                      
  |    -    +------------------------+                                                                      
  |    +--> | aspeed_mctp_filter_eid | copy data from user, write eid to hw reg                             
  |         +------------------------+                                                                      
  |--> case get_bdf                                                                                         
  |    -    +---------------------+                                                                         
  |    +--> | aspeed_mctp_get_bdf | assemble bdf from hw reg, copy to user                                  
  |         +---------------------+                                                                         
  |--> case get_medium_id                                                                                   
  |    -    +---------------------------+                                                                   
  |    +--> | aspeed_mctp_get_medium_id | it's fixed 0x09, copy to user                                     
  |         +---------------------------+                                                                   
  |--> case get_mtu                                                                                         
  |    -    +---------------------+                                                                         
  |    +--> | aspeed_mctp_get_mtu | it's fixed 64, copy to user                                             
  |         +---------------------+                                                                         
  |--> case register_default_handler                                                                        
  |    -    +--------------------------------------+                                                        
  |    +--> | aspeed_mctp_register_default_handler | register current client as mctp_priv's default handler 
  |         +--------------------------------------+                                                        
  |--> case register_type_handler                                                                           
  |    -    +-----------------------------------+                                                           
  |    +--> | aspeed_mctp_register_type_handler | copy from user, set up new handler and add to list in priv
  |         +-----------------------------------+                                                           
  |--> case unregister_type_handler                                                                         
  |    -    +-------------------------------------+                                                         
  |    +--> | aspeed_mctp_unregister_type_handler | copy from user, remove handler from list in priv        
  |         +-------------------------------------+                                                         
  |--> case get_eid_info or get_eid_ext_info                                                                
  |    -    +--------------------------+                                                                    
  |    +--> | aspeed_mctp_get_eid_info | copy generic or extended eid info to user                          
  |         +--------------------------+                                                                    
  |--> case set_eid_info or set_eid_ext_info                                                                
  |    -    +--------------------------+                                                                    
  |    +--> | aspeed_mctp_set_eid_info | given user data, build up new ep list and replace the old one      
  |         +--------------------------+                                                                    
  +--> case set_own_eid                                                                                     
       -    +-------------------------+                                                                     
       +--> | aspeed_mctp_set_own_eid | copy from user, save eid in priv                                    
            +-------------------------+                                                                     
```

```
drivers/soc/aspeed/aspeed-mctp.c                                                                 
+-----------------------------------+                                                             
| aspeed_mctp_register_type_handler | : copy from user, set up new handler and add to list in priv
+-|---------------------------------+                                                             
  |    +----------------+                                                                         
  |--> | copy_from_user |                                                                         
  |    +----------------+                                                                         
  |    +------------------------------+                                                           
  +--> | aspeed_mctp_add_type_handler | prepare new handler and add to list in priv               
       +------------------------------+                                                           
```

```
drivers/soc/aspeed/aspeed-mctp.c                                             
+------------------------------+                                              
| aspeed_mctp_add_type_handler | : prepare new handler and add to list in priv
+-|----------------------------+                                              
  |                                                                           
  |--> alloc and set up new handler                                           
  |                                                                           
  |--> for each type_handler in priv                                          
  |    -                                                                      
  |    +--> if there's other client occupies the (mctp, vendor, vdm)          
  |                                                                           
  +--> add new handler to the end of list                                     
```

```
drivers/soc/aspeed/aspeed-mctp.c                   
+--------------------------+                        
| aspeed_mctp_get_eid_info | : copy eid info to user
+-|------------------------+                        
  |    +----------------+                           
  |--> | copy_from_user | copy meta data from user  
  |    +----------------+                           
  |                                                 
  |--> for each endpoint on list in priv            
  |    |                                            
  |    |    +--------------+                        
  |    +--> | copy_to_user | copy ep data to user   
  |         +--------------+                        
  |                                                 
  |    +--------------+                             
  +--> | copy_to_user | copy meta data to user      
       +--------------+                             
```

```
drivers/soc/aspeed/aspeed-mctp.c                                                           
+--------------------------+                                                                
| aspeed_mctp_set_eid_info | : given user data, build up new ep list and replace the old one
+-|------------------------+                                                                
  |    +----------------+                                                                   
  |--> | copy_from_user | copy meta data from user                                          
  |    +----------------+                                                                   
  |                                                                                         
  |--> for each eid                                                                         
  |    |                                                                                    
  |    |--> alloc endpoint                                                                  
  |    |                                                                                    
  |    |    +----------------+                                                              
  |    |--> | copy_from_user | copy eid data from user                                      
  |    |    +----------------+                                                              
  |    |                                                                                    
  |    +--> add to end of local list                                                        
  |                                                                                         
  |    +-----------+                                                                        
  |--> | list_sort |                                                                        
  |    +-----------+                                                                        
  |    +---------------------------------+                                                  
  |--> | aspeed_mctp_eid_info_list_valid | check if every ep on local list is valid         
  |    +---------------------------------+                                                  
  |                                                                                         
  |--> swap local list with priv's list                                                     
  |                                                                                         
  |    +----------------------------------+                                                 
  +--> | aspeed_mctp_eid_info_list_remove | free each old ep                                
       +----------------------------------+                                                 
```

```
drivers/soc/aspeed/aspeed-mctp.c                        
+------------------+                                     
| aspeed_mctp_poll | : poll wait, label poll_in/out flags
+-|----------------+                                     
  |    +-----------+                                     
  |--> | poll_wait |                                     
  |    +-----------+                                     
  |                                                      
  |--> if queue isn't full, label 'epoll_out'            
  |                                                      
  +--> if there's data, label 'epoll_in'                 
```

### Kernel Network Driver

```
net/mctp/device.c                                                                                        
+------------------+                                                                                      
| mctp_device_init | : register callbacks for 'get addr', 'new addr', 'del addr', register af ops         
+-|----------------+                                                                                      
  |    +-----------------------------+                                                                    
  |--> | register_netdevice_notifier | register notifier                                                  
  |    +-----------------------------+                                                                    
  |    +----------------------+                                                                           
  |--> | rtnl_register_module | register dump callback for 'get addr'                                     
  |    +----------------------+ +--------------------+                                                    
  |                             | mctp_dump_addrinfo | find iface idx matched mctp dev, dump info into skb
  |                             +--------------------+                                                    
  |    +----------------------+                                                                           
  |--> | rtnl_register_module | register do callback for 'new addr'                                       
  |    +----------------------+ +------------------+                                                      
  |                             | mctp_rtm_newaddr | add new addr and routing info                        
  |                             +------------------+                                                      
  |    +----------------------+                                                                           
  |--> | rtnl_register_module | register do callback for 'del addr'                                       
  |    +----------------------+ +------------------+                                                      
  |                             | mctp_rtm_deladdr | remove routing info and addr                         
  |                             +------------------+                                                      
  |                                                                                                       
  |    +------------------+                                                                               
  +--> | rtnl_af_register | register 'mctp_af_ops' to 'rtnl_af_ops'                                       
       +------------------+                                                                               
```

```
net/mctp/device.c                                            
+------------------+                                          
| mctp_rtm_newaddr | : add new addr and routing info          
+-|----------------+                                          
  |                                                           
  |--> get iface idx from arg data, get it net_dev accordingly
  |                                                           
  |    +-------------------+                                  
  |--> | mctp_dev_get_rtnl | get mctp_dev from net_dev        
  |    +-------------------+                                  
  |                                                           
  |--> add new addr into mctp dev                             
  |                                                           
  |    +------------------+                                   
  |--> | mctp_addr_notify | prepare socket to notify the event
  |    +------------------+                                   
  |    +----------------------+                               
  +--> | mctp_route_add_local | add routing info              
       +----------------------+                               
```

```
net/mctp/device.c                                                                
+--------------------+                                                            
| mctp_dump_addrinfo | : find iface idx matched mctp dev, dump info into skb      
+-|------------------+                                                            
  |                                                                               
  +--> for each hlist head                                                        
       -                                                                          
       +--> for each entry on hlist                                               
            -                                                                     
            +--> if iface idx matches                                             
                 |                                                                
                 |    +------------------------+                                  
                 +--> | mctp_dump_dev_addrinfo | for each addr, fill info into skb
                      +------------------------+                                  
```

```
src/main.cpp                                                                
+------+                                                                     
| main |                                                                     
+-|----+                                                                     
  |    +------------------+                                                  
  |--> | getConfiguration | setup 'config' from entity_manager or config_file
  |    +------------------+                                                  
  |    +----------------+                                                    
  |--> | ->request_name | request service name                               
  |    +----------------+                                                    
  |    +---------------+                                                     
  |--> | getBindingPtr | get binding (handle dbus and get routing table)     
  |    +---------------+                                                     
  |    +--------------------------------+                                    
  +--> | PCIeBinding::initializeBinding |                                    
       +--------------------------------+                                    
```

```
src/utils/Configuration.cpp                                                                                       
+------------------+                                                                                               
| getConfiguration | : setup 'config' from entity_manager or config_file                                           
+-|----------------+                                                                                               
  |    +-----------------------------------+                                                                       
  |--> | getConfigurationFromEntityManager | (skip, not sure what it's like in entity_manager, but we have config!)
  |    +-----------------------------------+                                                                       
  |                                                                                                                
  +--> if that doesn't work                                                                                        
       |                                                                                                           
       |    +--------------------------+                                                                           
       +--> | getConfigurationFromFile | parse config file, get smbus or pcie fields and setup 'config'            
            +--------------------------+ (config_path = "/usr/share/mctp/mctp_config.json")                        
```

```
src/utils/Configuration.cpp                                                                 
+--------------------------+                                                                 
| getConfigurationFromFile | : parse config file, get smbus or pcie fields and setup 'config'
+-|------------------------+                                                                 
  |    +-------------+                                                                       
  |--> | json::parse |                                                                       
  |    +-------------+                                                                       
  |                                                                                          
  |--> if arg name == 'smbios'                                                               
  |    |                                                                                     
  |    |    +-----------------------+                                                        
  |    +--> | getSMBusConfiguration | (skip smbus part for now)                              
  |         +-----------------------+                                                        
  |                                                                                          
  +--> elif arg name == 'pcie'                                                               
       |                                                                                     
       |    +----------------------+                                                         
       +--> | getPcieConfiguration | get fields, setup config and return                     
            +----------------------+                                                         
```

```
src/utils/Configuration.cpp                                                                 
+----------------------+                                                                     
| getPcieConfiguration | : get fields, setup config and return                               
+-|--------------------+                                                                     
  |                                                                                          
  |--> get fields                                                                            
  |                                                                                          
  |--> (non-bus-owner, e.g., endpoint or bridge, is expected to have "GetRoutingInterval" set
  |                                                                                          
  +--> setup config                                                                          
```

```
src/main.cpp                                                                                              
+---------------+                                                                                          
| getBindingPtr | : get binding (handle dbus and get routing table)                                        
+-|-------------+                                                                                          
  |                                                                                                        
  |--> if arg config is for smbus                                                                          
  |    -                                                                                                   
  |    +--> (skip)                                                                                         
  |                                                                                                        
  +--> elif arg config is for pcie                                                                         
       -                                                                                                   
       +--> alloc PCIeBinding                                                                              
            +--------------------------+                                                                   
            | PCIeBinding::PCIeBinding | register iface/props, get routing table and update info internally
            +--------------------------+                                                                   
```

```
                                                  PCIeBinding                 
                                     +---------------------------------------+
                                     |          ------pub------              |
                                     |         initializeBinding()           |
                                     |          ------pro------              |
                                     |  handlePrepareForEndpointDiscovery()  |
                                     |      handleEndpointDiscovery()        |
      PCIeDriver                     |        handleGetEndpointId()          |
+---------------------+              |        handleSetEndpointId()          |
|  ------pub------    |              |      handleGetVersionSupport()        |
|       init()        |              |      handleGetMsgTypeSupport()        |
|     binding()       |              |        handleGetVdmSupport()          |
|      pollRx()       |              |         deviceReadyNotify()           |
| registerAsDefault() |              |      populateDeviceProperties()       |
|      getBdf()       |              |                 hw                    |
|   getMediumId()     |              |              hwMonitor                |
|  setEndpointMap()   |              |          ------pri------              |
|  ------pri------    |              |                bdf                    |
|    streamMonitor    |              |            busOwnerBdf                |
|       *pcie         |              |           pcieInterface               |
+---------------------+              |           discoveredFlag              |
                                     |         getRoutingInterval            |
                                     |        getRoutingTableTimer           |
                                     |            routingTable               |
                                     |       endpointDiscoveryFlow()         |
                                     |        updateRoutingTable()           |
    PCIeMonitor                      |     processRoutingTableChanges()      |
+------------------+                 |       processBridgeEntries()          |
| ------pub------  |                 |         readRoutingTable()            |
|   initialize()   |                 |      getRoutingEntryPhysAddr()        |
|     observe()    |                 |       isEntryInRoutingTable()         |
| ------pri------  |                 |     isEndOfGetRoutingTableResp()      |
|   *udevContext   |                 |      isActiveEntryBehindBridge()      |
|     *udevice     |                 |           isEntryBridge()             |
|     *umonitor    |                 |          isBridgeCalled()             |
|   ueventMonitor  |                 |         allBridgesCalled()            |
+------------------+                 |       setDriverEndpointMap()          |
                                     |       getBindingPrivateData()         |
                                     |    isReceivedPrivateDataCorrect()     |
                                     |           getBindingMode()            |
                                     |        changeDiscoveredFlag()         |
                                     +---------------------------------------+
```

```
src/PCIeBinding.cpp                                                                                   
+--------------------------+                                                                           
| PCIeBinding::PCIeBinding | : register iface/props, get routing table and update info internally      
+-|------------------------+                                                                           
  |    +-----------------+                                                                             
  |--> | ->add_interface |                                                                             
  |    +-----------------+                                                                             
  |                                                                                                    
  |--> register property "bdf"                                                                         
  |                                                                                                    
  |--> determine discover_flag and register property "DiscoveredFlag"                                  
  |                                                                                                    
  |    +---------------------------------+                                                             
  +--> | getRoutingTableTimer.async_wait |                                                             
       +---------------------------------+                                                             
       | PCIeBinding::updateRoutingTable | : send request to get routing tables, update info internally
       +---------------------------------+                                                             
```

```
src/PCIeBinding.cpp                                                                                                                        
+---------------------------------+                                                                                                         
| PCIeBinding::updateRoutingTable | : send request to get routing tables, update info internally                                            
+-|-------------------------------+                                                                                                         
  |                                                                                                                                         
  |--> config getRoutingTableTimer                                                                                                          
  |                                                                                                                                         
  |--> if we haven't been discovered, return                                                                                                
  |                                                                                                                                         
  |    +--------------------+                                                                                                               
  +--> | boost::asio::spawn |                                                                                                               
       +-----------------------------------------------------------------------------------------------------------------------------------+
       | +-------------------------------+                                                                                                 |
       | | PCIeBinding::readRoutingTable | commute with bus owner to update routing table internally                                       |
       | +-------------------------------+                                                                                                 |
       |                                                                                                                                   |
       | while not all bridges are contacted                                                                                               |
       | |                                                                                                                                 |
       | |    +-----------------------------------+                                                                                        |
       | |--> | PCIeBinding::processBridgeEntries | for each not-yet-contacted-bridge, get routing table from it                           |
       | |    +-----------------------------------+                                                                                        |
       | |                                                                                                                                 |
       | +--> if tmp_routing_table != routing_table                                                                                        |
       |      |                                                                                                                            |
       |      |    +-----------------------------------+                                                                                   |
       |      |--> | PCIeBinding::setDriverEndpointMap | set arg new_table in mctp driver                                                  |
       |      |    +-----------------------------------+                                                                                   |
       |      |    +-----------------------------------------+                                                                             |
       |      |--> | PCIeBinding::processRoutingTableChanges | compare old/new routint tables, unregister and register entries accordingly |
       |      |    +-----------------------------------------+                                                                             |
       |      |                                                                                                                            |
       |      +--> routing_table = tmp_routing_table                                                                                       |
       +-----------------------------------------------------------------------------------------------------------------------------------+
```

```
src/PCIeBinding.cpp                                                                                
+-----------------------------------+                                                               
| PCIeBinding::processBridgeEntries | : for each not-yet-contacted-bridge, get routing table from it
+-|---------------------------------+                                                               
  |                                                                                                 
  +--> for each entry in routing-table                                                              
       |                                                                                            
       |--> if it's not bridge || it's called already                                               
       |    -                                                                                       
       |    +--> continue                                                                           
       |                                                                                            
       |--> setup private packet                                                                    
       |                                                                                            
       |    +-------------------------------+                                                       
       +--> | PCIeBinding::readRoutingTable |                                                       
            +-------------------------------+                                                       
```

```
src/PCIeBinding.cpp                                                                               
+-------------------------------+                                                                  
| PCIeBinding::readRoutingTable | : commute with bus owner to update routing table internally      
+-|-----------------------------+                                                                  
  |                                                                                                
  +--> while not end of response                                                                   
       |                                                                                           
       |--> add (eid, bdf) to arg call_bridges                                                     
       |                                                                                           
       |    +------------------------------------+                                                 
       |--> | MCTPBridge::getRoutingTableCtrlCmd | send and receive mctp packet (get routing table)
       |    +------------------------------------+                                                 
       |                                                                                           
       +--> for each entry specified in response header                                            
            |                                                                                      
            |--> get target entry                                                                  
            |                                                                                      
            |--> advance offset                                                                    
            |                                                                                      
            +--> push entry to routing-table                                                       
```

```
src/mctp_bridge.cpp                                                                     
+------------------------------------+                                                   
| MCTPBridge::getRoutingTableCtrlCmd | : send and receive mctp packet (get routing table)
+-|----------------------------------+                                                   
  |                                                                                      
  |--> alloc request for 'get-routing-table-entries'                                     
  |                                                                                      
  |    +--------------------------------+                                                
  |--> | MCTPDevice::sendAndRcvMctpCtrl | send and receive mctp packet                   
  |    +--------------------------------+                                                
  |                                                                                      
  +--> for each entry specified in response header                                       
       |                                                                                 
       |--> get target entry from response                                               
       |                                                                                 
       +--> advance offset                                                               
```

```
src/mctp_device.cpp                                                                                                            
+--------------------------------+                                                                                              
| MCTPDevice::sendAndRcvMctpCtrl | : send and receive mctp packet                                                               
+-|------------------------------+                                                                                              
  |                                                                                                                             
  |--> prepare callback                                                                                                         
  |    +-----------------------------+                                                                                          
  |    | get response & packet state |                                                                                          
  |    |                             |                                                                                          
  |    | cancel timer                |                                                                                          
  |    +-----------------------------+                                                                                          
  |                                                                                                                             
  |    +-------------------------------+                                                                                        
  |--> | MCTPDevice::pushToCtrlTxQueue | add request to tx-queue, ensure timer is triggered to handle (re-send or discard) queue
  |    +-------------------------------+                                                                                        
  |                                                                                                                             
  +--> while pkt_state == pushed_for_transmission                                                                               
       -                                                                                                                        
       +--> create timer and wait for handling                                                                                  
```

```
src/mctp_device.cpp                                                                                                       
+-------------------------------+                                                                                          
| MCTPDevice::pushToCtrlTxQueue | : add request to tx-queue, ensure timer is triggered to handle (re-send or discard) queue
+-|-----------------------------+                                                                                          
  |                                                                                                                        
  |--> add request to tx_queue                                                                                             
  |                                                                                                                        
  |    +---------------------------------+                                                                                 
  |--> | MCTPDevice::sendMctpCtrlMessage | call libmctp to send tx                                                         
  |    +---------------------------------+                                                                                 
  |                                                                                                                        
  +--> if tx-timer expires                                                                                                 
       |                                                                                                                   
       |    +--------------------------------+                                                                             
       +--> | MCTPDevice::processCtrlTxQueue | setup periodic timer to re-send or discard requests in tx queue             
            +--------------------------------+                                                                             
```

```
src/mctp_device.cpp                                                                                
+--------------------------------+                                                                  
| MCTPDevice::processCtrlTxQueue | : setup periodic timer to re-send or discard requests in tx queue
+-|------------------------------+                                                                  
  |                                                                                                 
  +--> setup timer                                                                                  
       +----------------------------------------------------------------------+                     
       | discard packet from tx queue if condition met                        |                     
       | +------------------------------------------------------------------+ |                     
       | | if current pkt is still qualified for re-send                    | |                     
       | | |                                                                | |                     
       | | |    +---------------------------------+                         | |                     
       | | |--> | MCTPDevice::sendMctpCtrlMessage | call libmctp to send tx | |                     
       | | |    +---------------------------------+                         | |                     
       | | |                                                                | |                     
       | | |--> state = transmitted                                         | |                     
       | | |                                                                | |                     
       | | |--> retry_count--                                               | |                     
       | | |                                                                | |                     
       | | +--> return false                                                | |                     
       | |                                                                  | |                     
       | | run out of retry, call callback, e.g.,                           | |                     
       | | get pkt_state & response, cancel timer                           | |                     
       | +------------------------------------------------------------------+ |                     
       |                                                                      |                     
       | if  tx-queue becomes empty                                           |                     
       | |                                                                    |                     
       | |--> cancel tx-timer                                                 |                     
       | |                                                                    |                     
       | +--> timer_expired = true                                            |                     
       |                                                                      |                     
       | else                                                                 |                     
       | |                                                                    |                     
       | |    +--------------------------------+                              |                     
       | +--> | MCTPDevice::processCtrlTxQueue |                              |                     
       |      +--------------------------------+                              |                     
       +----------------------------------------------------------------------+                     
```

```
src/PCIeBinding.cpp                                                                            
+-----------------------------------+                                                           
| PCIeBinding::setDriverEndpointMap | : set arg new_table in mctp driver                        
+-|---------------------------------+                                                           
  |                                                                                             
  |--> for each tuple (eid, bdf, type) in new_table                                             
  |    |                                                                                        
  |    |                                                                                        
  |    +--> add to local 'endpoints'                                                            
  |                                                                                             
  |    +----------------------------+                                                           
  +--> | PCIeDriver::setEndpointMap | save eid table in mctp driver                             
       +----------------------------+ (libmcpt-intel implements mctp_astpcie_set_eid_info_ioctl)
```

```
src/PCIeBinding.cpp                                                                                                     
+-----------------------------------------+                                                                              
| PCIeBinding::processRoutingTableChanges | : compare old/new routint tables, unregister and register entries accordingly
+-|---------------------------------------+                                                                              
  |                                                                                                                      
  |--> for each entry in routing_table                                                                                   
  |    -                                                                                                                 
  |    +--> if it's not in new_table                                                                                     
  |         |                                                                                                            
  |         |    +--------------------------------+                                                                      
  |         +--> | MCTPDevice::unregisterEndpoint | given eid, remove its from dbus ifaces and routing_table             
  |              +--------------------------------+                                                                      
  |                                                                                                                      
  +--> for each entry in new_table                                                                                       
       -                                                                                                                 
       +--> if it's not in routing_table                                                                                 
            |                                                                                                            
            |--> prepare private data                                                                                    
            |                                                                                                            
            |    +-------------------------------+                                                                       
            |--> | MctpBinding::registerEndpoint | register eid (endpoint, or bus owner, or mix)                         
            |    +-------------------------------+                                                                       
            |                                                                                                            
            +--> log the bdf info                                                                                        
```

```
src/MCTPBinding.cpp                                                                                                            
+-------------------------------+                                                                                               
| MctpBinding::registerEndpoint | : register eid (endpoint, or bus owner, or mix)                                               
+-|-----------------------------+                                                                                               
  |                                                                                                                             
  |--> if the entry represents bus_owner                                                                                        
  |    |                                                                                                                        
  |    |    +--------------------------------------+                                                                            
  |    |--> | MCTPBridge::busOwnerRegisterEndpoint | given eid, send ctrl requests, register to bus owner and export dbus ifaces
  |    |    +--------------------------------------+                                                                            
  |    |                                                                                                                        
  |    |--> if anything goes wrong                                                                                              
  |    |    |                                                                                                                   
  |    |    |    +------------------------------------+                                                                         
  |    |    +--> | MctpBinding::clearRegisteredDevice |                                                                         
  |    |         +------------------------------------+                                                                         
  |    +--> return eid                                                                                                          
  |                                                                                                                             
  |    +--------------------------------------+                                                                                 
  |--> | MCTPBridge::getMsgTypeSupportCtrlCmd | send request to get supported_msg_types                                         
  |    +--------------------------------------+                                                                                 
  |                                                                                                                             
  |--> if eid is already registered, return                                                                                     
  |                                                                                                                             
  |    +----------------------------+                                                                                           
  |--> | MCTPBridge::getUuidCtrlCmd | send request to get uuid and check if it's valid                                          
  |    +----------------------------+                                                                                           
  |                                                                                                                             
  |--> setup ep properties                                                                                                      
  |                                                                                                                             
  |--> if eid is for bridge                                                                                                     
  |    |                                                                                                                        
  |    |    +--------------------------------------------------+                                                                
  |    |--> | MCTPBridge::sendNewRoutingTableEntryToAllBridges | send routing tables to all bridges                             
  |    |    +--------------------------------------------------+                                                                
  |    |                                                                                                                        
  |    +--> if it's bridge (with or without endpoints)                                                                          
  |         |                                                                                                                   
  |         |    +---------------------------------------------+                                                                
  |         +--> | MCTPBridge::sendRoutingTableEntriesToBridge | send routing tables                                            
  |              +---------------------------------------------+                                                                
  |    +---------------------------------------+                                                                                
  |--> | PCIeBinding::populateDeviceProperties | add interface for underlying device, e.g., bdf if it's pcie                    
  |    +---------------------------------------+                                                                                
  |    +------------------------------------------------+                                                                       
  +--> | MCTPDBusInterfaces::populateEndpointProperties | add interface for endpoint properties                                 
       +------------------------------------------------+                                                                       
```

```
src/mctp_bridge.cpp                                                                                                  
+--------------------------------------+                                                                              
| MCTPBridge::busOwnerRegisterEndpoint | : given eid, send ctrl requests, register to bus owner and export dbus ifaces
+-|------------------------------------+                                                                              
  |    +------------------------------------------+                                                                   
  |--> | MCTPBridge::getMctpVersionSupportCtrlCmd | send mctp request to get version data                             
  |    +------------------------------------------+                                                                   
  |    +---------------------------+                                                                                  
  |--> | MCTPBridge::getEidCtrlCmd | send request (get eid) and receive                                               
  |    +---------------------------+                                                                                  
  |--> get dst eid from response                                                                                      
  |    +---------------------------------------+                                                                      
  |--> | MCTPBridge::logUnsupportedMCTPVersion | log unsupported mctp version                                         
  |    +---------------------------------------+                                                                      
  |    +----------------------------+                                                                                 
  |--> | MCTPBridge::getUuidCtrlCmd | send request to get uuid and check if it's valid                                
  |    +----------------------------+                                                                                 
  |    +-------------------+                                                                                          
  |--> | isEIDMappedToUUID | given (eid, uuid), check if they are still the same in our mapping                       
  |    +-------------------+                                                                                          
  +--> if yes, return eid                                                                                             
  |    +-------------------------+                                                                                    
  |    | getEIDForReregistration | given uuid, get eid from map, and ensure there's no dbus interfaces of it          
  |    +-------------------------+                                                                                    
  |--> +-----------------------------------------+                                                                    
  |--> | DeviceWatcher::checkDeviceInitThreshold | check we haven't retried too many times on this device             
  |    +-----------------------------------------+                                                                    
  |--> ensure eid is valid                                                                                            
  |    +---------------------------+                                                                                  
  |--> | MCTPBridge::setEidCtrlCmd | send request to set eid                                                          
  |    +---------------------------+                                                                                  
  |--> update eid's status in pool                                                                                    
  |    +--------------------------------------+                                                                       
  |--> | MCTPBridge::getMsgTypeSupportCtrlCmd | send request to get supported_msg_types                               
  |    +--------------------------------------+                                                                       
  |--> if eid is already registered, return                                                                           
  |--> setup ep properties                                                                                            
  |    +---------------------------------------+                                                                      
  |--> | PCIeBinding::populateDeviceProperties | add interface for underlying device, e.g., bdf if it's pcie          
  |    +---------------------------------------+                                                                      
  |    +------------------------------------------------+                                                             
  |--> | MCTPDBusInterfaces::populateEndpointProperties | add interface for endpoint properties                       
  |    +------------------------------------------------+                                                             
  |--> update routing table                                                                                           
  +--> if eid is bridge                                                                                               
       -    +---------------------------------------------+                                                           
       +--> | MCTPBridge::sendRoutingTableEntriesToBridge |                                                           
            +---------------------------------------------+                                                           
```

```
src/mctp_bridge.cpp                                                                
+------------------------------------------+                                        
| MCTPBridge::getMctpVersionSupportCtrlCmd | : send mctp request to get version data
+-|----------------------------------------+                                        
  |    +-----------------+                                                          
  |--> | getFormattedReq | prepare mctp_ctrl request (get version support)          
  |    +-----------------+                                                          
  |    +--------------------------------+                                           
  |--> | MCTPDevice::sendAndRcvMctpCtrl | send and receive mctp packet              
  |    +--------------------------------+                                           
  |                                                                                 
  +--> get version data from response                                               
```

```
src/mctp_bridge.cpp                                                  
+---------------------------+                                         
| MCTPBridge::getEidCtrlCmd | : send request (get eid) and receive    
+-|-------------------------+                                         
  |                                                                   
  |--> prepare request of 'get eid'                                   
  |                                                                   
  |    +--------------------------------+                             
  |--> | MCTPDevice::sendAndRcvMctpCtrl | send and receive mctp packet
  |    +--------------------------------+                             
  |                                                                   
  +--> check resp size and completion code                            
```

```
src/mctp_bridge.cpp                                                             
+----------------------------+                                                   
| MCTPBridge::getUuidCtrlCmd | : send request to get uuid and check if it's valid
+-|--------------------------+                                                   
  |                                                                              
  |--> prepare request (gte eid uuid)                                            
  |                                                                              
  |    +--------------------------------+                                        
  +--> | MCTPDevice::sendAndRcvMctpCtrl | send and receive mctp packet           
       +--------------------------------+                                        
```
