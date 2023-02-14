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
