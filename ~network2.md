## <a name="ncsi"></a> NCSI

```
ncsi_probe_channel
    /* Probe */
    for each package, send cmd 'Deslect Package'      
    for each package
        send cmd 'Select Package'
        for each channel, send send cmd 'Clear Initial State'
        for each channel, send send cmd 'Get Version ID'
        for each channel, send send cmd 'Get Capabilities'
        for each channel, send send cmd 'Get Link'
        send cmd 'Deslect Package'      
    choose active channel
    
    /* Configure */
    send cmd 'Select Package'
    send cmd 'Clear Initial State'
    send cmd 'Set VLAN Filter' to clear VID
    send cmd 'Set VLAN Filter' to set VID
    send cmd 'Enable VLAN' or 'Disable VLAN'
    send cmd 'Set MAC Address'
    send cmd 'Enable Broadcast Filter'
    send cmd 'Disable Global Multicast Filter'
    send cmd 'Enable Channel Network TX'
    send cmd 'Enable Channel'
    send cmd 'AEN Enable'
    send cmd 'Get Link'
    start monitor (ncsi_channel_monitor)
    
        

```

###

```                                                                                                           
 ncsi-netlink.c                                                                                            
 [main]                                                                                                    
 |                                                                                                         
 |--> parse arguments                                                                                      
 |                                                                                                         
 |--> switch op                                                                                            
 |                                                                                                         
 |--> case pkg_info                                                                                        
 |    ---> [run_command_info] setup msg (pkg info), add ifindex/package to it, send msg and receive        
 |                                                                                                         
 |--> case set_interface                                                                                   
 |    ---> [run_command_set] setup msg (set iface), add ifindex/package/channel to it, send msg and receive
 |                                                                                                         
 |--> case clear_interface                                                                                 
 |    ---> [run_command_clear] setup msg (clear iface), add ifindex to it, send msg and receive            
 |                                                                                                         
 +--> case send_cmd                                                                                        
      ---> [run_command_send] setup msg (send cmd), add ifindex/package/channel to it, send msg and receive
```

```                                                                                           
 ncsi-netlink.c                                                                            
 [run_command_info] : setup msg (pkg info), add ifindex/package to it, send msg and receive
 |                                                                                         
 |--> [setup_ncsi_message] setup ncsi msg                                                  
 |                                                                                         
 |--> [nla_put_u32] add ifindex to msg                                                     
 |                                                                                         
 |--> if arg package is provided                                                           
 |    -                                                                                    
 |    +--> [nla_put_u32] add packagke to msg                                               
 |                                                                                         
 |--> [nl_socket_modify_cb] install func/arg to cb                                         
 |                          [info_cb]                                                      
 |                                                                                         
 |--> [nl_send_auto] complete and send out                                                 
 |                                                                                         
 +--> [nl_recvmsgs_default] receive msg                                                    
```

```                                                                           
 ncsi-netlink.c                                                            
 [setup_ncsi_message] : setup ncsi msg                                     
 |                                                                         
 |--> [nl_socket_alloc] alloc and init cb                                  
 |                                                                         
 |--> [__alloc_socket] alloc netlink socket                                
 |                                                                         
 |--> [genl_connect] alloc socket and setup (buf size, port, protocol, ...)
 |                                                                         
 |--> [genl_ctrl_resolve] given family name, get family id                 
 |                                                                         
 |--> [nlmsg_alloc] alloc msg                                              
 |                                                                         
 +--> [genlmsg_put] add header to msg                                      
```

```                                                                               
 lib/genl/genl.c                                                               
 [genl_connect] : alloc socket and setup (buf size, port, protocol, ...)       
 [nl_connect] : alloc socket and setup (buf size, port, protocol, ...)         
 |                                                                             
 |--> [socket] alloc socket                                                    
 |                                                                             
 |--> [nl_socket_set_buffer_size] set send/recv buffer size separately         
 |                                                                             
 |--> if socket port isn't specified                                           
 |    |                                                                        
 |    |--> endless loop                                                        
 |    |    |                                                                   
 |    |    |--> [_nl_socket_set_local_port_no_release] determine port, set flag
 |    |    |                                                                   
 |    |    |--> [bind] bind socket to local                                    
 |    |    |                                                                   
 |    |    +--> break                                                          
 |    |                                                                        
 |    +--> [_nl_socket_used_ports_release_all] update 'used_ports_map'         
 |                                                                             
 |--> [getsockname] get 'local' info                                           
 |                                                                             
 |--> if kernel assigned a different port                                      
 |    -                                                                        
 |    +--> [nl_socket_set_local_port] save it in sk                            
 |                                                                             
 +--> save local/protocol in sk                                                
```

```                                                                  
 lib/socket.c                                                     
 [_nl_socket_set_local_port_no_release] : determine port, set flag
 |                                                                
 |--> if 'generate_other' is specified                            
 |    -                                                           
 |    +--> pot = [generate_local_port] determine local port       
 |                                                                
 |--> else, port = 0                                              
 |                                                                
 +--> set flag accordingly                                        
```

```                                                                
 lib/genl/ctrl.c                                                
 [genl_ctrl_resolve] : given family name, get family id         
 |                                                              
 |--> [genl_ctrl_probe_by_name] given family name, get family id
 |                                                              
 +--> [genl_family_get_id] get family id                        
```

```                                                             
 lib/genl/ctrl.c                                             
 [genl_ctrl_probe_by_name] : given family name, get family id
 |                                                           
 |--> [genl_family_alloc] alloc netlink_family               
 |                                                           
 |--> save arg (name) in netlink_family                      
 |                                                           
 |--> [nlmsg_alloc] alloc msg                                
 |                                                           
 |--> [nl_socket_get_cb] get cb from netlink_socket          
 |                                                           
 |--> [nl_cb_clone] clone cb                                 
 |                                                           
 |--> [genlmsg_put] fill out header                          
 |                                                           
 |--> [nla_put_string] add string to msg                     
 |                                                           
 |--> [nl_cb_set] given type, install func/arg in cb         
 |                                                           
 |--> [nl_send_auto_complete] complete msg and send out      
 |                                                           
 |--> [nl_recvmsgs] receive msg                              
 |                                                           
 |--> [wait_for_ack] wait for ack if required                
 |                                                           
 +--> [genl_family_get_id] get family id                     
```

```                                                                                                   
 ncsi-netlink.c                                                                                    
 [run_command_set] : setup msg (set iface), add ifindex/package/channel to it, send msg and receive
 |                                                                                                 
 |--> [setup_ncsi_message] setup ncsi msg                                                          
 |                                                                                                 
 |--> [nla_put_u32] add ifindex to msg                                                             
 |                                                                                                 
 |--> [nla_put_u32] add package to msg                                                             
 |                                                                                                 
 |--> if channel is provided                                                                       
 |    -                                                                                            
 |    +--> [nla_put_u32] add channel to msg                                                        
 |                                                                                                 
 |--> [nl_send_auto] complete and send out                                                         
 |                                                                                                 
 +--> [nl_recvmsgs_default] receive msg                                                            
```

```                                                                                       
 ncsi-netlink.c                                                                        
 [run_command_clear] : setup msg (clear iface), add ifindex to it, send msg and receive
 |                                                                                     
 |--> [setup_ncsi_message] setup ncsi msg                                              
 |                                                                                     
 |--> [nla_put_u32] add ifindex to msg                                                 
 |                                                                                     
 |--> [nl_send_auto] complete and send out                                             
 |                                                                                     
 +--> [nl_recvmsgs_default] receive msg                                                
```

```                                                                                                   
 ncsi-netlink.c                                                                                    
 [run_command_send] : setup msg (send cmd), add ifindex/package/channel to it, send msg and receive
 |                                                                                                 
 |--> alloc data buffer                                                                            
 |                                                                                                 
 |--> copy arg payload to buffer                                                                   
 |                                                                                                 
 |--> [setup_ncsi_message] setup ncsi msg                                                          
 |                                                                                                 
 |--> [nla_put_u32] add ifindex to msg                                                             
 |                                                                                                 
 +--> if package is provided                                                                       
 |    -                                                                                            
 |    +--> [nla_put_u32] add package to msg                                                        
 |                                                                                                 
 |--> [nla_put_u32] add channel to msg                                                             
 |                                                                                                 
 |--> fill header                                                                                  
 |                                                                                                 
 |--> [nl_socket_disable_seq_check] disable seq# checking                                          
 |                                                                                                 
 |--> [nl_socket_modify_cb] install func/arg to cb                                                 
 |                          [send_cb]                                                              
 |                                                                                                 
 |--> [nl_send_auto] complete and send out                                                         
 |                                                                                                 
 +--> [nl_recvmsgs_default] receive msg                                                            
```

### mdio-tool

```                                                                                                                                                
 mdio-tool.c                                                                                                                                    
 [main]                                                                                                                                         
 |                                                                                                                                              
 |--> [socket] open a dgram socket                                                                                                              
 |                                                                                                                                              
 |--> given iface name (from arg), save to ifr                                                                                                  
 |                                                                                                                                              
 |--> [ioctl] get_mii_phy                                                                                                                       
 |                                                                                                                                              
 |--> if 'read' is specified                                                                                                                    
 |    |                                                                                                                                         
 |    |--> parse addr from arg                                                                                                                 -
 |    |                                                                                                                                         
 |    +--> [mdio_read] given addr, get mii register value                                                                                       
 |                                                                                                                                              
 +--> elif 'write' is specified                                                                                                                 
      |                                                                                                                                         
      |--> parse addr/val from arg                                                                                                              
      |                                                                                                                                         
      +--> [mdio_write] given addr/val, set mii register value                                                                                  
```
