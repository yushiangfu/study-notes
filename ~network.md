> Study case: Linux version 5.15.0 on AST2500 emulation

## Index

- [Introduction](#introduction)
- [Application Layer](#application-layer)
- [Transport Layer](#transport-layer)
- [Internet Layer](#internet-layer)
- [Network Interface Layer](#network-interface-layer)
- [Boot Flow](#boot-flow)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

The network stack, originally consisting of seven layers in the OSI model, is typically streamlined to four layers in kernel implementations. 
Despite this simplification, tracing network functions remains challenging due to the abundant use of function pointers resulting from the rigid layer design.

Common protocols like TCP, UDP, IP, ARP, and ICMP belong to the INET family, which is a group within the network stack. 
The INET6 family comprises their IPv6 counterparts. 
Additionally, there are other notable families in the network stack, including UNIX, NETLINK, and PACKET.

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

To avoid confusion, this illustration is based on the TCP/IP model, as evident in the code trace and packet header analysis. 
In network programming, machines that provide services are referred to as servers, while those making requests are called clients.

The kernel offers a set of syscalls that both clients and servers utilize to achieve their respective objectives of requesting and fulfilling services. 
The process begins with both sides utilizing the socket() syscall to create a socket, which acts as a handle for subsequent operations such as establishing connections, reading, and writing.

A server may possess multiple network interfaces with corresponding addresses. 
It utilizes bind() to associate the socket with a specific address and subsequently employs listen() to enter a state of readiness for incoming connections.

On the client side, initiating a connect() syscall is sufficient to notify the server of the intention to establish a connection. 
The server, in response, employs accept() to formally establish the connection.

Once the connection is established, both sides can engage in read and write operations, similar to typical file operations, with the communication taking place over the network.

To conclude the interaction, either or both sides can employ the close() syscall to terminate the communication, signaling the end of the interaction.

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

The transport layer handles the following tasks:

1. Multiplexing and demultiplexing packets from/to multiple applications.
2. Segmenting and reassembling data from/to applications.
3. Connection establishment and termination.
4. Error detection and correction.
5. Quality of service.

Now, let's focus on TCP and explore its underlying functionality through each syscall to serve the application layer.

### socket()

```
prototype:
SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)
```

- `family`
  - In our study case, the network family being used is PF_INET, which refers to the IPv4 protocol family.
- `type`
  - When selecting the PF_INET family, the next step is to choose a type from the following list, and in our case, it is SOCK_STREAM (TCP).
  ```
  SOCK_STREAM = 1,  <-- TCP
  SOCK_DGRAM  = 2,
  SOCK_RAW    = 3,
  SOCK_RDM    = 4,
  SOCK_SEQPACKET  = 5,
  SOCK_DCCP   = 6,
  SOCK_PACKET = 10, 
  ```
- `protocol`
  - When the family is set to PF_INET and the type is specified as TCP, the only valid option for this field is IP.

This syscall allocates a socket and sets up the appropriate operations based on the provided arguments. 
These operations define the interface between the application layer, transport layer, and internet layer. 
The syscall also prepares a file that represents the socket, enabling users to interact with it using general file operations.

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

<details><summary> More Details </summary>

```
net/socket.c                                                                                                           
+------------+                                                                                                          
| sys_socket | : alloc socket pair (generic/specific), install family+type ops, prepare file of socket, install to fdt  
+--------------+                                                                                                        
| __sys_socket | : alloc socket pair (generic/specific), install family+type ops, prepare file of socket, install to fdt
+-|------------+                                                                                                        
  |    +-------------+                                                                                                  
  |--> | sock_create | alloc gen/tcp sockets and relate them, install family+type ops                                   
  |    +-------------+                                                                                                  
  |    +-------------+                                                                                                  
  +--> | sock_map_fd | alloc a file handle for the socket, and install it to fd table                                   
       +-------------+                                                                                                  
```

```
net/socket.c                                                                     
+-------------+                                                                   
| sock_create | : alloc gen/tcp sockets and relate them, install family+type ops  
+---------------+                                                                 
| __sock_create | : alloc gen/tcp sockets and relate them, install family+type ops
+-|-------------+                                                                 
  |    +------------+                                                             
  |--> | sock_alloc | allocate socket                                             
  |    +------------+                                                             
  |                                                                               
  +--> call ->create(), e.g.,                                                     
       +-------------+                                                            
       | inet_create | install family+type ops, prepare tcp socket                
       +-------------+                                                            
```

```
net/ipv4/af_inet.c                                             
+-------------+                                                 
| inet_create | : install family+type ops, prepare tcp socket   
+-|-----------+                                                 
  |                                                             
  |--> install family + type operations, e.g., inet_stream_ops  
  |                                                             
  |--> allocate tcp socket (not the same socket created earlier)
  |                                                             
  +--> call ->init(), e.g.,                                     
       +------------------+                                     
       | tcp_v4_init_sock | init tcp socket                     
       +------------------+                                     
```

```
net/ipv4/af_inet.c                           
+------------------+                          
| tcp_v4_init_sock | : init tcp socket        
+-|----------------+                          
  |    +---------------+                      
  |--> | tcp_init_sock |                      
  |    +---------------+                      
  |                                           
  +--> install ipv4 operations 'ipv4_specific'
```

</details>
  
### setsockopt()

(TBD)

<details><summary> More Details </summary>

```
net/socket.c                            
+----------------+                       
| sys_setsockopt | : set socket options  
+------------------+                     
| __sys_setsockopt | : set socket options
+-|----------------+                     
  |                                      
  |--> if level is SOL_SOCKET            
  |    |                                 
  |    |    +-----------------+          
  |    +--> | sock_setsockopt |          
  |         +-----------------+          
  |                                      
  +--> else                              
       -                                 
       +--> call ->setsockopt(), e.g.,   
            +------------------------+   
            | sock_common_setsockopt |   
            +------------------------+   
```
  
</details>
  
### bind()

TCP and UDP protocols introduce ports in the transport layer, separate from physical network ports. 
When setting up a server, each service occupies at least one port using the `bind()` function. 
Common TCP ports include 80 (HTTP), 8080 (HTTPS), 22 (SSH), 53 (DNS), and more. 
If the caller does not specify a port, `bind()` will assign a valid port automatically.

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

<details><summary> More Details </summary>

```
net/socket.c                                                     
+----------+                                                      
| sys_bind | : ensure a valid port is ready                       
+----------+-+                                                    
| __sys_bind | : ensure a valid port is ready                     
+-|----------+                                                    
  |    +---------------------+                                    
  |--> | sockfd_lookup_light | get socket by file descriptor      
  |    +---------------------+                                    
  |    +---------------------+                                    
  |--> | move_addr_to_kernel | copy data from user to kernel space
  |    +---------------------+                                    
  |                                                               
  +--> call ->bind(), e.g.,                                       
       +-----------+                                              
       | inet_bind | ensure a valid port is ready                 
       +-----------+                                              
```

```
net/ipv4/af_inet.c                                 
+-----------+                                       
| inet_bind | : ensure a valid port is ready        
+-|---------+                                       
  |                                                 
  |--> call type->bind() if it exists (not our case)
  |                                                 
  |    +-------------+                              
  +--> | __inet_bind | ensure a valid port is ready 
       +-------------+                              
```

```
net/ipv4/af_inet.c                                      
+-------------+                                          
| __inet_bind | : ensure a valid port is ready           
+-|-----------+                                          
  |                                                      
  +--> call type->get_port(), e.g.,                      
       +-------------------+                             
       | inet_csk_get_port | ensure a valid port is ready
       +-------------------+                             
```

```
net/ipv4/inet_connection_sock.c                    
+-------------------+                               
| inet_csk_get_port | : ensure a valid port is ready
+-|-----------------+                               
  |                                                 
  |--> if no given port, find a valid one and return
  |                                                 
  +--> check if given port is valid                 
```
  
</details>

### listen()

In TCP, sockets have various states that indicate their current status. 
The `listen()` function alters the socket state and adds it to the hash table, where it waits for a client to connect. 
The system socket status can be viewed using the `netstat` utility, which likely retrieves information from the hash table.

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

<details><summary> More Details </summary>

```
net/socket.c                                                                                               
+------------+                                                                                              
| sys_listen | : set tcp state to LISTEN, ensure a ready port, add socket to hash table for connection      
+--------------+                                                                                            
| __sys_listen | : set tcp state to LISTEN, ensure a ready port, add socket to hash table for connection    
+-|------------+                                                                                            
  |    +---------------------+                                                                              
  |--> | sockfd_lookup_light |                                                                              
  |    +---------------------+                                                                              
  |                                                                                                         
  +--> call ->listen(), e.g.,                                                                               
       +-------------+                                                                                      
       | inet_listen | set tcp state to LISTEN, ensure a ready port, add socket to hash table for connection
       +-------------+                                                                                      
```

```
net/socket.c                                                                                                         
+-------------+                                                                                                       
| inet_listen | : set tcp state to LISTEN, ensure a ready port, add socket to hash table for connection               
+-|-----------+                                                                                                       
  |    +-----------------------+                                                                                      
  +--> | inet_csk_listen_start | set tcp state to LISTEN, ensure a ready port, add socket to hash table for connection
       +-----------------------+                                                                                      
```

```
net/ipv4/inet_connection_sock.c                                                                                 
+-----------------------+                                                                                        
| inet_csk_listen_start | : set tcp state to LISTEN, ensure a ready port, add socket to hash table for connection
+-|---------------------+                                                                                        
  |                                                                                                              
  |--> set tcp state to LISTEN                                                                                   
  |                                                                                                              
  |--> call ->get_port(), e.g.,                                                                                  
  |    +-------------------+                                                                                     
  |    | inet_csk_get_port | ensure we have a valid port                                                         
  |    +-------------------+                                                                                     
  |                                                                                                              
  +--> call ->hash(), e.g.,                                                                                      
       +-----------+                                                                                             
       | inet_hash | add socket to hash table waiting for connection                                             
       +-----------+                                                                                             
```

</details>
  
### connect()

Clients use this function to send packets by providing the server's address and port. 
The goal is to establish a successful external connection. Unlike servers, clients can use a random port for their communication needs. 
In TCP, if a sender does not receive a response from the receiver within a predefined interval, retransmission occurs.

On the server side, the operating system receives the packet and identifies the appropriate socket for the target service from a hash table, using the combination of address and port as the key. 

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

<details><summary> More Details </summary>

```
net/socket.c                                                                                        
+-------------+                                                                                      
| sys_connect |                                                                                      
+-------------+-+                                                                                    
| __sys_connect |                                                                                    
+-|-------------+                                                                                    
  |    +---------------------+                                                                       
  |--> | move_addr_to_kernel | copy data from user to kernel space                                   
  |    +---------------------+                                                                       
  |                                                                                                  
  +--> call ->connect(), e.g.,                                                                       
       +---------------------+                                                                       
       | inet_stream_connect | set addr and port, allocate skb and build tcp header, send to ip layer
       +---------------------+                                                                       
```

```
net/ipv4/af_inet.c                                                                               
+---------------------+                                                                           
| inet_stream_connect | : set addr and port, allocate skb and build tcp header, send to ip layer  
+-----------------------+                                                                         
| __inet_stream_connect | : set addr and port, allocate skb and build tcp header, send to ip layer
+-|---------------------+                                                                         
  |                                                                                               
  +--> call ->connect(), e.g.,                                                                    
       +----------------+                                                                         
       | tcp_v4_connect | set addr and port, allocate skb and build tcp header, send to ip layer  
       +----------------+                                                                         
```

```
net/ipv4/tcp_ipv4.c
+----------------+
| tcp_v4_connect | : set addr and port, allocate skb and build tcp header, send to ip layer
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
    +--> | tcp_connect | alloc skb and pass to ip layer for transmit, re-send if no response before timeout
         +-------------+
```

```
net/ipv4/tcp_ipv4.c                                                                                
+-------------+                                                                                     
| tcp_connect | : alloc skb and pass to ip layer for transmit, re-send if no response before timeout
+-|-----------+                                                                                     
  |    +---------------------+                                                                      
  |--> | sk_stream_alloc_skb | allocate socket buffer (skb)                                         
  |    +---------------------+                                                                      
  |    +------------------+                                                                         
  |--> | tcp_transmit_skb | pass packekt to ip layer for transmit                                   
  |    +------------------+                                                                         
  |    +---------------------------+                                                                
  +--> | inet_csk_reset_xmit_timer | re-send the packet if no response before timeout               
       +---------------------------+                                                                
```

```
net/ipv4/tcp_output.c                                        
+------------------+                                          
| tcp_transmit_skb | : pass packekt to ip layer for transmit  
+--------------------+                                        
| __tcp_transmit_skb | : pass packekt to ip layer for transmit
+-|------------------+                                        
  |                                                           
  +--> call ->queue_xmit(), e.g.,                             
       +---------------+                                      
       | ip_queue_xmit |                                      
       +---------------+                                      
```

</details>
  
### accept()

After the client invokes `connect()` to connect to the server, it waits for acceptance. 
When the server decides to accept the request, a socket and file pair are prepared on the server side for actual data transmission. 
Simultaneously, the client side updates its socket state to CONNECTED. 
At this stage, both sides have completed the well-known three-way handshake of TCP and officially established the connection.

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

<details><summary> More Details </summary>

```
net/socket.c                                                                                                         
+------------+                                                                                                        
| sys_accept | : alloc socket for connection, prepare valid fd/file, remove req from queue, install file, return fd   
+------------+--+                                                                                                     
| __sys_accept4 | : alloc socket for connection, prepare valid fd/file, remove req from queue, install file, return fd
+-|-------------+                                                                                                     
  |    +------------+                                                                                                 
  +--> | sock_alloc | allocate socket for the connection                                                              
  |    +------------+                                                                                                 
  |    +---------------------+                                                                                        
  |--> | get_unused_fd_flags | get an valid file descriptor                                                           
  |    +---------------------+                                                                                        
  |    +-----------------+                                                                                            
  |--> | sock_alloc_file | allocate file for the connection                                                           
  |    +-----------------+                                                                                            
  |                                                                                                                   
  |--> call ->accept(), e.g.,                                                                                         
  |    +-------------+                                                                                                
  |    | inet_accept | remove a request (if there's one) from queue                                                   
  |    +-------------+                                                                                                
  |                                                                                                                   
  |--> copy address info to userspace if asked to                                                                     
  |                                                                                                                   
  |    +------------+                                                                                                 
  |--> | fd_install | install the file to allocated fd                                                                
  |    +------------+                                                                                                 
  |                                                                                                                   
  +--> return fd                                                                                                      
```

```
net/ipv4/af_inet.c                                                    
+-------------+                                                        
| inet_accept | : remove a request (if there's one) from queue         
+-|-----------+                                                        
  |                                                                    
  +--> call ->accept(), e.g.,                                          
       +-----------------+                                             
       | inet_csk_accept | remove a request (if there's one) from queue
       +-----------------+                                             
```

```
net/ipv4/inet_connection_sock.c                                  
+-----------------+                                               
| inet_csk_accept | : remove a request (if there's one) from queue
+-|---------------+                                               
  |                                                               
  |--> if no connection request in queue                          
  |    |                                                          
  |    |    +---------------------------+                         
  |    +--> | inet_csk_wait_for_connect |                         
  |         +---------------------------+                         
  |    +--------------------+                                     
  +--> | reqsk_queue_remove | remove a request                    
       +--------------------+                                     
```
  
</details>
  
### write()

Each side can freely write data to its respective socket, and this data follows the standard "file write" flow before reaching the transport layer. 
TCP prepares the socket buffer (SKB), copies the data onto it, including the TCP header, and then transmits it to the Internet layer.

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

<details><summary> More Details </summary>

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

The driver activates the NAPI mechanism when the network interface hardware receives a packet. 
This mechanism subsequently places the partially stripped packet onto the receive queue in the TCP layer. 
Each time users read, the `tcp_recvmsg` function is triggered to retrieve the packet.

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

<details><summary> More Details </summary>

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

When TCP is selected as the transport layer, Internet Protocol (IP) serves as the network layer by default, regardless of our explicit configuration. 
The network layer determines the packet's direction:

- Output: The packet is sent to the network interface layer for transmission.
- Input: The packet is sent to the transport layer.
- Forward: This is typically the case for intermediate stops along the route.

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

In the TCP layer mentioned above, the IP layer is involved only in the `connect()` and `write()` operations, where the packet is sent out using `tcp_transmit_skb()`. 
In the case of the `read()` action, the IP layer activates the TCP handler to place the data on the socket's receive queue.

| TCP Layer                | IP Layer                         |
| ---                      | ---                              |
| tcp_v4_connect           | ip_route_connect & ip_queue_xmit |
| tcp_sendmsg              | ip_queue_xmit                    |
| tcp_recvmsg & tcp_v4_rcv | ip_rcv                           |

After the kernel completes the boot process and hands over control to systemd, a userspace utility is responsible for configuring the route information. 
The specific method used to obtain this data remains unknown to me at this time. 
However, the kernel possesses the necessary knowledge regarding which gateway's IP address to incorporate into the packet header during construction.

```
root@romulus:~# cat /proc/net/route 
Iface   Destination     Gateway         Flags   RefCnt  Use     Metric  Mask            MTU     Window  IRTT
eth0    00000000        0202000A        0003    0       0       1024    00000000        0       0       0
               
eth0    0002000A        00000000        0001    0       0       1024    00FFFFFF        0       0       0
               
eth0    0202000A        00000000        0005    0       0       1024    FFFFFFFF        0       0       0
               
eth0    0302000A        00000000        0005    0       0       1024    FFFFFFFF        0       0       0
             
```

We can use the `route` utility to display the route information in a user-friendly format.

```
root@romulus:~# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         10.0.2.2        0.0.0.0         UG    1024   0        0 eth0
10.0.2.0        *               255.255.255.0   U     1024   0        0 eth0
10.0.2.2        *               255.255.255.255 UH    1024   0        0 eth0
10.0.2.3        *               255.255.255.255 UH    1024   0        0 eth0                                    
```

The `ip_route_connect` function performs the crucial task of looking up the gateway IP address, which is necessary for constructing the IP header.

<details><summary> More Details </summary>

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
  
During the transmit process, `ip_queue_xmit()` serves as the entry point from the TCP layer. 
It constructs the IP header for the packet, handles fragmentation if the packet size exceeds the Maximum Transmission Unit (MTU), and then forwards the packet to the network device driver.

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

<details><summary> More Details </summary>

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

Upon arrival via the NAPI mechanism, `ip_rcv()` examines the IP header and determines the appropriate handler to invoke based on the `protocol` field. 
Through manipulation of the skb pointers, the higher layer remains unaware of the IP header, creating the illusion that it has been stripped.

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

Upon receiving the skb from the Internet layer, the network device driver constructs the MAC header, populates the TX descriptor(s), and instructs the hardware to perform the necessary action.

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

<details><summary> More Details </summary>

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

To handle incoming packets, the registered Interrupt Service Routine (ISR) schedules the NAPI structure of the driver for processing. 
Instead of relying solely on traditional interrupt mechanisms triggered by hardware events, NAPI combines interrupt and polling methods. 
This approach is particularly beneficial for modern high-speed network interface cards (NICs) that generate a high volume of network interrupts.

When a network interrupt occurs, the NIC interrupt is disabled, and the polling method takes over. 
The polling function, registered by the NIC driver, continues to receive packets if any are available, effectively reducing the frequency of interrupts and alleviating the strain on the system. 
Once no further packets arrive, the system reverts to using the interrupt mechanism.

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

The `poll` function identifies the packet type by examining the `protocol` field and determines the appropriate handler for transferring the packet to the higher layer.

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

<details><summary> More Details </summary>

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
                                             
Summary of read and write flow in the network layer model.

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

During the boot-up process, several init calls utilize the `sock_register()` function to register the network family and add support for the specified socket type to the system. 
It is worth noting that AF_OOO and PF_OOO are sometimes used interchangeably, although they represent the same concept.

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

<details><summary> More Details </summary>

```
drivers/net/ethernet/faraday/ftgmac100.c                                                                                                                      
+-----------------+                                                                                                                                            
| ftgmac100_probe | : prepare net_dev, init reset work, determine mac, handle ncsi or phy, register net_dev                                                    
+-|---------------+                                                                                                                                            
  |    +------------------+                                                                                                                                    
  |--> | platform_get_irq |                                                                                                                                    
  |    +------------------+                                                                                                                                    
  |    +----------------+                                                                                                                                      
  |--> | alloc_etherdev | alloc net_device + priv, alloc tx/rx queues, install basic ops                                                                       
  |    +----------------+                                                                                                                                      
  |              +----------------------+                                                                                                                      
  |--> init work | ftgmac100_reset_task | reset hw, reset sw (all desc/skb)                                                                                    
  |              +----------------------+                                                                                                                      
  |    +---------+                                                                                                                                             
  +--> | ioremap | map hw reg                                                                                                                                  
  |    +---------+                                                                                                                                             
  |    +-----------------------+                                                                                      net_device                               
  |--> | ftgmac100_initial_mac | determine mac and write to reg                                                     +-------------+                            
  |    +-----------------------+                                                                                    | .header_ops |---->  eth_header_ops       
  |                                                                                                                 |.ethtool_ops |--\    default_ethtool_ops- 
  |--> if property has specified 'use-ncsi'                                                                         |             |   ->  ftgmac100_ethtool_ops
  |    |                                                                                                            |             |                            
  |    |--> print "Using NCSI interface\n"                                                                          |  netdev_ops |       ftgmac100_netdev_ops 
  |--> |                                                                                                            |             |                            
  |    |    +-------------------+                                                                                   |             |                            
  |    +--> | ncsi_register_dev | (skip for now)                                                                    |             |                            
  |         +-------------------+                                                                                   +-------------+                            
  |                                                                                                                                                            
  |--> elif property has specified 'phy-handle'                                                                                                                
  |    |                                                                                                                                                       
  |    |    +------------------------+                                                                                                                         
  |    |--> | of_phy_get_and_connect | get phy-mode from dt, bind phy dev/drv, install handler to adjust link                                                  
  |    |    +------------------------+                                                                                                                         
  |    |    +-------------------+                                                                                                                              
  |    +--> | phy_attached_info | print phy info                                                                                                               
  |         +-------------------+                                                                                                                              
  |                                                                                                                                                            
  |--> elif it's aspeed old-style device tree (skip)                                                                                                           
  |                                                                                                                                                            
  |--> if aspeed, setup clk                                                                                                                                    
  |                                                                                                                                                            
  |--> set tx/rx entry#, base feature,                                                                                                                         
  |                                                                                                                                                            
  |    +-----------------+                                                                                                                                     
  +--> | register_netdev | determine name/idx, config features, add dev, label 'present', apply noop_qdisc, setup watchdog                                     
       +-----------------+                                                                                                                                     
```

```
include/linux/etherdevice.h                                                           
+----------------+                                                                     
| alloc_etherdev | : alloc net_device + priv, alloc tx/rx queues, install basic ops    
+-------------------+                                                                  
| alloc_etherdev_mq | : alloc net_device + priv, alloc tx/rx queues, install basic ops 
+--------------------+                                                                 
| alloc_etherdev_mqs | : alloc net_device + priv, alloc tx/rx queues, install basic ops
+------------------+-+                                                                 
| alloc_netdev_mqs | : alloc net_device + priv, alloc tx/rx queues, install basic ops  
+-|----------------+                                                                   
  |                                                                                    
  |--> alloc 'net_device' + 'priv' combined                                            
  |                                                                                    
  |    +---------------+                                                               
  |--> | dev_addr_init | init dev addr list and creat the first node                   
  |    +---------------+                                                               
  |    +-------------+                                                                 
  |--> | dev_net_set | set dev's network namespace                                     
  |    +-------------+                                                                 
  |                                                                                    
  |--> call ->setup, e.g.,                                                             
  |    +-------------+                                                                 
  |    | ether_setup | init with ethernet-generic values                               
  |    +-------------+                                                                 
  |                                                                                    
  |    +---------------------------+                                                   
  |--> | netif_alloc_netdev_queues | alloc and init tx queue                           
  |    +---------------------------+                                                   
  |    +-----------------------+                                                       
  |--> | netif_alloc_rx_queues | alloc and init rx queue                               
  |    +-----------------------+                                                       
  |                                                                                    
  +--> ensure dev has its ethtool_ops installed                                        
```

```
net/core/dev_addr_lists.c                                     
+---------------+                                              
| dev_addr_init | : init dev addr list and creat the first node
+-|-------------+                                              
  |    +---------------+                                       
  |--> | __hw_addr_add | add addr to dev                       
  |    +---------------+                                       
  |                                                            
  +--> have dev addr point to the one just created             
```

```
drivers/net/ethernet/faraday/ftgmac100.c                                                                      
+----------------------+                                                                                       
| ftgmac100_reset_task | : reset hw, reset sw (all desc/skb)                                                   
+-|--------------------+                                                                                       
  |                                                                                                            
  |--> if the interface isn't running, return                                                                  
  |                                                                                                            
  |    +--------------+                                                                                        
  |--> | napi_disable | disable napi                                                                           
  |    +--------------+                                                                                        
  |    +------------------+                                                                                    
  |--> | netif_tx_disable | disable tx queue                                                                   
  |    +------------------+                                                                                    
  |    +-------------------+                                                                                   
  |--> | ftgmac100_stop_hw | stop hw                                                                           
  |    +-------------------+                                                                                   
  |    +--------------------------------+                                                                      
  |--> | ftgmac100_reset_and_config_mac | reset hw reg                                                         
  |    +--------------------------------+                                                                      
  |    +------------------------+                                                                              
  |--> | ftgmac100_free_buffers | free all packets in rx/tx queues                                             
  |    +------------------------+                                                                              
  |    +--------------------+                                                                                  
  +--> | ftgmac100_init_all | init desc & alloc skb for rx/tx queues, start hw, enable napi/tx_queue/interrupts
       +--------------------+                                                                                  
```

```
drivers/net/ethernet/faraday/ftgmac100.c                                                                 
+--------------------+                                                                                    
| ftgmac100_init_all | : init desc & alloc skb for rx/tx queues, start hw, enable napi/tx_queue/interrupts
+-|------------------+                                                                                    
  |    +----------------------+                                                                           
  |--> | ftgmac100_init_rings | init all desc on rx/tx queues                                             
  |    +----------------------+                                                                           
  |    +----------------------------+                                                                     
  |--> | ftgmac100_alloc_rx_buffers | alloc skb for each desc on rx queue                                 
  |    +----------------------------+                                                                     
  |    +-------------------+                                                                              
  |--> | ftgmac100_init_hw | (rw)init hw (e.g., clear interrupts, write mac addr, ...)                    
  |    +-------------------+                                                                              
  |    +------------------------+                                                                         
  |--> | ftgmac100_config_pause |                                                                         
  |    +------------------------+                                                                         
  |    +--------------------+                                                                             
  |--> | ftgmac100_start_hw | start hw                                                                    
  |    +--------------------+                                                                             
  |    +-------------+                                                                                    
  |--> | napi_enable | enable napi                                                                        
  |    +-------------+                                                                                    
  |    +-------------------+                                                                              
  |--> | netif_start_queue | enable tx queue (allow transmit)                                             
  |    +-------------------+                                                                              
  |                                                                                                       
  +--> enable itnerrupts                                                                                  
```

```
drivers/of/of_mdio.c                                                                              
+------------------------+                                                                         
| of_phy_get_and_connect | : get phy-mode from dt, bind phy dev/drv, install handler to adjust link
+-|----------------------+                                                                         
  |                                                                                                
  |--> get property 'phy-mode' as interface                                                        
  |                                                                                                
  |--> if it's a fixed link (not the case of aspeed-ast2600-ncsi.dts)                              
  |    -                                                                                           
  |    +--> (skip)                                                                                 
  |                                                                                                
  |--> else                                                                                        
  |    |                                                                                           
  |    |    +------------------+                                                                   
  |    +--> | of_parse_phandle | get dt node of "phy-handle"                                       
  |         +------------------+                                                                   
  |    +----------------+                                                                          
  +--> | of_phy_connect | given phy dt_node, bind its dev/drv, install handler to adjust link      
       +----------------+                                                                          
```

```
drivers/of/of_mdio.c                                                                   
+----------------+                                                                      
| of_phy_connect | : given phy dt_node, bind its dev/drv, install handler to adjust link
+-|--------------+                                                                      
  |    +--------------------+                                                           
  |--> | of_phy_find_device | given phy dt_node, get its mdio_dev                       
  |    +--------------------+                                                           
  |    +--------------------+                                                           
  +--> | phy_connect_direct | bind phy dev/drv, install handler to adjust link          
       +--------------------+                                                           
```

```
drivers/net/phy/phy_device.c                                                             
+--------------------+                                                                    
| phy_connect_direct | : bind phy dev/drv, install handler to adjust link                 
+-|------------------+                                                                    
  |    +-------------------+                                                              
  |--> | phy_attach_direct | ensure dev/drv of phy is bound, adjust carrier/link, init phy
  |    +-------------------+                                                              
  |    +------------------+                                                               
  |--> | phy_prepare_link | install handler to phy_dev                                    
  |    +------------------+ +-----------------------+                                     
  |                         | ftgmac100_adjust_link | reset hw, reset sw (all desc/skb)   
  |                         +-----------------------+                                     
  |                                                                                       
  +--> if phy_dev has valid interrupt (probably not our case)                             
       |                                                                                  
       |    +-----------------------+                                                     
       +--> | phy_request_interrupt | (skip)                                              
            +-----------------------+                                                     
```

```
drivers/net/phy/phy_device.c                                                        
+-------------------+                                                                
| phy_attach_direct | : ensure dev/drv of phy is bound, adjust carrier/link, init phy
+-|-----------------+                                                                
  |                                                                                  
  |--> if the phy_dev has no driver yet                                              
  |    -                                                                             
  |    +--> use the generic phy driver                                               
  |                                                                                  
  |--> if we use the generic phy driver                                              
  |    |                                                                             
  |    |--> call ->probe(), e.g.,                                                    
  |    |    +-----+                                                                  
  |    |    | ??? |                                                                  
  |    |    +-----+                                                                  
  |    |                                                                             
  |    |    +--------------------+                                                   
  |    +--> | device_bind_driver |                                                   
  |         +--------------------+                                                   
  |                                                                                  
  |--> install phy_link_change handler                                               
  |    +-----------------+                                                           
  |    | phy_link_change | enable/disable carrier, adjust link (reset hw/sw)         
  |    +-----------------+                                                           
  |                                                                                  
  |    +------------------------+                                                    
  |--> | phy_sysfs_create_links |                                                    
  |    +------------------------+                                                    
  |                                                                                  
  |--> if arg netdev is given                                                        
  |    |                                                                             
  |    |    +-------------------+                                                    
  |    +--> | netif_carrier_off | label 'off' on netdev                              
  |         +-------------------+                                                    
  |    +-------------+                                                               
  |--> | phy_init_hw | reset phy dev, init config                                    
  |    +-------------+                                                               
  |    +------------+                                                                
  |--> | phy_resume |                                                                
  |    +------------+                                                                
  |    +---------------------------+                                                 
  +--> | phy_led_triggers_register | do nothing bc of disabled config                
       +---------------------------+                                                 
```

```
drivers/net/phy/phy_device.c                                                                  
+-----------------+                                                                            
| phy_link_change | : enable/disable carrier, adjust link (reset hw/sw)                        
+-|---------------+                                                                            
  |                                                                                            
  |--> if arg 'do_carrier' is set                                                              
  |    |                                                                                       
  |    |--> if up                                                                              
  |    |    |                                                                                  
  |    |    |    +------------------+                                                          
  |    |    +--> | netif_carrier_on | clear 'no carrier' flag, enable watchdog timer for netdev
  |    |         +------------------+                                                          
  |    |                                                                                       
  |    +--> else                                                                               
  |         |                                                                                  
  |         |    +-------------------+                                                         
  |         +--> | netif_carrier_off | label 'no carrier' on netdev                            
  |              +-------------------+                                                         
  |                                                                                            
  +--> call ->adjust_link(), e.g.,                                                             
       +-----------------------+                                                               
       | ftgmac100_adjust_link | save new attributes in priv, reset hw/sw                      
       +-----------------------+                                                               
```

```
                                                                                
 net/sched/sch_generic.c                                                        
+------------------+                                                            
| netif_carrier_on | : clear 'no carrier' flag, enable watchdog timer for netdev
+-|----------------+                                                            
  |                                                                             
  |--> clear 'no carrier' flag from netdev                                      
  |                                                                             
  |    +----------------------+                                                 
  |--> | linkwatch_fire_event | (skip)                                          
  |    +----------------------+                                                 
  |                                                                             
  +--> if netdev is running                                                     
       |                                                                        
       |    +----------------------+                                            
       +--> | __netdev_watchdog_up | enable watchdog timer for netdev           
            +----------------------+                                            
```

```
net/sched/sch_generic.c                                   
+----------------------+                                   
| __netdev_watchdog_up | : enable watchdog timer for netdev
+-|--------------------+                                   
  |                                                        
  +--> if ->ndo_tx_timeout() exists                        
       |                                                   
       |    +-----------+                                  
       +--> | mod_timer | watchdog timer                   
            +-----------+                                  
```

```
net/sched/sch_generic.c
+-------------------+                               
| netif_carrier_off | : label 'no carrier' on netdev
+-|-----------------+                               
  |                                                 
  |--> label 'no carrier' on netdev                 
  |                                                 
  |    +----------------------+                     
  +--> | linkwatch_fire_event | (skip)              
       +----------------------+                     
```

```
drivers/net/phy/phy_device.c                                                      
+-----------------------+                                                          
| ftgmac100_adjust_link | : save new attributes in priv, reset hw/sw               
+-|---------------------+                                                          
  |                                                                                
  |--> if everything (speed/duplex/pause) is the same, return                      
  |                                                                                
  |--> if link status changes                                                      
  |    |                                                                           
  |    |    +------------------+                                                   
  |    +--> | phy_print_status | print link status (up or down)                    
  |         +------------------+                                                   
  |                                                                                
  |--> save new attributes (speed/duplex/pause) in private                         
  |                                                                                
  |--> if link is down, return                                                     
  |                                                                                
  |--> write hw reg to disable interrupts                                          
  |                                                                                
  |    +---------------+                                                           
  +--> | schedule_work | place work to global queue                                
       +---------------+ +----------------------+                                  
                         | ftgmac100_reset_task | reset hw, reset sw (all desc/skb)
                         +----------------------+                                  
```

```
drivers/net/phy/phy.c                                    
+------------------+                                      
| phy_print_status | : print link status (up or down)     
+-|----------------+                                      
  |                                                       
  |--> if phy_dev has link                                
  |    -                                                  
  |    +--> print "Link is Up - %s/%s - flow control %s\n"
  |                                                       
  +--> else                                               
       -                                                  
       +--> print "Link is Down\n"                        
```

```
drivers/net/phy/phy_device.c               
+-------------+                             
| phy_init_hw | : reset phy dev, init config
+-|-----------+                             
  |    +------------------+                 
  |--> | phy_device_reset |                 
  |    +------------------+                 
  |                                         
  |--> if ->soft_reset() exists             
  |    -                                    
  |    +--> call it                         
  |                                         
  |    +-----------------+                  
  |--> | phy_scan_fixups | (skip)           
  |    +-----------------+                  
  |                                         
  +--> if ->config_init() exists            
       -                                    
       +--> call it                         
```

```
drivers/net/phy/phy_device.c                                        
+-------------------+                                                
| phy_attached_info | : print phy info                               
+--------------------+                                               
| phy_attached_print | : print phy info                              
+-|------------------+                                               
  |                                                                  
  +--> print "attached PHY driver [%s] (mii_bus:phy_addr=%s, irq=%s)"
```

```
net/core/dev.c
+-----------------+
| register_netdev | : determine name/idx, config features, add dev, label 'present', apply noop_qdisc, setup watchdog
+--------------------+
| register_netdevice | : determine name/idx, config features, add dev, label 'present', apply noop_qdisc, setup watchdog
+-|------------------+
  |    +--------------------+
  |--> | dev_get_valid_name | determine name
  |    +--------------------+
  |
  |--> if ->ndo_init() exists, call it (not our case)
  |
  |--> determine index
  |
  |--> config features
  |
  |    +--------------------------+
  |--> | call_netdevice_notifiers | 'post init'
  |    +--------------------------+
  |    +-------------------------+
  |--> | netdev_register_kobject | set dev name and add dev to framework
  |    +-------------------------+
  |    +--------------------------+
  |--> | __netdev_update_features | (skip)
  |    +--------------------------+
  |
  |--> label 'present' on net_dev
  |
  |    +--------------------+
  |--> | linkwatch_init_dev | (skip)
  |    +--------------------+
  |    +--------------------+
  |--> | dev_init_scheduler | apply noop_qdisc, setup timer of watchdog monitoring device
  |    +--------------------+
  |    +----------------+
  |--> | list_netdevice | add to net namespace and hashtables
  |    +----------------+
  |    +--------------------------+
  +--> | call_netdevice_notifiers | 'netdev register'
       +--------------------------+
```

```
net/sched/sch_generic.c                                                              
+--------------------+                                                                
| dev_init_scheduler |  : apply noop_qdisc, setup timer of watchdog monitoring device 
+-|------------------+                                                                
  |                                                                                   
  |--> apply 'noop_qdisc' to net_dev and its tx queues                                
  |                                                                                   
  |    +-------------+                                                                
  +--> | timer_setup | setup a timer with function                                    
       +-------------+ +--------------+                                               
                       | dev_watchdog | show warning if it expires, update next expiry
                       +--------------+                                               
```

```
net/sched/sch_generic.c                                                                            
+--------------+                                                                                    
| dev_watchdog | : show warning if it expires, update next expiry                                   
+-|------------+                                                                                    
  |                                                                                                 
  +--> get related net_dev                                                                          
  |                                                                                                 
  +--> if its qdisc != noop                                                                         
       -                                                                                            
       +--> if the net_dev is present && running && carrier ok                                      
            |                                                                                       
            |--> for each tx queue                                                                  
            |    -                                                                                  
            |    +--> if transmit is off and it expires                                             
            |         |                                                                             
            |         |--> update timeout statistics                                                
            |         |                                                                             
            |         +--> break (to warn immediatly)                                               
            |                                                                                       
            |--> if timeout was detected                                                            
            |    |                                                                                  
            |    |--> warn "NETDEV WATCHDOG: %s (%s): transmit queue %u timed out\n"                
            |    |                                                                                  
            |    +--> call ->ndo_tx_timeout(), e.g.,                                                
            |         +----------------------+                                                      
            |         | ftgmac100_tx_timeout | disable interrupts, reset hw, reset sw (all desc/skb)
            |         +----------------------+                                                      
            |                                                                                       
            +--> update next expiry of watchdog                                                     
```

</details>

## <a name="reference"></a> Reference

- [TCP Server-Client implementation in C](https://www.geeksforgeeks.org/tcp-server-client-implementation-in-c/)
- [IPv4 route lookup on Linux](https://vincent.bernat.ch/en/blog/2017-ipv4-route-lookup-linux)
