## Index

1. [Introduction](#introduction)
2. [Reference](#reference)

## <a name="introduction"></a> Introduction

```
+----------------+
| netboot_common |
+---|------------+
    |
    |--> set default 'load_addr'
    |
    |--> switch argc
    |
    |------> case 1 (only cmd 'dhcp' and no argument)
    |
    |----------> set 'net_boot_file_name' from env "bootfile"
    |
    +------> case 2 (cmd 'dhcp' and one argument)
    |
    |----------> if the arg is addr
    |
    |--------------> set 'load_addr' from the arg
    |
    |--------------> set 'net_boot_file_name' from env "bootfile"
    |
    |----------> else
    |
    |--------------> set 'net_boot_file_name' from the arg
    |
    |------> case 3 (we have everything we need)
    |
    +----------> set 'load_addr' and 'net_boot_file_name' from args
    |
    |    +----------+
    |--> | net_loop | send and receive dhcp packets, load boot file
    |    +----------+
    |    +--------------------+
    |--> | netboot_update_env | update variables to uboot env
    |    +--------------------+
    |    +-----------------------+
    +--> | bootm_maybe_autostart | boot up if "autostart" is set
         +-----------------------+
```

```
+---------------+                                                               
| bootp_request | set up tx packet, install timeout and udp handler, send packet
+---|-----------+                                                               
    |                                                                           
    |--> determine timeout period                                               
    |                                                                           
    |--> print "BOOTP broadcast %d\n"                                           
    |                                                                           
    |--> set up a tx packet (udp, ip, broadcast)                                
    |                                                                           
    |    +---------------+                                                      
    |--> | dhcp_extended | further set up                                       
    |    +---------------+                                                      
    |                                                                           
    |--> set timeout handler 'bootp_timeout_handler()'                          
    |                                                                           
    |--> set udp handler 'dhcp_handler'                                         
    |                                                                           
    |    +-----------------+                                                    
    +--> | net_send_packet |                                                    
         +----|------------+                                                    
              |    +----------+                                                 
              +--> | eth_send |                                                 
                   +--|-------+                                                 
                      |                                                         
                      +--> call ->send(), e.g.,                                 
                           +---------------+                                    
                           | ftgmac100_send|                                    
                           +---------------+                                    
```

```
 +----------+                                                                               
 | net_loop | send and receive dhcp packets, load boot file
 +--|-------+                                                                               
    |    +----------+                                                                       
    |--> | net_init | inti variables and get mac addr                                       
    |    +----------+                                                                       
    |    +----------+                                                                       
    |--> | eth_init |                                                                       
    |    +--|-------+                                                                       
    |       |                                                                               
    |       +--> call ->start(), e.g.,                                                      
    |            +-----------------+                                                        
    |            | ftgmac100_start |                                                        
    |            +----|------------+                                                        
    |                 |                                                                     
    |                 |--> init                                                             
    |                 |                                                                     
    |                 +--> print "%s: link up, %d Mbps %s-duplex mac:%pM\n"                 
 restart:                                                                                   
    |    +---------------+                                                                  
    |--> | net_init_loop | get mac addr                                                     
    |    +---------------+                                                                  
    |                                                                                       
    |--> swtich 'protocol'                                                                  
    |                                                                                       
    |--> case 'dhcp'                                                                        
    |                                                                                       
    |        +--------------+                                                               
    |------> | dhcp_request | set up tx packet, install timeout and udp handler, send packet
    |        +--------------+                                                               
    |                                                                                       
    |--> endless loop                                                                       
    |                                                                                       
    |        +--------+                                                                     
    |------> | eth_rx | receive and handle up to 32 packets, load the boot file implicitly
    |        +--------+                                                                     
    |                                                                                       
    |------> run timeout handler if necessary                                               
    |                                                                                       
    |------> switch net_state                                                               
    |                                                                                       
    |------> case 'restart'                                                                 
    |                                                                                       
    |-----------> go to restart                                                             
    |                                                                                       
    |------> case                                                                           
    |                                                                                       
    +-----------> print "Bytes transferred = %d (%x hex)\n"                                 
```

```
+--------+                                                                               
| eth_rx | receive and handle up to 32 packets, load the boot file implicitly
+-|------+                                                                               
  |                                                                                      
  |--> for up to 32 packets                                                              
  |                                                                                      
  |------> call ->recv(), e.g.,                                                          
  |        +----------------+                                                            
  |        | ftgmac100_recv |                                                            
  |        +----------------+                                                            
  |        +-----------------------------+                                               
  |------> | net_process_received_packet |                                               
  |        +-------|---------------------+                                               
  |                |                                                                     
  |                |--> parse header and check                                           
  |                |                                                                     
  |                +--> call udp handler, e.g.,                                          
  |                     +--------------+                                                 
  |                     | dhcp_handler |                                                 
  |                     +---|----------+                                                 
  |                         |                                                            
  |                         |--> switch dhcp_state                                       
  |                         |                                                            
  |                         |--> case REQUESTING                                         
  |                         |                                                            
  |                         |------> print "DHCP client bound to address %pI4 (%lu ms)\n"
  |                         |                                                            
  |                         |        +---------------+                                   
  |                         +------> | net_auto_load |                                   
  |                                  +---|-----------+                                   
  |                                      |    +------------+                             
  |                                      +--> | tftp_start |                             
  |                                           +------------+                             
  |                                                                                      
  +------> break loop early if no packet                                                 
```

```
+--------+                                                                                                        
| _start | [arch/arm/lib/vectors.S]                                                                               
+-|------+                                                                                                        
  |    +-------+                                                                                                  
  +--> | reset | [arch/arm/cpu/arm1176/start.S]                                                                   
       +-|-----+                                                                                                  
         |    +------------------+                                                                                
         +--> | save_boot_params |                                                                                
              +----|-------------+                                                                                
                   |    +----------------------+                                                                  
                   +--> | save_boot_params_ret |                                                                  
                        +-----|----------------+                                                                  
                              |                                                                                   
                              |--> set cpu to svc mode                                                            
                              |                                                                                   
                              |    +---------------+                                                              
                              +--> | cpu_init_crit |                                                              
                                   +---|-----------+                                                              
                                       |    +-------------+                                                       
                                       +--> | mmu_disable |                                                       
                                            +---|---------+                                                       
                                                |    +---------------+                                            
                                                |--> | lowlevel_init | [arch/arm/mach-aspeed/ast2500/platform.S]  
                                                |    +---------------+                                            
                                                |    +-------+                                                    
                                                +--> | _main | arch/arm/lib/crt0.S                                
                                                     +-|-----+                                                    
                                                       |    +---------------------------+                         
                                                       |--> | board_init_f_init_reserve | alloc the reserved space
                                                       |    +---------------------------+                         
                                                       |    +---------------------------+                         
                                                       |--> | board_init_f_init_reserve | init t he reserved space
                                                       |    +---------------------------+                         
                                                       |    +--------------+                                      
                                                       +--> | board_init_f | [common/board_f.c]                   
                                                            +--------------+                                      
```

## <a name="reference"></a> Reference

(TBD)
