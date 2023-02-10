### mctp-demux-daemon

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
