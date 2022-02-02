> Study case: Linux version 5.15.0 on AST2500 emulation

## Index

- [Introduction](#introduction)
- [Network Layers & Families](#network-layers-and-families)
- [Application Layer](#application-layer)
- [Transport Layer](#transport-layer)
- [Internet Layer](#internet-layer)
- [Network Interface Layer](#network-interface-layer)
- [Boot Flow](#boot-flow)
- [To-Do List](#to-do-list)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

(TBD)

## <a name="network-layers-and-families"></a> Network Layers & Families

The server and client models are the most common in our daily network environment. 
The providers build a machine, or a bunch of them, to wait for the client-side requests and either serve or drop it. 
The service type defines the system as a file server, web server, DHCP server, etc. 
Let's start from the server-side and break the network stack into four implementation layers.
- Application layer
  - Determine how to service the request, e.g., to transmit the video content back
- Transport layer
  - Improve transmission robustness, e.g., re-sending the packet if the recipient complains about getting nothing.
- Internet protocol layer
  - Route a packet to the destination, or split a packet into smaller sizes if necessary.
- Physical layer
  - Works on the data sending and receiving without any idea of the big picture.

The four-layers framework kernel implementation can accommodate the most common protocols in our daily lives. 
Well-known protocols such as TCP, UDP, IP, ICMP come from the same network family. 
Their IPv6 versions get grouped in another family. 
Netlink, a message exchange method between user and kernel space, is also a network family.

```
# Check how many network families the kernel supports
root@romulus:~# dmesg | grep family
[    0.156635] NET: Registered PF_NETLINK/PF_ROUTE protocol family
[    0.411547] NET: Registered PF_INET protocol family
[    0.435757] NET: Registered PF_UNIX/PF_LOCAL protocol family <========== no idea yet
[    0.525302] NET: Registered PF_ALG protocol family           <========== no idea yet
[    2.908709] NET: Registered PF_INET6 protocol family
[    2.931712] NET: Registered PF_PACKET protocol family        <========== no idea yet

```

We will introduce how the network works in Linux based on the below combination of TCP and IP (INET family), the so-called TCP/IP model.

## <a name="application-layer"></a> Application Layer

The kernel provides a few network-related syscalls to userspace, and whichever requests the services through those functions belongs to the application layer.
Both the client and server sides call **socket()** to get the handle. 
The server calls **bind()** to relate the socket handle to its network address and then calls **listen()** to standby. 
Anytime the client can **connect()** to the server, wait for its **accept()**, and advance to the real deal. 
Data operation can be as simple as combining a few **read()** and **write()**. 
Since kernel conceals the most complicated part, users can operate socket data like handling a file. 
Either side can **close()** this connection, and the other side will process the termination appropriately.

```                                 
     client                 server                                           
  -----------------------------------                                        
    socket()               socket()     prepare handle for network operations
        |                      |                                             
        |                      v                                             
        |                   bind()      bind address (port) to the socket    
        |                      |                                             
        |                      v                                             
        |                  listen()     wait for connection                  
        v                      |                                             
    connect()                  |                                             
        |                      v                                             
        |                  accept()     accept the connection                
        |                      |                                             
        v                      v                                             
 read()/write()         read()/write()  regular data operation               
        |                      |                                             
        v                      v                                             
     close()                close()     close the connection                  
```

## <a name="transport-layer"></a> Transport Layer

The transport layer provides reliable transmission, flow control, connection concepts. 
Transmission Control Protocol (TCP) consists of the above features, while User 
Datagram Protocol (UDP) provides a simplified method for another transmission. 
Let's inspect how the TCP layer works behind each syscall to serve the request from the application layer.

### socket()

```
prototype:
SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)
```

- **family**
  - It's the network family we mentioned above, and it will be PF_INET in our study case.
- **type**
  - Once specifying the family as PF_INET, we have to select a type from the below list, and it will be SOCK_STREAM (TCP) in our case.
  ```
  SOCK_STREAM = 1,  <========== TCP
  SOCK_DGRAM  = 2,  <========== UDP
  SOCK_RAW    = 3,
  SOCK_RDM    = 4,
  SOCK_SEQPACKET  = 5,
  SOCK_DCCP   = 6,
  SOCK_PACKET = 10, 
  ```
- **protocol**
  - With the family PF_INET and type TCP or UDP selected, this field can only be IP.

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
            |                      e.g., | inet_create | install family  operations to socket and call ->init()
            |                            +---|---------+
            |                                |
            |                                |--> set socket state to UNCONNECTED
            |                                |
            |                                |--> install family + type operations, e.g., inet_stream_ops
            |                                |
            |                                |--> allocate tcp socket (not the same socket created earlier)
            |                                |
            |                                +--> call ->init()
            |                                           +------------------+
            |                                     e.g., | tcp_v4_init_sock | init tcp socket
            |                                           +------------------+
            |                                                |    +---------------+
            |                                                |--> | tcp_init_sock |
            |                                                |    +---------------+
            |                                                |
            |                                                +--> install ipv4 operations
            |
            |    +-------------+
            +--> | sock_map_fd | allocate a file handle for the socket, and install it to fd table
                 +-------------+                                                                        
```

### bind()

The caller will prepare the address structure, and it's optional to specify the port. 
The **bind** function will further check if the specified port is available or prepare a valid one if the caller hasn't set it.

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
                                             +-------------------+
                                       e.g., | inet_csk_get_port | 
                                             +-------------------+
                                                  |
                                                  |--> if no given port, find a valid one and return
                                                  |
                                                  +--> check if given port is valid
```

### listen()

In TCP design, the socket has different states indicating its current status. 
The function **listen** will change the socket state accordingly and add it to the hash table, waiting for the client to connect. 
We can use the utility **netstat** to display system socket status, and I guess the information comes from the hash table.

```
+------------+
| sys_listen |
+--|---------+
   |    +--------------+
   +--> | __sys_listen |
        +---|----------+
            |    +---------------------+
            |--> | sockfd_lookup_light | get socket by file descriptor
            |    +---------------------+
            |
            +--> call ->listen()
                       +-------------+
                 e.g., | inet_listen | set tcp state to 'LISTEN' and add to hash table wating for connection
                       +---|---------+
                           |    +-----------------------+
                           +--> | inet_csk_listen_start |
                                +-----|-----------------+
                                      |
                                      |--> set tcp state to LISTEN
                                      |
                                      |--> call type->get_port()
                                      |          +-------------------+
                                      |    e.g., | inet_csk_get_port | ensure we have a valid port
                                      |          +-------------------+
                                      |
                                      +--> call type->hash()
                                                 +-----------+
                                           e.g., | inet_hash | add socket to hash table waiting for connection
                                                 +-----------+ 
```

### connect()

They call this function with server address information on the client side to build the packet and send it to the destination. 
In TCP design, re-sending is possible if the sender receives no response from the target in a specified interval. 
The server determines whether to accept the packet, and the client socket will change the socket state to CONNECTED after learning the acceptance from the server.

```
 +-------------+
 | sys_connect |
 +---|---------+
     |    +---------------+
     +--> | __sys_connect |
          +---|-----------+
              |    +---------------------+
              |--> | move_addr_to_kernel | copy data from user to kernel space
              |    +---------------------+
              |    +--------------------+
              +--> | __sys_connect_file |
                   +----|---------------+
                        |
                        +--> call ->->connect()
                                   +---------------------+
                             e.g., | inet_stream_connect | build socket buffer, send out, wait for acception
                                   +---------------------+
                                         |    +-----------------------+
                                         +--> | __inet_stream_connect |
                                              +-----|-----------------+
                                                    |
                                                    +--> call ->connect()
                                                               +----------------+
                                                         e.g., | tcp_v4_connect | (refer to the below)
                                                               +----------------+
```

```
+----------------+
| tcp_v4_connect | set addr and port, allocate skb and build tcp header, send to ip layer
+---|------------+
    |    +------------------+
    |--> | ip_route_connect | prepare routing table, and decide next hop
    |    +------------------+
    |
    |--> set destination addr & port
    |
    |--> change tcp state to SYN_SENT
    |
    |    +-------------------+
    |--> | inet_hash_connect | bind a port to the socket and add to hash table
    |    +-------------------+
    |    +-------------+
    +--> | tcp_connect |
         +---|---------+
             |    +---------------------+
             |--> | sk_stream_alloc_skb | allocate socket buffer (skb)
             |    +---------------------+
             |    +------------------+
             |--> | tcp_transmit_skb |
             |    +----|-------------+
             |         |    +--------------------+
             |         +--> | __tcp_transmit_skb | build tcp header, send to ip layer for transmit
             |              +--------------------+
             |    +---------------------------+
             +--> | inet_csk_reset_xmit_timer | re-send the packet if no response before timeout
                  +---------------------------+                                        
```

### accept()

Like the function **connect()** waits for acceptance, function **accept()** waits for connection. 
It will prepare another file descriptor, socket, file for the connection.

```
+------------+
| sys_accept |
+--|---------+
   |    +----------------+
   +--> | __sys_accept4  |
        +---|------------+
            |    +--------------------+
            +--> | __sys_accept4_file |
                 +----|---------------+
                      |    +-----------------------+
                      +--> | __get_unused_fd_flags | get an valid file descriptor
                      |    +-----------------------+
                      |    +-----------+
                      |--> | do_accept |
                      |    +--|--------+
                      |       |    +------------+
                      |       |--> | sock_alloc | allocate socket for the connection
                      |       |    +------------+
                      |       |    +-----------------+
                      |       |--> | sock_alloc_file | allocate file for the connection
                      |       |    +-----------------+
                      |       |
                      |       |--> call ->accept()
                      |       |          +-------------+
                      |       |    e.g., | inet_accept | wait for connection, set socket state to CONNECTED
                      |       |          +---|---------+
                      |       |              |
                      |       |              +--> call ->accept()
                      |       |                         +-----------------+
                      |       |                   e.g., | inet_csk_accept |
                      |       |                         +----|------------+
                      |       |                              |
                      |       |                              |--> if no connection request in queue
                      |       |                              |
                      |       |                              |        +---------------------------+
                      |       |                              |        | inet_csk_wait_for_connect |
                      |       |                              |        +---------------------------+
                      |       |                              |    +--------------------+
                      |       |                              +--> | reqsk_queue_remove | remove a request
                      |       |                                   +--------------------+
                      |       |
                      |       |--> copy address info to userspace if asked to
                      |       |
                      |       +--> return allocated file
                      |
                      |    +------------+
                      +--> | fd_install | install the file to allocated fd
                           +------------+                                           
```

### sys_write()

VFS layer

```
+-----------+                                                                                
| sys_write |                                                                                
+--|--------+                                                                                
   |    +------------+                                                                       
   +--> | ksys_write |                                                                       
        +--|---------+                                                                       
           |    +-----------+                                                                
           |--> | file_ppos | get file offset                                                
           |    +-----------+                                                                
           |    +-----------+                                                                
           |--> | vfs_write |                                                                
           |    +--|--------+                                                                
           |       |                                                                         
           |       |--> if file has ->write                                                  
           |       |                                                                         
           |       |         call ->write()                                                  
           |       |                                                                         
           |       +--> else if file has ->write_iter                                        
           |                                                                                 
           |                +----------------+                                               
           |                | new_sync_write |                                               
           |                +---|------------+                                               
           |                    |    +-----------------+                                     
           |                    +--> | call_write_iter |                                     
           |                         +----|------------+                                     
           |                              |                                                  
           |                              +--> call ->write_iter()                           
           |                                         +-----------------+                     
           |                                   e.g., | sock_write_iter | <========== our case
           |                                         +-----------------+                     
           |                                                                                 
           +--> update file offset
```

TCP layer

```
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
                                            e.g., | tcp_sendmsg |
                                                  +-------------+
```

```
+-------------+
| tcp_sendmsg |
+---|---------+
    |
    |--> where there's data to be sent
    |
    |        +---------------------+
    |        | sk_stream_alloc_skb | allcoate struct sk_buff for buffer management
    |        +---------------------+
    |        +---------------------+
    |        | sk_page_frag_refill | allocate a few pages as buffer
    |        +---------------------+
    |        +--------------------------+
    |        | skb_copy_to_page_nocache | copy data to the buffer
    |        +--------------------------+
    |        +--------------------+
    |        | skb_fill_page_desc | file the buffer descriptor in struct sk_buff
    |        +--------------------+
    |
    |        if not data left to be sent, exit the loop
    |
    +--> if we did copy data to socket buffer

             +----------+
             | tcp_push |
             +--|-------+
                |    +---------------------------+
                +--> | __tcp_push_pending_frames |
                     +------|--------------------+
                            |    +----------------+
                            +--> | tcp_write_xmit |
                                 +---|------------+
                                     |    +---------------+
                                     |--> | tcp_mtu_probe | get MTU size
                                     |    +---------------+
                                     |
                                     |--> while socket has buffer (one socket might have multiple skb)
                                     |
                                     |    +------------------+
                                     +--> | tcp_transmit_skb |
                                          +----|-------------+
                                               |    +--------------------+
                                               +--> | __tcp_transmit_skb | build tcp header and send to ip layer
                                                    +--------------------+                                     
```

### sys_read()

VFS layer

```
+----------+                                                          
| sys_read |                                                          
+--|-------+                                                          
   |    +-----------+                                                 
   +--> | ksys_read |                                                 
        +--|--------+                                                 
           |    +-----------+                                         
           |--> | file_ppos | get file offset                         
           |    +-----------+                                         
           |    +----------+                                          
           |--> | vfs_read |                                          
           |    +--|-------+                                          
           |       |                                                  
           |       |--> if ->read exists                              
           |       |                                                  
           |       |        call ->read()                             
           |       |                                                  
           |       +--> else if ->read_iter exists                    
           |                                                          
           |                +---------------+                         
           |                | new_sync_read |                         
           |                +---|-----------+                         
           |                    |    +----------------+               
           |                    +--> | call_read_iter |               
           |                         +---|------------+               
           |                             |                            
           |                             +--> call ->read_iter()      
           |                                                          
           |                                  e.g., +----------------+
           |                                        | sock_read_iter |
           |                                        +----------------+
           |                                                          
           +--> update file offset                                    
```

TCP layer

```
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
                                           e.g., | tcp_recvmsg |
                                                 +-------------+
```

```
+-------------+                                                                                      
| tcp_recvmsg |                                                                                      
+---|---------+                                                                                      
    |    +--------------------+                                                                      
    +--> | tcp_recvmsg_locked |                                                                      
         +----|---------------+                                                                      
              |                                                                                      
              +--> loop                                                                              
                                                                                                     
                       if receive queue has skb(s)                     --+                           
                                                                         |                           
                           +-----------------------+                     |                           
                           | skb_copy_datagram_msg |                     |                           
                           +-----------------------+                     |                           
                                                                         | extremely simplified logic
                       else                                              |                           
                                                                         |                           
                           +--------------+                              |                           
                           | sk_wait_data | wait for data to arrive      |                           
                           +--------------+                            --+                           
```

### sys_close()

(TBD)

## <a name="internet-layer"></a> Internet Layer

If we specify TCP as our transport layer, it must be Internet Protocol (IP) as the network layer whether we set it. 
As we can observe from the above TCP layer, only **connect()** and **write()** involve the IP layer through **tcp_transmit_skb()** for sending the packet out. 
As for **read()**,  it retrieves skb from the receive queue, but it's probably the IP layer that places skb(s) onto the queue.

| TCP Layer      | IP Layer                         |
| ---            | ---                              |
| tcp_v4_connect | ip_route_connect & ip_queue_xmit |
| tcp_sendmsg    | ip_queue_xmit                    |
| tcp_recvmsg    | ip_rcv                           |

Let's skip **ip_route_connect** for now since I don't understand fib lookup and neighbor mechanism. 

```
                                             +-------- bits                                 
                                             | +------ full children                        
                                             | | +---- empty children                       
                                             | | |                                          
                             Main:           v v v                                          
                               +-- 0.0.0.0/1 2 0 2                                          
           node (+--) ==========> +-- 0.0.0.0/4 2 0 2                                       
           leaf (|--) ==========>    |-- 0.0.0.0           <========== added by userspace   
                                        /0 universe UNICAST                                 
                                     +-- 10.0.2.0/24 2 0 2                                  
                                        +-- 10.0.2.0/28 2 1 2                               
                                           +-- 10.0.2.0/30 2 0 1                            
                                              |-- 10.0.2.0 <========== added by userspace   
             (mask_len, scope, type) ==========> /24 link UNICAST                           
                                              |-- 10.0.2.2 <========== added by userspace   
                                                 /32 link UNICAST                           
                                              |-- 10.0.2.3 <========== added by userspace   
                                                 /32 link UNICAST                           
                                           |-- 10.0.2.15   <========== added by userspace   
                                              /32 host LOCAL                                
                                        |-- 10.0.2.255     <========== added by userspace   
                                           /32 link BROADCAST                               
                                  +-- 127.0.0.0/8 2 0 2                                     
                                     +-- 127.0.0.0/31 1 0 0                                 
                                        |-- 127.0.0.0      <========== added by userspace   
                                           /8 host LOCAL                                    
                                        |-- 127.0.0.1      <========== added by userspace   
                                           /32 host LOCAL                                   
                                     |-- 127.255.255.255   <========== added by userspace   
                                        /32 link BROADCAST                                  
 point to 'Main' ==========> Local:                                                         
                             	...                                                           
```

```
+--------------------+
| fib_inetaddr_event |
+----|---------------+
     |
     |--> if event is NETDEV_UP
     |
     |        +----------------+
     |        | fib_add_ifaddr |
     |        +----------------+
     |
     +--> else if event if NETDEV_DOWN

              +----------------+
              | fib_del_ifaddr |
              +----------------+
```

```
+----------------+                                                          
| fib_add_ifaddr |                                                          
+---|------------+                                                          
    |    +-----------+                                                      
    +--> | fib_magic |                                                      
         +--|--------+                                                      
            |    +---------------+                                          
            |--> | fib_new_table |                                          
            |    +---|-----------+                                          
            |        |    +---------------+                                 
            |        |--> | fib_get_table |                                 
            |        |    +---------------+                                 
            |        |                                                      
            |        |--> return table if found                             
            |        |                                                      
            |        |    +----------------+                                
            |        |--> | fib_trie_table | allocate and set up a fib table
            |        |    +----------------+                                
            |        |                                                      
            |        +--> add to hash table for later search                
            |                                                               
            |--> if command is NEW_ROUTE                                    
            |                                                               
            |        +------------------+                                   
            |        | fib_table_insert |                                   
            |        +------------------+                                   
            |                                                               
            +--> else                                                       
                                                                            
                     +------------------+                                   
                     | fib_table_delete |                                   
                     +------------------+                                   
```

For transmit, **ip_queue_xmit** is the entry point from the TCP layer. 
It builds the packet's IP header, performs fragmentation if the size exceeds MTU, and transfers the packet to the network device driver.

```
+---------------+
| ip_queue_xmit |
+---|-----------+
    |    +-----------------+
    +--> | __ip_queue_xmit |
         +----|------------+
              |    +----------------+
              |--> | __sk_dst_check |
              |    +----------------+
              |
              |--> build ip header
              |    1. set protocol to tcp
              |    2. set src & dst addr
              |
              |    +--------------+
              +--> | ip_local_out |
                   +---|----------+
                       |    +------------+
                       +--> | dst_output |
                            +--|---------+
                               |
                               +--> call ->output()
                                          +-----------+
                                    e.g., | ip_output |
                                          +--|--------+
                                             |    +------------------+
                                             +--> | ip_finish_output |
                                                  +----|-------------+
                                                       |    +--------------------+
                                                       +--> | __ip_finish_output |
                                                            +----|---------------+
                                                                 |    +----------------+
                                                                 |--> | ip_skb_dst_mtu |
                                                                 |    +----------------+
                                                                 |
                                                                 |--> fragment the packet if it's larger than mtu
                                                                 |
                                                                 |    +-------------------+
                                                                 +--> | ip_finish_output2 |
                                                                      +----|--------------+
                                                                           |    +-----------------+
                                                                           |--> | ip_neigh_for_gw |
                                                                           |    +-----------------+
                                                                           |    +--------------+
                                                                           +--> | neigh_output |
                                                                                +---|----------+
                                                                                    |    +-----------------+
                                                                                    +--> | neigh_hh_output | 
                                                                                         +-----------------+
                                                                                         connect to driver layer
```

The function **ip_rcv** comes from the NAPI mechanism, and it places the packet onto receive queue of the target socket. 
Every time users read, it eventually triggers **tcp_recvmsg** and picks up the packet from the queue.

```
+--------+                                                                                   
| ip_rcv |                                                                                   
+-|------+                                                                                   
  |    +-------------+                                                                       
  |--> | ip_rcv_core |                                                                       
  |    +---|---------+                                                                       
  |        |                                                                                 
  |        |--> drop packet if it's not for us                                               
  |        |                                                                                 
  |        +--> locate transport header                                                      
  |                                                                                          
  |    +---------------+                                                                     
  +--> | ip_rcv_finish |                                                                     
       +---|-----------+                                                                     
           |    +--------------------+                                                       
           +--> | ip_rcv_finish_core | determine packet direction (deliver to tcp layer or forward to other host)
                +--------------------+                                                       
                +-----------+                                                                
                | dst_input |                                                                
                +--|--------+                                                                
                   |                                                                         
                   +--> call ->input()                                                       
                              +------------------+                                           
                        e.g., | ip_local_deliver | (if decided to deliver to tcp layer)
                              +----|-------------+                                           
                                   |                                                         
                                   |--> reassemble the ip fragments if necessary                         
                                   |                                                         
                                   |    +-------------------------+                          
                                   +--> | ip_local_deliver_finish |                          
                                        +------|------------------+                          
                                               |                                             
                                               |--> adjust data ptr to tcp header (strip ip header)         
                                               |                                             
                                               |--> get next protocol (e.g., tcp) from packet
                                               |                                             
                                               |    +-------------------------+              
                                               +--> | ip_protocol_deliver_rcu |              
                                                    +-------------------------+              
                                                                                             
                                                            call ->handler()                 
                                                                  +------------+             
                                                            e.g., | tcp_v4_rcv | change tcp state or queue skb
                                                                  +------------+             
```





```
+-----------------+                                                                                         
| neigh_hh_output |                                                                                         
+----|------------+                                                                                         
     |    +----------------+                                                                                
     +--> | dev_queue_xmit |                                                                                
          +---|------------+                                                                                
              |    +------------------+                                                                     
              +--> | __dev_queue_xmit |                                                                     
                   +----|-------------+                                                                     
                        |    +---------------------+                                                        
                        |--> | netdev_core_pick_tx | select tx queue                                        
                        |    +---------------------+                                                        
                        |    +---------------------+                                                        
                        +--> | dev_hard_start_xmit |                                                        
                             +-----|---------------+                                                        
                                   |                                                                        
                                   +--> for each skb                                                        
                                                                                                            
                                            +----------+                                                    
                                            | xmit_one |                                                    
                                            +--|-------+                                                    
                                               |    +-------------------+                                   
                                               +--> | netdev_start_xmit |                                   
                                                    +----|--------------+                                   
                                                         |    +---------------------+                       
                                                         +--> | __netdev_start_xmit |                       
                                                              +-----|---------------+                       
                                                                    |                                       
                                                                    +--> call ->ndo_start_xmit()            
                                                                               +---------------------------+
                                                                         e.g., | ftgmac100_hard_start_xmit |
                                                                               +---------------------------+
```

## <a name="network-interface-layer"></a> Network Interface Layer

After receiving skb from the Internet layer, the network device driver fills the TX descriptor(s) and triggers the register for hardware to take action.

```
+---------------------------+                                              
| ftgmac100_hard_start_xmit |                                              
+------|--------------------+                                              
       |    +----------------+                                             
       |--> | dma_map_single |  map the data so the hardware can access it?
       |    +----------------+                                             
       |                                                                   
       |--> get the next available tx descriptor and set up it             
       |                                                                   
       |--> set up extra tx descriptors if skb has fragments               
       |                                                                   
       +--> trigger the hardware to read the updated tx descriptors        
```

```
+---------------------+                                                                                          
| ftgmac100_interrupt |                                                                                          
+-----|---------------+                                                                                          
      |    +--------------------+                                                                                
      |--> | napi_schedule_prep | label SCHED in napi_struct                                                     
      |    +--------------------+                                                                                
      |                                                                                                          
      +--> if SCHED is labeled by us                                                                             
                                                                                                                 
               +------------------------+                                                                        
               | __napi_schedule_irqoff |                                                                        
               +-----|------------------+                                                                        
                     |    +-------------------+                                                                  
                     +--> | ____napi_schedule |                                                                  
                          +----|--------------+                                                                  
                               |                                                                                 
                               |--> add the napi_struct to the end of list 'softnet_data'                        
                               |                                                                                 
                               |    +------------------------+                                                   
                               +--> | __raise_softirq_irqoff | label NET_RX and somewhere will handle it properly
                                    +------------------------+                                                   
```

## <a name="boot-flow"></a> Boot Flow

During boot up flow, a few init calls register the network family by function **sock_register()** to add the supported socket type on the system.
Sometimes we might see AF_OOO instead of PF_OOO, but they are the equivalent.

```
+--------------------+     +---------------+           
| netlink_proto_init | --> | sock_register | PF_NETLINK
+--------------------+     +---------------+           
+-----------+              +---------------+           
| inet_init | -----------> | sock_register | PF_INET   
+-----------+              +---------------+           
+--------------+           +---------------+           
| af_unix_init | --------> | sock_register | PF_UNIX   
+--------------+           +---------------+           
+-------------+            +---------------+           
| af_alg_init | ---------> | sock_register | PF_ALG    
+-------------+            +---------------+           
+------------+             +---------------+           
| inet6_init | ----------> | sock_register | PF_INET6  
+------------+             +---------------+           
+-------------+            +---------------+           
| packet_init | ---------> | sock_register | PF_PACKET 
+-------------+            +---------------+           
```

## <a name="to-do-list"></a> To-Do List

- Introduce fib and neighbor in IP layer

## <a name="reference"></a> Reference

- [TCP Server-Client implementation in C](https://www.geeksforgeeks.org/tcp-server-client-implementation-in-c/)
- [IPv4 route lookup on Linux](https://vincent.bernat.ch/en/blog/2017-ipv4-route-lookup-linux)
