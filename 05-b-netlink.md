> Study case: Linux version 5.15.0 on AST2500 emulation

## Index

- [Introduction](#introduction)

Netlink is one of the network family and focus on bi-direction communication betwen user/user and user/kernel.
During boot time, kernel initializes the netlink table (nl_table), and each element contains a hash table for socket lookup.

| Name                   | Table Index | Receive Function |
| ---                    | ---         | ---              |
| NETLINK_ROUTE          | 0           | rtnetlink_rcv    |
| NETLINK_SOCK_DIAG      | 4           | sock_diag_rcv    |
| NETLINK_FIB_LOOKUP     | 10          | nl_fib_input     |
| NETLINK_NETFILTER      | 12          | nfnetlink_rcv    |
| NETLINK_KOBJECT_UEVENT | 15          | uevent_net_rcv   |
| NETLINK_GENERIC        | 16          | genl_rcv         |


- [Reference](#reference)

## <a name="introduction"></a> Introduction

(TBD)

```
+--------------+                                                                          
| sys_recvmmsg |                                                                          
+---|----------+                                                                          
    |    +----------------+                                                               
    +--> | __sys_recvmmsg |                                                               
         +---|------------+                                                               
             |    +-------------+                                                         
             +--> | do_recvmmsg |                                                         
                  +---|---------+                                                         
                      |    +---------------------+                                        
                      |--> | sockfd_lookup_light | get socket through fd                  
                      |    +---------------------+                                        
                      |    +----------------+                                             
                      +--> | ___sys_recvmsg |                                             
                           +---|------------+                                             
                               |    +---------------------+                               
                               |--> | recvmsg_copy_msghdr | copy msg iov to kernel        
                               |    +---------------------+                               
                               |    +-----------------+                                   
                               +--> | ____sys_recvmsg |                                   
                                    +----|------------+                                   
                                         |    +--------------+                            
                                         |--> | sock_recvmsg |                            
                                         |    +---|----------+                            
                                         |        |    +--------------------+             
                                         |        +--> | sock_recvmsg_nosec |             
                                         |             +----|---------------+             
                                         |                  |                             
                                         |                  +--> call ->recvmsg()         
                                         |                             +-----------------+
                                         |                       e.g., | netlink_recvmsg |
                                         |                             +-----------------+
                                         |                                                
                                         +--> copy data to userspace                      
```

```
+-------------------+                                                   
| skb_recv_datagram |                                                   
+----|--------------+                                                   
     |    +-------------------+                                         
     |--> | skb_recv_datagram | get an skb from receive queue           
     |    +-------------------+                                         
     |    +-----------------------+                                     
     |--> | skb_copy_datagram_msg | copy data from skb to msg iov       
     |    +-----------------------+                                     
     |    +-------------------+                                         
     |--> | skb_free_datagram | free the skb                            
     |    +-------------------+                                         
     |    +------------------+                                          
     +--> | netlink_rcv_wake | wake up the task waiting to read the data
          +------------------+                                          
```

```
+-----------------+                                                                               
| netlink_sendmsg |                                                                               
+----|------------+                                                                               
     |                                                                                            
     |--> if netlink sock isn't bound to netlink table yet                                        
     |                                                                                            
     |        +------------------+                                                                
     |------> | netlink_autobind | add to netlink table[portid]                                   
     |        +------------------+                                                                
     |    +-------------------------+                                                             
     |--> | netlink_alloc_large_skb | allocate skb                                                
     |    +-------------------------+                                                             
     |    +-----------------+                                                                     
     |--> | memcpy_from_msg | copy msg iov to skb                                                 
     |    +-----------------+                                                                     
     |                                                                                            
     |--> if dst group is specified                                                               
     |                                                                                            
     |        +-------------------+                                                               
     |------> | netlink_broadcast | for each sock on list, add socket into its queue and inform it
     |        +-------------------+                                                               
     |    +-----------------+                                                                     
     +--> | netlink_unicast | add socket into dst sock's queue and inform it                      
          +-----------------+                                                                     
```

```
+-----------------+                                                                                
| netlink_unicast |                                                                                
+----|------------+                                                                                
     |    +-------------------------+                                                              
     |--> | netlink_getsockbyportid | get dst sock by protocol and port id                         
     |    +-------------------------+                                                              
     |                                                                                             
     |--> if it's a kernel netlink                                                                 
     |                                                                                             
     |        +------------------------+                                                           
     |------> | netlink_unicast_kernel | call ->netlink_rcv() if it exists, otherwise free the skb 
     |        +------------------------+                                                           
     |                                                                                             
     |------> return                                                                               
     |                                                                                             
     |    +-----------------+                                                                      
     +--> | netlink_sendskb | add socket to dst sock's receive queue and call its ->sk_data_ready()
          +-----------------+                                                                      
```

```
+-------------------+                                                                                                                      
| netlink_broadcast |                                                                                                                      
+----|--------------+                                                                                                                      
     |    +----------------------------+                                                                                                   
     +--> | netlink_broadcast_filtered |                                                                                                   
          +------|---------------------+                                                                                                   
                 |                                                                                                                         
                 |--> set up broadcast info                                                                                                
                 |                                                                                                                         
                 |--> get list from netlink table[protocol]                                                                                
                 |                                                                                                                         
                 |--> for each sock on list                                                                                                
                 |                                                                                                                         
                 |        +------------------+                                                                                             
                 +------> | do_one_broadcast |                                                                                             
                          +----|-------------+                                                                                             
                               |    +---------------------------+                                                                          
                               +--> | netlink_broadcast_deliver |                                                                          
                                    +------|--------------------+                                                                          
                                           |    +-------------------+                                                                      
                                           +--> | __netlink_sendskb | add socket to dst sock's receive queue and call its ->sk_data_ready()
                                                +-------------------+                                                                      
```

```
+-------------+                                                                  
| sys_sendmsg |                                                                  
+---|---------+                                                                  
    |    +---------------+                                                       
    +--> | __sys_sendmsg |                                                       
         +---|-----------+                                                       
             |    +---------------------+                                        
             |--> | sockfd_lookup_light | get socket through fd                  
             |    +---------------------+                                        
             |    +----------------+                                             
             +--> | ___sys_sendmsg |                                             
                  +---|------------+                                             
                      |    +---------------------+                               
                      |--> | sendmsg_copy_msghdr | copy msg to kernel space      
                      |    +---------------------+                               
                      |    +-----------------+                                   
                      +--> | ____sys_sendmsg |                                   
                           +----|------------+                                   
                                |    +--------------+                            
                                +--> | sock_sendmsg |                            
                                     +---|----------+                            
                                         |    +--------------------+             
                                         +--> | sock_sendmsg_nosec |             
                                              +----|---------------+             
                                                   |                             
                                                   +--> call ->sendmsg()         
                                                              +-----------------+
                                                        e.g., | netlink_sendmsg |
                                                              +-----------------+
```

```
+------------+
| sys_socket |
+--|---------+
   |    +--------------+
   +--> | __sys_socket |
        +---|----------+
            |    +-------------+
            |--> | sock_create |
            |    +---|---------+
            |        |    +---------------+
            |        +--> | __sock_create |
            |             +---------------+
            |                 |   +------------+
            |                 |-->| sock_alloc | allocate socket
            |                 |   +------------+
            |                 |
            |                 +--> call family->create()
            |                            +----------------+
            |                      e.g., | netlink_create |
            |                            +---|------------+
            |                                |    +------------------+
            |                                |--> | __netlink_create | install 'netlink_ops' to socket, and prepare init sock
            |                                |    +------------------+
            |                                |
            |                                +--> get bind() and unbine() from netlink table and install to netlink sock
            |
            |    +-------------+
            +--> | sock_map_fd | allocate a file handle for the socket, and install it to fd table
                 +-------------+
```

```
+-------------------------+                                                            
| __netlink_kernel_create |                                                            
+------|------------------+                                                            
       |    +------------------+                                                       
       |--> | sock_create_lite | allocate socket                                       
       |    +------------------+                                                       
       |    +------------------+                                                       
       |--> | __netlink_create | install 'netlink_ops' to socket, and prepare sock
       |    +------------------+                                                       
       |                                                                               
       |--> install netlink_sock->netlink_rcv(), which is provided by caller           
       |                                                                               
       |    +----------------+                                                         
       |--> | netlink_insert | insert netlink sock to table[unit]                      
       |    +----------------+                                                         
       |                                                                               
       +--> ensure netlink table[unit] is initialized                                  
```

```
+--------------------+                                               
| netlink_proto_init |                                               
+----|---------------+                                               
     |    +----------------+                                         
     |--> | proto_register | register 'netlink_proto'                
     |    +----------------+                                         
     |    +-------------------+                                      
     |--> | bpf_iter_register |                                      
     |    +-------------------+                                      
     |                                                               
     |--> prepare netlink table                                      
     |                                                               
     |    +----------------------------+                             
     |--> | netlink_add_usersock_entry | init netlink table[USERSOCK]
     |    +----------------------------+                             
     |    +---------------+                                          
     |--> | sock_register | netlink_family_ops                       
     |    +---------------+                                          
     |    +------------------------+                                 
     |--> | register_pernet_subsys | register 'netlink_net_ops'      
     |    +------------------------+                                 
     |    +------------------------+                                 
     +--> | register_pernet_subsys | register 'netlink_tap_net_ops'  
          +------------------------+                                 
```

## <a name="reference"></a> Reference

[Netlink Examples](https://github.com/mwarning/netlink-examples)
