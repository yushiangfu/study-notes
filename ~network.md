> Study case: Linux version 5.15.0 on AST2500 emulation

## Index

- [Introduction](#introduction)
- [Network Layers & Families](#network-layers-and-families)
- [Application Layer](#application-layer)
- [Transport Layer](#transport-layer)
- [Internet Layer](#internet-layer)
- [Network Interface Layer](#network-interface-layer)
- [Boot Flow](#boot-flow)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

```          
               OSI Model            TCP/IP Model  
                                                  
           +--------------+       +--------------+
         7 |  Application |       |              |
           +--------------+       |              |
         6 | Presentation |       |  Application |
           +--------------+       |              |
         5 |    Session   |       |              |
           +--------------+       +--------------+
         4 |   Transport  |       |   Transport  |
           +--------------+       +--------------+
 router  3 |    Network   |       |    Network   |
           +--------------+       +--------------+
 switch  2 |   Data Link  |       |              |
           +--------------+       |    Network   |
         1 |   Physical   |       |   Interface  |
           +--------------+       +--------------+
```

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

This syscall allocates a socket and installs corresponding operations based on the specified arguments. 
These operations determine how the application layer connects the transport layer and the internet layer. 
Later it prepares a file representing the socket so the users can operate it by general file operations.

```
  server

 +------+
 | file |
 +------+
     |
     |
+--------+
| socket | ops = inet_stream_ops
+--------+ proto = tcp_prot
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
            |                                                +--> install ipv4 operations 'ipv4_specific'
            |
            |    +-------------+
            +--> | sock_map_fd | allocate a file handle for the socket, and install it to fd table
                 +-------------+                                                                        
```
</details>
  
### setsockopt()

It's optional.

<details>
  <summary> Code Trace </summary>

```
+----------------+                                     
| sys_setsockopt |                                     
+---|------------+                                     
    |    +------------------+                          
    +--> | __sys_setsockopt |                          
         +----|-------------+                          
              |                                        
              |--> if level is SOL_SOCKET              
              |                                        
              |        +-----------------+             
              |------> | sock_setsockopt |             
              |        +-----------------+             
              |                                        
              |--> else                                
              |                                        
              +------> call ->setsockopt()             
                             +------------------------+
                       e.g., | sock_common_setsockopt |
                             +------------------------+
```
  
</details>
  
### bind()

TCP and UDP protocols introduce the port concept in the transport layer, different from the physical port that connects to the network cable. 
Every service occupies at least one port when it sets up the server with the help of **bind()**. 
The common TCP ports are 80 (HTTP), 8080 (HTTPS), 22 (SSH), 53 (DNS), etc. 
The caller can opt not to provide the port, and **bind()** will prepare a valid one in that case.

```
  server

 +------+
 | file |
 +------+
     |
     |
+--------+
| socket | ops = inet_stream_ops
+--------+ proto = tcp_prot
     |
     |
 +------+
 | port |
 +------+                  
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
                                             +-------------------+
                                       e.g., | inet_csk_get_port | 
                                             +-------------------+
                                                  |
                                                  |--> if no given port, find a valid one and return
                                                  |
                                                  +--> check if given port is valid
```
  
</details>

### listen()

In TCP design, the socket has different states indicating its current status. 
The function **listen** will change the socket state accordingly and add it to the hash table, waiting for the client to connect. 
We can use the utility **netstat** to display system socket status, and I guess the information comes from the hash table.

```
                     server

                    +------+
                    | file |
                    +------+
                        |
                        |
                   +--------+
                   | socket | ops = inet_stream_ops
                   +--------+ proto = tcp_prot
                        |
                        |
 hash table         +------+
+-+--------+        | port |
+-+--------+        +------+
+-+--------+
+-+--------+
+-+--------+
```

<details>
  <summary> Code Trace </summary>

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

</details>
  
### connect()

Clients call this function with server address and port information to build the packet and send it to the destination. 
Client port is required for the connection establishment, though it could be random. 
In TCP design, re-sending is possible if the sender receives no response from the target in a specified interval. 
The server side will receive the packet and locate the corresponding socket by server address and port through the hash table. 
The client socket will change the socket state to CONNECTED after learning the acceptance from the server.

```
  client                                                    server

 +------+                                                  +------+
 | file |                                                  | file |
 +------+                                                  +------+
     |                                                         |
     |                                                         |
+--------+                                                +--------+
| socket |                                                | socket | ops = inet_stream_ops
+--------+ ------------------------+                  +-> +--------+ proto = tcp_prot
     |                             |                  |        |
     |                             |                  |        |
 +------+                          |    hash table    |    +------+
 | port |                          |   +-+--------+   |    | port |
 +------+                          |   +-+--------+   |    +------+
                                   +-> +-+--------+ --+
                                       +-+--------+
                                       +-+--------+
```

<details>
  <summary> Code Trace </summary>

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
             |              +----|---------------+
             |                   |
             |                   +--> call ->queue_xmit()
             |                              +---------------+
             |                        e.g., | ip_queue_xmit |
             |                              +---------------+
             |    +---------------------------+
             +--> | inet_csk_reset_xmit_timer | re-send the packet if no response before timeout
                  +---------------------------+
```

</details>
  
### accept()

After the client connects to the server, it waits for acceptance.
Once server decides to accept the request, it prepares another pair of socket and file for real data transmission.
Till this stage, both sides have been through the famous three-way handshake of TCP and officially establish the connection.

```
  client                                                    server

 +------+                                +------+          +------+
 | file |                                | file |          | file |
 +------+                                +------+          +------+
     |                                       |                 |
     |                                       |                 |
+--------+                              +--------+        +--------+
| socket | <--------------------------> | socket |        | socket | ops = inet_stream_ops
+--------+ ------------------------+    +--------+    +-> +--------+ proto = tcp_prot
     |                             |                  |        |
     |                             |                  |        |
 +------+                          |    hash table    |    +------+
 | port |                          |   +-+--------+   |    | port |
 +------+                          |   +-+--------+   |    +------+
                                   +-> +-+--------+ --+
                                       +-+--------+
                                       +-+--------+
```

<details>
  <summary> Code Trace </summary>

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
                      +--> | do_accept |
                      |    +--|--------+
                      |       |    +------------+
                      |       +--> | sock_alloc | allocate socket for the connection
                      |       |    +------------+
                      |       |    +-----------------+
                      |       +--> | sock_alloc_file | allocate file for the connection
                      |       |    +-----------------+
                      |       |
                      |       +--> call ->accept()
                      |       |          +-------------+
                      |       |    e.g., | inet_accept | wait for connection, set socket state to CONNECTED
                      |       |          +---|---------+
                      |       |              |
                      |       |              +--> call ->accept()
                      |       |                         +-----------------+
                      |       |                   e.g., | inet_csk_accept |
                      |       |                         +----|------------+
                      |       |                              |
                      |       |                              +--> if no connection request in queue
                      |       |                              |
                      |       |                              |        +---------------------------+
                      |       |                              +------> | inet_csk_wait_for_connect |
                      |       |                              |        +---------------------------+
                      |       |                              |    +--------------------+
                      |       |                              +--> | reqsk_queue_remove | remove a request
                      |       |                                   +--------------------+
                      |       |
                      |       +--> copy address info to userspace if asked to
                      |       |
                      |       +--> return allocated file
                      |
                      |    +------------+
                      +--> | fd_install | install the file to allocated fd
                           +------------+
```
  
</details>
  
### write()

Each side is free to write data to its socket, and the data goes through the normal flow of 'file write' before reaching the transport layer. 
TCP then prepares the socket buffer (SKB), copies data onto it, builds the TCP header in SKB, and sends it to the Internet layer.

```
                     client                                                    server

                    +------+                                +------+          +------+
                    | file |                                | file |          | file |
                    +------+                                +------+          +------+
                        |                                       |                 |
                        |                                       |                 |
                   +--------+                              +--------+        +--------+
                   | socket | <--------------------------> | socket |        | socket | ops = inet_stream_ops
                   +--------+ ------------------------+    +--------+    +-> +--------+ proto = tcp_prot
                        |                             |                  |        |
                        |                             |                  |        |
                    +------+                          |    hash table    |    +------+
                    | port |                          |   +-+--------+   |    | port |
                    +------+                          |   +-+--------+   |    +------+
                                                      +-> +-+--------+ --+
                                                          +-+--------+
                                                          +-+--------+


                     packet
               |   +---------+
from app layer |   | tcp hdr |
               +-> +---------+
                   |         |
               +-- |  data   |
   to ip layer |   |         |
               v   +---------+
```

<details>
  <summary> Code Trace </summary>

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
           |                                   e.g., | sock_write_iter | (refer to the below)
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
                                            e.g., | tcp_sendmsg | (refer to the below)
                                                  +-------------+
```

```
+-------------+
| tcp_sendmsg | copy data to skb and send to ip layer
+---|---------+
    |
    +--> while there's data to be sent
    |
    |        +---------------------+
    +------> | sk_stream_alloc_skb | allcoate struct sk_buff for buffer management
    |        +---------------------+
    |        +---------------------+
    +------> | sk_page_frag_refill | allocate a few pages as buffer
    |        +---------------------+
    |        +--------------------------+
    +------> | skb_copy_to_page_nocache | copy data to the buffer
    |        +--------------------------+
    |        +--------------------+
    +------> | skb_fill_page_desc | file the buffer descriptor in struct sk_buff
    |        +--------------------+
    |
    +------> if not data left to be sent, exit the loop
    |
    +--> if we did copy data to socket buffer
    |
    |        +----------+
    +------> | tcp_push |
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
                                                    +----|---------------+
                                                         |
                                                         +--> call ->queue_xmit()
                                                                    +---------------+
                                                              e.g., | ip_queue_xmit |
                                                                    +---------------+
```
  
</details>
  
### read()

The driver triggers the NAPI mechanism whenever the network interface hardware receives the packet. 
That mechanism eventually places the partially stripped packet onto the receive queue in the TCP layer. 
Every time users read, it triggers **tcp_recvmsg** and picks up the packet. 

```
                     client                                                    server

                    +------+                                +------+          +------+
                    | file |                                | file |          | file |
                    +------+                                +------+          +------+
                        |                                       |                 |
                        |                                       |                 |
                   +--------+                              +--------+        +--------+
                   | socket | <--------------------------> | socket |        | socket | ops = inet_stream_ops
                   +--------+ ------------------------+    +--------+    +-> +--------+ proto = tcp_prot
                        |                             |                  |        |
                        |                             |                  |        |
                    +------+                          |    hash table    |    +------+
                    | port |                          |   +-+--------+   |    | port |
                    +------+                          |   +-+--------+   |    +------+
                                                      +-> +-+--------+ --+
                                                          +-+--------+
                                                          +-+--------+


                     packet                                                    packet
               |   +---------+                                               +---------+   |
from app layer |   | tcp hdr |                                               | tcp hdr |   | tcp layer: get
               +-> +---------+   write                               read    +---------+ <-+
                   |         |  ----------------------------------------->   |         |
               +-- |  data   |                                               |  data   | <-+
   to ip layer |   |         |                                               |         |   | tcp layer: put
               v   +---------+                                               +---------+   |
```

<details>
  <summary> Code Trace </summary>

VFS layer

```
+----------+
| sys_read |
+--|-------+
   |    +-----------+
   +--> | ksys_read |
        +--|--------+
           |    +-----------+
           +--> | file_ppos | get file offset
           |    +-----------+
           |    +----------+
           +--> | vfs_read |
           |    +--|-------+
           |       |
           |       +--> if ->read exists
           |       |
           |       |        call ->read()
           |       |
           |       +--> else if ->read_iter exists
           |       |
           |       |        +---------------+
           |       +------> | new_sync_read |
           |                +---|-----------+
           |                    |    +----------------+
           |                    +--> | call_read_iter |
           |                         +---|------------+
           |                             |
           |                             +--> call ->read_iter()
           |                                        +----------------+
           |                                  e.g., | sock_read_iter | (refer to the below)
           |                                        +----------------+
           |
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
                                           e.g., | tcp_recvmsg | (refer to the below)
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
              |
              +------> if receive queue has skb(s)                     --+
              |                                                          |
              |            +-----------------------+                     |
              +----------> | skb_copy_datagram_msg |                     |
              |            +-----------------------+                     |
              |                                                          | extremely simplified logic
              +------> else                                              |
              |                                                          |
              |            +--------------+                              |
              +----------> | sk_wait_data | wait for data to arrive      |
                           +--------------+                            --+                      
```

Function **tcp_v4_rcv** is responsible for placing data onto the receive queue, so **tcp_recvmsg** can later retrieve data or wait till there's something.

```
+------------+                                                                                               
| tcp_v4_rcv |                                                                                               
+--|---------+                                                                                               
   |    +-------------------+                                                                                
   |--> | __inet_lookup_skb | lookup target socket through address information                               
   |    +-------------------+                                                                                
   |                                                                                                         
   |--> if socket state is LISTEN                                                                            
   |                                                                                                         
   |        +---------------+                                                                                
   +------> | tcp_v4_do_rcv |                                                                                
            +---|-----------+                                                                                
                |                                                                                            
                |--> if state is ESTABLISHED                                                                 
                |                                                                                            
                |        +---------------------+                                                             
                |------> | tcp_rcv_established |                                                             
                |        +-----|---------------+                                                             
                |              |    +---------------+                                                        
                |              |--> | tcp_queue_rcv | add skb to the receive queue of the socket             
                |              |    +---------------+                                                        
                |              |    +----------------+                                                       
                |              +--> | tcp_data_ready |                                                       
                |                   +---|------------+                                                       
                |                       |                                                                    
                |                       +--> call ->sk_data_ready                                            
                |                                  +-------------------+                                     
                |                            e.g., | sock_def_readable | wake up the task waiting on the data
                |                                  +-------------------+                                     
                |    +-----------------------+                                                               
                +--> | tcp_rcv_state_process | handle the tcp state change and packet reply                  
                     +-----------------------+                                                               
```
  
</details>
  
### sys_close()

(TBD)

## <a name="internet-layer"></a> Internet Layer

If we specify TCP as our transport layer, it must be Internet Protocol (IP) as the network layer whether we set it. 
This layer dictates the direction of packets:
- output: send to network interface layer for packet transmission
- input: send transport layer
- forward: this is probably the pervasive situation for those middle stops on the route

```
 transport       |          ^                          
   layer         |          |                          
                 |          |                          
                 |          |                          
      -----------|----------|-----------------------   
                 |          |                          
                 |          |                          
 internet        |output    |input    forward          
   layer         |          |          +---+           
                 |          |          |   |           
                 |          |          |   |           
      -----------|----------|----------|---|--------   
                 |          |          |   |           
  network        |          |          |   |           
 interface       |          |          |   |           
   layer         v          |          |   v           
```

As we can observe from the above TCP layer, only **connect()** and **write()** involve the IP layer through **tcp_transmit_skb()** for sending the packet out. 
For **read()** action, the IP layer triggers the TCP handler to place data on the socket's receive queue.

| TCP Layer                | IP Layer                         |
| ---                      | ---                              |
| tcp_v4_connect           | ip_route_connect & ip_queue_xmit |
| tcp_sendmsg              | ip_queue_xmit                    |
| tcp_recvmsg & tcp_v4_rcv | ip_rcv                           |

Once the kernel finishes the boot and transfers the control to the systemd, one of the userspace utility sets up the route information. 
How it gets the data is another story I don't know yet. 
The kernel knows which gateway's IP address to use when building the packet header with this knowledge.

```
root@romulus:~# cat /proc/net/route 
Iface   Destination     Gateway         Flags   RefCnt  Use     Metric  Mask            MTU     Window  IRTT
eth0    00000000        0202000A        0003    0       0       1024    00000000        0       0       0
               
eth0    0002000A        00000000        0001    0       0       1024    00FFFFFF        0       0       0
               
eth0    0202000A        00000000        0005    0       0       1024    FFFFFFFF        0       0       0
               
eth0    0302000A        00000000        0005    0       0       1024    FFFFFFFF        0       0       0
             
```

We can also utilize the utility **route** to display the pretty format.

```
root@romulus:~# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         10.0.2.2        0.0.0.0         UG    1024   0        0 eth0
10.0.2.0        *               255.255.255.0   U     1024   0        0 eth0
10.0.2.2        *               255.255.255.255 UH    1024   0        0 eth0
10.0.2.3        *               255.255.255.255 UH    1024   0        0 eth0                                    
```

The function **ip_route_connect** does the gateway IP address lookup, which is essential when building the IP header.

<details>
  <summary> Code Trace </summary>

```
+------------------+                                                                                               
| ip_route_connect |                                                                                               
+----|-------------+                                                                                               
     |    +-----------------------+                                                                                
     |--> | ip_route_connect_init | init flowi4                                                                    
     |    +-----------------------+                                                                                
     |    +----------------------+                                                                                 
     +--> | ip_route_output_flow |                                                                                 
          +-----|----------------+                                                                                 
                |    +-----------------------+                                                                     
                +--> | __ip_route_output_key |                                                                     
                     +-----|-----------------+                                                                     
                           |    +--------------------------+                                                       
                           +--> | ip_route_output_key_hash |                                                       
                                +------|-------------------+                                                       
                                       |    +------------------------------+                                       
                                       +--> | ip_route_output_key_hash_rcu |                                       
                                            +-------|----------------------+                                       
                                                    |                                                              
                                                    |--> try to determine output interface, src addr, dst addr     
                                                    |                                                              
                                                    |    +------------+                                            
                                                    |--> | fib_lookup | lookup fib based on dst addr               
                                                    |    +------------+                                            
                                                    |    +------------------+                                      
                                                    +--> | __mkroute_output |                                      
                                                         +----|-------------+                                      
                                                              |    +--------------+                                
                                                              |--> | rt_dst_alloc | set up rtable and install ops  
                                                              |    +--------------+ ->output(): ip_output            
                                                              |                     ->input(): ip_local_deliver      
                                                              |                                                    
                                                              |    +----------------+                              
                                                              +--> | rt_set_nexthop | save gw ip from fib to rtable
                                                                   +----------------+
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

</details>
  
For transmit, **ip_queue_xmit** is the entry point from the TCP layer. 
It builds the packet's IP header, performs fragmentation if the size exceeds MTU, and transfers the packet to the network device driver.

```
output packet:

+---------+                                    
|  ip hdr | src & dst ip, higher protocol (tcp)
+---------+                                    
| tcp hdr | src & dst port                               
+---------+                                    
|         |                                    
|  data   |                                    
|         |                                    
+---------+                                    
```

<details>
  <summary> Code Trace </summary>

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
                                                                           |    get 'neighbor' struct throug gw ip
                                                                           |
                                                                           |    +--------------+
                                                                           +--> | neigh_output |
                                                                                +---|----------+
                                                                                    |    +----------------------+
                                                                                    +--> | neigh_resolve_output | 
                                                                                         +----------------------+
                                                                                         connect to driver layer
```
  
</details>

Coming from the NAPI mechanism, **ip_rcv** analyzes the IP header and knows which handler to call by inspecting field **protocol**.
By adjusting the skb pointers, the higher layer has no clue about the IP header as if it's stripped.

```
input packet:

+---------+                                     
|  ip hdr | src & dst ip, higher protocol (tcp) 
+---------+  --+                                
|         |    |                                
|         |    |                                
|   ???   |    | this is what tcp layer receives
|         |    |                                
|         |    |                                
+---------+  --+                                                               
```

<details>
  <summary> Code Trace </summary>

```
+--------+
| ip_rcv |
+-|------+
  |    +-------------+
  +--> | ip_rcv_core |
  |    +---|---------+
  |        |
  |        +--> drop packet if it's not for us
  |        |
  |        +--> locate transport header
  |
  |    +---------------+
  +--> | ip_rcv_finish |
       +---|-----------+
           |    +--------------------+
           +--> | ip_rcv_finish_core | determine packet direction (deliver to tcp layer or forward to other host)
           |    +--------------------+
           |    +-----------+
           +--> | dst_input |
                +--|--------+
                   |
                   +--> call ->input()
                              +------------------+
                        e.g., | ip_local_deliver | (if decided to deliver to tcp layer)
                              +----|-------------+
                                   |
                                   +--> reassemble the ip fragments if necessary
                                   |
                                   |    +-------------------------+
                                   +--> | ip_local_deliver_finish |
                                        +------|------------------+
                                               |
                                               +--> adjust data ptr to tcp header (strip ip header)
                                               |
                                               +--> get next protocol (e.g., tcp) from packet
                                               |
                                               |    +-------------------------+
                                               +--> | ip_protocol_deliver_rcu |
                                                    +------|------------------+
                                                           |
                                                           +--> call ->handler()
                                                                      +------------+
                                                                e.g., | tcp_v4_rcv | change tcp state or queue skb
                                                                      +------------+         
```
  
</details>

## <a name="network-interface-layer"></a> Network Interface Layer

After receiving skb from the Internet layer, the network device driver builds the MAC header, fills the TX descriptor(s) and ask hardware to take action.

```
+---------+                                    
| mac hdr | src & dst mac, higher protocol (ip)
+---------+                                    
|  ip hdr | src & dst ip, higher protocol (tcp)
+---------+                                    
| tcp hdr | src & dst port
+---------+                                    
|         |                                    
|  data   |                                    
|         |                                    
+---------+                                    
```

<details>
  <summary> Code Trace </summary>

```
+--------------+
| neigh_output |
+---|----------+
    |
    +--> call ->output()
               +----------------------+
         e.g., | neigh_resolve_output |
               +-----|----------------+
                     |    +-----------------+
                     |--> | dev_hard_header |
                     |    +----|------------+
                     |         |
                     |         +--> call ->create()
                     |                    +------------+
                     |              e.g., | eth_header | build mac header
                     |                    +------------+
                     |    +----------------+
                     +--> | dev_queue_xmit |
                          +----------------+
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
                                                   |
                                                   |        +----------+
                                                   +------> | xmit_one |
                                                            +--|-------+
                                                               |    +-------------------+
                                                               +--> | netdev_start_xmit |
                                                                    +----|--------------+
                                                                         |    +---------------------+
                                                                         +--> | __netdev_start_xmit |
                                                                              +-----|---------------+
                                                                                    |
                                                                                    +--> call ->ndo_start_xmit(), e.g.,
                                                                                         +---------------------------+
                                                                                         | ftgmac100_hard_start_xmit |
                                                                                         +---------------------------+
```

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

</details>

For packet receiving, the registered ISR schedules the NAPI struct of the driver to process the ingress packets. 
Conventional interrupt mechanism is triggered by hardware components, notifying kernel some events happen and need the ISR to handle them. 
This method saves more effort than the polling mechanism, which wastes the system resource if hardware event rarely shows. 
However, modern network interface cards (NIC) support high-speed bandwidth and can quickly generate network interrupts storm. 
NAPI is the mechanism introduced to solve this problem by mixing interrupt and polling methods. 
Once a network interrupt happens, somewhere disables the NIC interrupt, and the polling method gets in the way. 
The polling function registered by the NIC driver continues to receive the packet if there's any, hence saving the system from suffering high-frequency interrupt. 
Once no more packets arrive, it switches back to the interrupt mechanism.

```
                                                                                                
               interrupt                                                                        
             ------------>                                                                      
                           disable interrupt, schedule napi of the driver | hw interrupt handler
                                                                                                
+--------+                 napi->poll and send to higher layer            |                     
| +----+ |                                                                |                     
| | hw | |                 napi->poll and send to higher layer            |                     
| +----+ |                                                                | sw interrupt handler
+--------+                 napi->poll and send to higher layer            |                     
                                                                          |                     
                           enable interrupt                               |                     
                                                                                                
               interrupt                                                                        
             ------------>                                                                      
                           (repeat)                                                             
```

The **poll** function learns the packet type by inspecting the field **protocol**, and knows which handler will help transfer the packet to the higher layer.

```
+---------+                                     
| mac hdr | src & dst mac, higher protocol (ip) 
+---------+  --+                                
|         |    |                                
|         |    |                                
|         |    |                                
|   ???   |    | this is what ip layer receives
|         |    |                                
|         |    |                                
|         |    |                                
+---------+  --+                                
```

<details>
  <summary> Code Trace </summary>

```
+---------------------+
| ftgmac100_interrupt |
+-----|---------------+
      |    +----------------------+
      +--> | napi_schedule_irqoff |
           +-----|----------------+
                 |    +--------------------+
                 +--> | napi_schedule_prep | label SCHED in napi_struct
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
                                          +--> | __raise_softirq_irqoff | label NET_RX and somewhere will handle it
                                               +------------------------+                                                 
```

```
+---------------+                                                                                 
| net_rx_action | handle the list of napi_struct (not guaranteed to process them all)
+---|-----------+                                                                                 
    |                                                                                             
    |--> splice percpu list into local one                                                        
    |                                                                                             
    |--> endless loop                                                                             
    |                                                                                             
    |------> exit loop if the list is empty                                                       
    |                                                                                             
    |------> get the first napi_struct in list                                                    
    |                                                                                             
    |        +-----------+                                                                        
    |------> | napi_poll |                                                                        
    |        +--|--------+                                                                        
    |           |                                                                                 
    |           |--> remove napi_struct from list                                                 
    |           |                                                                                 
    |           |    +-------------+                                                              
    |           |--> | __napi_poll |                                                              
    |           |    +---|---------+                                                              
    |           |        |                                                                        
    |           |        |--> if the napi_struct is labeled SCHED                                 
    |           |        |                                                                        
    |           |        |------> call ->poll()                                                   
    |           |        |              +----------------+                                        
    |           |        |        e.g., | ftgmac100_poll | send packets to higher layer (e.g., ip)
    |           |        |              +----------------+ restore hardware interrupts            
    |           |        |                                                                        
    |           |        |--> return if no work left                                              
    |           |        |                                                                        
    |           |        +--> or inform the caller to re-poll, and then return                    
    |           |                                                                                 
    |           +--> if need re-poll, add napi_struct back to the list                            
    |                                                                                             
    +--> splice the re-poll list back to the percpu list                                          
```

```
+----------------+                                                                                                 
| ftgmac100_poll | send packets to higher layer (e.g., ip), restore hardware interrupts                            
+---|------------+                                                                                                 
    |    +-----------------------+                                                                                 
    |--> | ftgmac100_tx_complete | unmap complete packets and release their skbs                                   
    |    +-----------------------+                                                                                 
    |                                                                                                              
    |--> while workload remains and we have budget to handle it                                                    
    |                                                                                                              
    |        +---------------------+                                                                               
    |        | ftgmac100_rx_packet |                                                                               
    |        +-----|---------------+                                                                               
    |              |                                                                                               
    |              |--> get pre-allocated skb & unmap dma mapping                                                  
    |              |                                                                                               
    |              |    +------------------------+                                                                 
    |              |--> | ftgmac100_alloc_rx_buf | allocate skb and map dma mapping for next time                  
    |              |    +------------------------+                                                                 
    |              |                                                                                               
    |              |--> get protocol (e.g., IP) from packet                                                        
    |              |                                                                                               
    |              |    +-------------------+                                                                      
    |              +--> | netif_receive_skb | determine packet type (e.g., ip) and call its ->func() (e.g., ip_rcv)
    |                   +-------------------+                                                                      
    |                                                                                                              
    |--> if we finish the workload                                                                                 
    |                                                                                                              
    |        +---------------+                                                                                     
    |------> | napi_complete | clear the state of napi_struct                                                      
    |        +---------------+                                                                                     
    |                                                                                                              
    +------> enable hardware interrupts                                                                            
```

```
+-------------------+                                                                           
| netif_receive_skb | determine packet type (e.g., ip) and call its ->func() (e.g., ip_rcv)
+----|--------------+                                                                           
     |    +----------------------------+                                                        
     +--> | netif_receive_skb_internal |                                                        
          +------|---------------------+                                                        
                 |    +---------------------+                                                   
                 +--> | __netif_receive_skb |                                                   
                      +-----|---------------+                                                   
                            |    +------------------------------+                               
                            +--> | __netif_receive_skb_one_core |                               
                                 +-------|----------------------+                               
                                         |    +--------------------------+                      
                                         |--> | __netif_receive_skb_core | determine packet type
                                         |    +--------------------------+                      
                                         |                                                      
                                         +--> call ->func()                                     
                                                    +--------+                                  
                                              e.g., | ip_rcv |                                  
                                                    +--------+                                  
```

Function **net_rx_action** has its counterpart named **net_tx_action**, responsible for skb cleaning after transmission.

```
+---------------+                                               
| net_tx_action |                                               
+---|-----------+                                               
    |                                                           
    |--> free each skb on completion queue                      
    |                                                           
    +--> run each qdisc on output queue <========== not our case
```

</details>
                                             
Let's summarize the read and write flow from the perspective of the network layer model.

```
                             write                         read                                             
                                                                                                            
                                                                                                            
                         +-----------+                 +----------+                                         
 application             | sys_write |                 | sys_read |                                         
    layer                +-----------+                 +----------+                                         
                               |                             |                                              
               ----------------|-----------------------------v----------------                              
                               v                     +-------------+                                        
                        +-------------+              | tcp_recvmsg | get data from queue                    
  transport             | tcp_sendmsg |              +-------------+                                       -
    layer               +-------------+               | tcp_v4_rcv | put data onto queue                    
                               |                      +------------+                                        
               ----------------|-----------------------------^----------------                              
                               v                             |                                              
                       +---------------+                +--------+                                          
  internet             | ip_queue_xmit |                | ip_rcv |                                          
    layer              +---------------+                +--------+                                          
                               |                             ^                                              
               ----------------|-----------------------------|----------------                              
                               v                             |                                              
   network       +---------------------------+    +---------------------+                                   
  interface      | ftgmac100_hard_start_xmit |    | ftgmac100_interrupt |                                   
    layer        +---------------------------+    +---------------------+                                   
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

## <a name="reference"></a> Reference

- [TCP Server-Client implementation in C](https://www.geeksforgeeks.org/tcp-server-client-implementation-in-c/)
- [IPv4 route lookup on Linux](https://vincent.bernat.ch/en/blog/2017-ipv4-route-lookup-linux)
