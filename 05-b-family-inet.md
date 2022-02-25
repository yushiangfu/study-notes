> Study case: Linux version 5.15.0 on AST2500 emulation

## Index

- [Introduction](#introduction)
- [User Datagram Protocol (UDP)](#udp)
- [Internet Control Message Protocol (ICMP)](#icmp)
- [Address Resolution Protocol (ARP)](#arp)
- [Raw](#raw)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

(TBD)
     
## <a name="udp"></a> User Datagram Protocol (UDP)

(TBD)

## <a name="udp"></a> Internet Control Message Protocol (ICMP)

(TBD)

## <a name="udp"></a> Address Resolution Protocol (ARP)

(TBD)

## <a name="raw"></a> Raw

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
            |                            +-------------+
            |                      e.g., | inet_create | install family operations to socket and call ->init()
            |                            +---|---------+
            |                                |
            |                                |--> install family + type operations, e.g., inet_stream_ops
            |                                |
            |                                |--> allocate tcp socket (not the same socket created earlier)
            |                                |
            |                                +--> call ->init()
            |                                          +-------------+
            |                                     e.g.,| raw_sk_init | basically do nothing
            |                                          +-------------+
            |    +-------------+
            +--> | sock_map_fd | allocate a file handle for the socket, and install it to fd table
                 +-------------+
```

```
+------------+                                                                                                          
| sys_sendto |                                                                                                          
+--|---------+                                                                                                          
   |    +--------------+                                                                                                
   +--> | __sys_sendto |                                                                                                
        +---|----------+                                                                                                
            |    +---------------------+                                                                                
            |--> | import_single_range | copy buf from user space to kernel iov                                         
            |    +---------------------+                                                                                
            |    +---------------------+                                                                                
            |--> | sockfd_lookup_light | get socket through fd                                                          
            |    +---------------------+                                                                                
            |    +---------------------+                                                                                
            |--> | move_addr_to_kernel | copy 'addr' struct from user space to kernel                                   
            |    +---------------------+                                                                                
            |    +--------------+                                                                                       
            +--> | sock_sendmsg |                                                                                       
                 +---|----------+                                                                                       
                     |    +--------------------+                                                                        
                     +--> | sock_sendmsg_nosec |                                                                        
                          +----|---------------+                                                                        
                               |                                                                                        
                               +--> call ->sendmsg()                                                                    
                                          +--------------+                                                              
                                    e.g., | inet_sendmsg |                                                              
                                          +---|----------+                                                              
                                              |                                                                         
                                              +--> call ->sendmsg()                                                     
                                                         +-------------+                                                
                                                   e.g., | raw_sendmsg |                                                
                                                         +---|---------+                                                
                                                             |    +----------------------+                              
                                                             |--> | ip_route_output_flow | decide addr, interface, gw ip
                                                             |    +----------------------+                              
                                                             |    +-----------------+                                   
                                                             +--> | raw_send_hdrinc |                                   
                                                                  +----|------------+                                   
                                                                       |                                                
                                                                       |--> set up skb                                  
                                                                       |                                                
                                                                       |    +------------+                              
                                                                       +--> | dst_output | send to driver layer         
                                                                            +------------+                              
```

```
+--------------+
| sys_recvfrom |
+---|----------+
    |    +----------------+
    +--> | __sys_recvfrom |
         +---|------------+
             |    +---------------------+
             |--> | import_single_range | copy buf from user space to kernel iov
             |    +---------------------+
             |    +---------------------+
             |--> | sockfd_lookup_light | get socket through fd
             |    +---------------------+
             |    +--------------+
             |--> | sock_recvmsg |
             |    +---|----------+
             |        |    +--------------------+
             |        +--> | sock_recvmsg_nosec |
             |             +----|---------------+
             |                  |
             |                  +--> call ->sendmsg()
             |                             +--------------+
             |                       e.g., | inet_sendmsg |
             |                             +---|----------+
             |                                 |
             |                                 +--> call ->sendmsg()
             |                                            +-------------+
             |                                      e.g., | raw_recvmsg |
             |                                            +---|---------+
             |                                                |    +-------------------+
             |                                                |--> | skb_recv_datagram | receive skb from queue, might block
             |                                                |    +-------------------+
             |                                                |    +-----------------------+
             |                                                |--> | skb_copy_datagram_msg | copy data to msg
             |                                                |    +-----------------------+
             |                                                |    +-------------------+
             |                                                |--> | skb_free_datagram | free the skb
             |                                                |    +-------------------+
             |                                                |
             |                                                +--> fill 'addr' struct
             |
             |--> if user space 'addr' is provided
             |
             |        +-------------------+
             +------> | move_addr_to_user |
                      +-------------------+
```

</details>

## <a name="reference"></a> Reference

(TBD)
