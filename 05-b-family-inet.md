> Study case: Linux version 5.15.0 on AST2500 emulation

## Index

- [Introduction](#introduction)
- [User Datagram Protocol (UDP)](#udp)
- [Internet Control Message Protocol (ICMP)](#icmp)
- [Internet Group Management Protocol (IGMP)](#igmp)
- [Address Resolution Protocol (ARP)](#arp)
- [Raw](#raw)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

There are seven socket types in our study case. The INET family prepares five operation sets and registers them into three types.
- Type **STREAM**: it can only mean the combination of TCP and IP.
- Type **DGRAM**: the transport layer can be UDP, UDP lite, or ICMP.
- Type **RAW**: users prepare the transport layer themselves or even handle the Internet layer.

```
+---------------------------------------+                                
|      +--------------------------+     |                                
|      |                          |     |                                
|      |                socket(family, type, protocol)                   
|      |                                         |                       
|      |                                         |                       
|      v                                         |                       
|    INET                                        |                       
|   +--------------------------------------------|----------------------+
|   |    [STREAM]   ----   TCP/IP                |                      |
|   |                                            v                      |
+-------> [DGRAM]   ----   UDP/IP   ----   UDP_LITE/IP   ----   ICMP/IP |
    |                                                                   |
    |       [RAW]   ----   */IP                                         |
    |                                                                   |
    |       [RDM]                                                       |
    |                                                                   |
    | [SEQPACKET]                                                       |
    |                                                                   |
    |      [DCCP]                                                       |
    |                                                                   |
    |    [PACKET]                                                       |
    +-------------------------------------------------------------------+
```

On the receiving side, if the **protocol** filed in the MAC header is **IP**, the NAPI mechanism calls the handler **ip_rcv** to take over the packet. 
And IP layer will parse the **protocol** field in the IP header to decide the corresponding handler and deliver the packet for further processing.

```
                                          +----------+   
                +----    [ICMP]    -----  | icmp_rcv |   
                |                         +----------+   
                |                         +----------+   
                +----    [IGMP]    -----  | igmp_rcv |   
                |                         +----------+   
+--------+      |                         +------------+ 
| ip_rcv |  ---------     [TCP]    -----  | tcp_v4_rcv | 
+--------+      |                         +------------+ 
                |                         +---------+    
                |----     [UDP]    -----  | udp_rcv |    
                |                         +---------+    
                |                         +-------------+
                +----  [UDP_LITE]  -----  | udplite_rcv |
                                          +-------------+
```
     
## <a name="udp"></a> User Datagram Protocol (UDP)

```
              +--------+                                +--------+
              | TCP/IP |                                | UDP/IP |
              +--------+                                +--------+
                                       │
    client                 server      │      client                 server
 -----------------------------------   │   -----------------------------------
   socket()               socket()     │     socket()               socket()
       |                      |        │         |                      |
       |                      v        │         |                      v
       |                   bind()      │         |                   bind()
       |                      |        │         |                      |
       |                      v        │         |                      |
       |                  listen()     │         |                      |
       v                      |        │         |                      |
   connect()                  |        │         |                      |
       |                      v        │         |                      |
       |                  accept()     │         |                      |
       |                      |        │         |                      |
       v                      v        │         v                      v
read()/write()         read()/write()  │  read()/write()         read()/write()
       |                      |        │         |                      |
       v                      v        │         v                      v
    close()                close()     │      close()                close()
                                       │
```

```
                            write                         read


                        +-----------+                 +----------+
application             | sys_write |                 | sys_read |
   layer                +-----------+                 +----------+
                              |                             |
              ----------------|-----------------------------v----------------
                              v                      +-------------+
                       +-------------+               | udp_recvmsg | get data from queue
 transport             | udp_sendmsg |               +-------------+                                      -
   layer               +-------------+                 | udp_rcv |   put data onto queue
                              |                        +---------+
              ----------------|-----------------------------^----------------
                              v                             |
                       +-------------+                 +--------+
 internet              | ip_send_skb |                 | ip_rcv |
   layer               +-------------+                 +--------+
                              |                             ^
              ----------------|-----------------------------|----------------
                              v                             |
  network       +---------------------------+    +---------------------+
 interface      | ftgmac100_hard_start_xmit |    | ftgmac100_interrupt |
   layer        +---------------------------+    +---------------------+
```

<details>
  <summary> Code Trace </summary>

```
+----------+
| sys_bind |
+--|-------+
   |    +------------+
   +--> | __sys_bind |
        +--|---------+
           |    +---------------------+
           |--> | sockfd_lookup_light | get socket by file descriptor
           |    +---------------------+
           |    +---------------------+
           |--> | move_addr_to_kernel | copy data from user to kernel space
           |    +---------------------+
           |
           +--> call ->bind()
                      +-----------+
                e.g., | inet_bind | bind the given address to the socket
                      +--|--------+
                         |
                         |--> call type->bind() if it exists (not our case)
                         |
                         |    +-------------+
                         +--> | __inet_bind |
                              +---|---------+
                                  |
                                  +--> call type->get_port()
                                             +-----------------+
                                       e.g., | udp_v4_get_port |
                                             +----|------------+
                                                  |    +------------------+
                                                  +--> | udp_lib_get_port |
                                                       +----|-------------+
                                                            |
                                                            |--> if src port isn't specified
                                                            |
                                                            |------> find an available one
                                                            |
                                                            |--> else
                                                            |
                                                            |        check if it's available
                                                            |
                                                            +--> add the sock to the udp hash table
```

```
    +-----------+                                                                      
    | sys_write |                                                                      
    +-----------+                                                                      
          -                                                                            
          -                                                                            
          -                                                                            
 +-----------------+                                                                   
 | sock_write_iter |                                                                   
 +----|------------+                                                                   
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
                                             e.g., | udp_sendmsg | (refer to the below)
                                                   +-------------+                     
```

```
+-------------+                                                             
| udp_sendmsg |                                                             
+---|---------+                                                             
    |                                                                       
    |--> get dst addr and port by user arguments                            
    |                                                                       
    |--> if it's connected already                                          
    |                                                                       
    |------> get existing route table                                       
    |                                                                       
    |--> else                                                               
    |                                                                       
    |        +----------------------+                                       
    |------> | ip_route_output_flow | prepare route table                   
    |        +----------------------+                                       
    |                                                                       
    |--> decide src addr and port                                           
    |                                                                       
    |    +----------------+                                                 
    |--> | ip_append_data |                                                 
    |    +----------------+                                                 
    |    +-------------------------+                                        
    +--> | udp_push_pending_frames |                                        
         +------|------------------+                                        
                |    +----------------+                                     
                |--> |  ip_finish_skb | build ip header
                |    +----------------+                                     
                |    +-------------+                                        
                +--> | udp_send_skb|                                        
                     +---|---------+                                        
                         |                                                  
                         |--> build udp header                              
                         |                                                  
                         |    +-------------+                               
                         +--> | ip_send_skb | enter ip layer                
                              +-------------+                               
```

```
   +----------+
   | sys_read |
   +----------+
        -
        -
        -
+----------------+
| sock_read_iter |
+---|------------+
    |    +--------------+
    +--> | sock_recvmsg |
         +---|----------+
             |    +--------------------+
             +--> | sock_recvmsg_nosec |
                  +----|---------------+
                       |
                       +--> call ->recvmsg()
                                  +--------------+
                            e.g., | inet_recvmsg |
                                  +---|----------+
                                      |
                                      +--> call ->recvmsg()
                                                 +-------------+
                                           e.g., | udp_recvmsg | (refer to the below)
                                                 +-------------+
```

```
+-------------+                                         
| udp_recvmsg |                                         
+---|---------+                                         
    |    +----------------+                             
    |--> | __skb_recv_udp | get a skb from receive queue
    |    +----------------+                             
    |                                                   
    |--> copy data to msg                               
    |                                                   
    |--> if sin ('addr' struct) is provided             
    |                                                   
    |------> write info of family/port/addr to it       
    |                                                   
    |    +-----------------+                            
    +--> | skb_consume_udp | free skb                   
         +-----------------+                            
```

```
+---------+                                                                                     
| udp_rcv |                                                                                     
+--|------+                                                                                     
   |    +----------------+                                                                      
   +--> | __udp4_lib_rcv |                                                                      
        +---|------------+                                                                      
            |                                                                                   
            |--> analyze udp header                                                             
            |                                                                                   
            |    +-----------------------+                                                      
            |--> | __udp4_lib_lookup_skb | lookup target sock based on src & dst port           
            |    +-----------------------+                                                      
            |    +---------------------+                                                        
            +--> | udp_unicast_rcv_skb |                                                        
                 +-----|---------------+                                                        
                       |    +-------------------+                                               
                       +--> | udp_queue_rcv_skb |                                               
                            +----|--------------+                                               
                                 |    +-----------------------+                                 
                                 +--> | udp_queue_rcv_one_skb | add skb to receive queue of sock
                                      +-----------------------+                                 
```

</details>

## <a name="icmp"></a> Internet Control Message Protocol (ICMP)

ICMP is the protocol that serves the purpose of diagnosing and controlling. 
Utility **ping** is the application that works on top of ICMP, which supports up to eighteen types of messages. 
The typical **ECHO** is one of them and the only one I've ever used.

```
| Name                | Number | Note                    | Handler        |
| ---                 | ---    | ---                     | ---            |
| ICMP_ECHOREPLY      | 0      | Echo Reply              | ping_rcv       |
| ICMP_DEST_UNREACH   | 3      | Destination Unreachable | icmp_unreach   |
| ICMP_SOURCE_QUENCH  | 4      | Source Quench           | icmp_unreach   |
| ICMP_REDIRECT       | 5      | Redirect (change route) | icmp_redirect  |
| ICMP_ECHO           | 8      | Echo Request            | icmp_echo      |
| ICMP_TIME_EXCEEDED  | 11     | Time Exceeded           | icmp_unreach   |
| ICMP_PARAMETERPROB  | 12     | Parameter Problem       | icmp_unreach   |
| ICMP_TIMESTAMP      | 13     | Timestamp Request       | icmp_timestamp |
| ICMP_TIMESTAMPREPLY | 14     | Timestamp Reply         | icmp_discard   |
| ICMP_INFO_REQUEST   | 15     | Information Request     | icmp_discard   |
| ICMP_INFO_REPLY     | 16     | Information Reply       | icmp_discard   |
| ICMP_ADDRESS        | 17     | Address Mask Request    | icmp_discard   |
| ICMP_ADDRESSREPLY   | 18     | Address Mask Reply      | icmp_discard   |
```

By applying the **strace** on utility **ping** of busybox and **ping** in Ubuntu, we can observe that they create different socket types.

```
ping of busybox
    - socket(AF_INET, SOCK_DGRAM, IPPROTO_ICMP)
    - application provides the arguments, and kernel builds the icmp header
    
ping in ubuntu
    - socket(PF_INET, SOCK_RAW, IPPROTO_ICMP)
    - application builds the icmp header
```

```
                            write                                       read


                        +-----------+
application             | sys_write |
   layer                +-----------+
                              |
              ----------------|----------------------------------------------------------------------------
                              v
                       +-------------+
 transport             | raw_sendmsg |                +----------+                 +------------+
   layer               +-------------+                | icmp_rcv |  ------------>  | icmp_reply |
                              |                       +----------+                 +------------+
              ----------------|-----------------------------^-----------------------------|----------------
                              v                             |                             v
                       +-------------+                 +--------+                  +-------------+
 internet              | ip_send_skb |                 | ip_rcv |                  | ip_send_skb |
   layer               +-------------+                 +--------+                  +-------------+
                              |                             ^                             |
              ----------------|-----------------------------|-----------------------------|----------------
                              v                             |                             v
  network       +---------------------------+    +---------------------+    +---------------------------+
 interface      | ftgmac100_hard_start_xmit |    | ftgmac100_interrupt |    | ftgmac100_hard_start_xmit |
   layer        +---------------------------+    +---------------------+    +---------------------------+
```

<details>
  <summary> Code Trace </summary>

```
+----------+                                         
| icmp_rcv |                                         
+--|-------+                                         
   |                                                 
   |--> get icmp header from packet                  
   |                                                 
   +--> call [type].handler()                        
              +-----------+                          
        e.g., | icmp_echo |                          
              +--|--------+                          
                 |                                   
                 |--> set icmp type to ECHOREPLY     
                 |                                   
                 |    +------------+                 
                 +--> | icmp_reply |                 
                      +--|---------+                 
                         |    +---------------------+
                         |--> | ip_route_output_key |
                         |    +---------------------+
                         |    +-----------------+    
                         +--> | icmp_push_reply |    
                              +-----------------+    
```

</details>
    
## <a name="igmp"></a> Internet Group Management Protocol (IGMP)

(TBD)

## <a name="arp"></a> Address Resolution Protocol (ARP)

(TBD)

## <a name="raw"></a> Raw

When users choose the **raw** type, they will build the header of the transport layer specified in the argument **protocol**. 
Setting **protocol** to IPPROTO_RAW indicates the users will even prepare the IP header independently.

```
socket(PF_INET, SOCK_RAW, IPPROTO_ICMP)        <== users build transport header (e.g., icmp)
socket(PF_INET, SOCK_RAW, IPPROTO_RAW)         <== users build transport header + ip header
socket(PF_PACKET, SOCK_DGRAM, htons(ETH_P_IP)) <== users build transport header + ip header + mac header
                                                   (actually it's kernel that helps build the customized mac header)
```

<details>
  <summary> Code Trace </summary>

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
            |                                |--> allocate protocol sock (different from generic socket)
            |                                |
                                             |--> set ->hdrincl to 1 if user specifies protocol as 'raw'
            |                                |
            |                                +--> call ->init()
            |                                          +-------------+
            |                                     e.g.,| raw_sk_init | basically do nothing
            |                                          +-------------+
            |    +-------------+
            +--> | sock_map_fd | allocate a file handle for the socket, and install it to fd table
                 +-------------+
```

</details>
    

Type **RAW** supports ICMP in sendmsg() by the below two steps.
- Copy setting from user space to kernel space.
- Ask kernel to build the IP header and send out the packet.

<details>
  <summary> Code Trace </summary>

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
                                                         +-------------+                                                
                                           
```

```
+-------------+
| raw_sendmsg |
+---|---------+
    |
    |--> if ->hdrincl isn't set (kernel builds the ip header)
    |
    |        +---------------------+
    |------> | raw_probe_proto_opt |
    |        +-----|---------------+
    |              |
    |              |--> return if it's not icmp protocl
    |              |
    |              +--> copy icmp data from user to kernel
    |
    |    +----------------------+
    |--> | ip_route_output_flow | decide addr, interface, gw ip
    |    +----------------------+
    |
    +--> if ->hdrincl is set (user builds the ip header)
    |
    |        +-----------------+
    |------> | raw_send_hdrinc |
    |        +----|------------+
    |             |
    |             |--> set up skb
    |             |
    |             |    +------------+
    |             +--> | dst_output | send to driver layer
    |                  +------------+
    |
    |--> else (kernel builds the ip header)
    |
    |        +----------------+
    |------> | ip_append_data | append data to make a larger datagram
    |        +----------------+
    |
    |------> if no more data
    |
    |            +------------------------+
    +----------> | ip_push_pending_frames |
                 +-----|------------------+
                       |    +---------------+
                       |--> | ip_finish_skb | build ip header
                       |    +---------------+
                       |    +-------------+
                       +--> | ip_send_skb | send to driver layer
                            +-------------+
```

</details>

When the incoming packet arrives at the Internet layer, function ip_rcv parses the IP header to get the higher protocol. 
Knowing that helps it transfer the packet by calling the corresponding registered handler. 
So far, the INET family supports up to five handlers in my study case.
    
```
+---------+
| mac hdr | src & dst mac, higher protocol (ip)
+---------+
|  ip hdr | src & dst ip, higher protocol (???)
+---------+
|   ???   |
|---------|
|         |
|   data  |
|         |
+---------+
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

- [UDP Server-Client implementation in C](https://www.geeksforgeeks.org/udp-server-client-implementation-c/)
