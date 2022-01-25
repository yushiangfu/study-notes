> Study case: Linux version 5.15.0 on AST2500 emulation

## Index

- [Introduction](#introduction)
- [Network Layers & Families](#network-layers-and-families)
- [Application Layer](#application-layer)
- [Transport Layer](#transport-layer)
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

Both the client and server sides call **socket()** to get the handle. 
The server side alone calls **bind()** to relate the socket handle to its network address and then calls **listen()** to standby. 
Anytime the client can **connect()** to the server, wait for its **accept()**, and advance to the real deal. 
Data operation can be as simple as combining a few **read()** and **write()**. 
Since kernel conceals the most complicated part, users can operate socket data like handling a file. 
Either side can **close()** this connection, and the other side will process the termination appropriately.

```                                 
   client                 server    
----------------------------------- 
  socket()               socket()   
      |                      |      
      |                      v      
      |                   bind()    
      |                      |      
      |                      v      
      |                  listen()   
      v                      |      
  connect()                  |      
      |                      v      
      |                  accept()   
      |                      |      
      v                      v      
+-----------+          +-----------+
|   data    |          |   data    |
| operation |          | operation |
+-----------+          +-----------+
      |                      |      
      v                      v      
   close()                close()   
```

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
            |                      e.g., | inet_create | install family operations to socket and call type->init()
            |                            +---|---------+
            |                                |
            |                                |--> set socket state to UNCONNECTED
            |                                |
            |                                |--> install family operations, e.g., inet_stream_ops
            |                                |
            |                                |--> allocate tcp socket (not the same socket created earlier)
            |                                |
            |                                +--> call type->init()
            |                                           +------------------+
            |                                     e.g., | tcp_v4_init_sock | init tcp socket
            |                                           +------------------+
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
                                       e.g., | inet_csk_get_port | check if the given port is valid or prepare a valid one
                                             +-------------------+                                     
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
                 e.g., | inet_listen | set socket state to 'LISTEN' and add it to hash table wating for connection
                       +-------------+                                                                            
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
                            e.g., | inet_stream_connect | build socket buffer (skb), send out, wait for acception, change state to CONNECTED
                                  +---------------------+    
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
                           +-----|-----------------+                                                                  
                                 |    +-----------+                                                                   
                                 |--> | do_accept |                                                                   
                                 |    +--|--------+                                                                   
                                 |       |    +----------------+                                                      
                                 |       |--> | sock_from_file | get socket from file                                 
                                 |       |    +----------------+                                                      
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
                                 |       |          +-------------+                                                   
                                 |       |                                                                            
                                 |       |--> copy address info to userspace if asked to                              
                                 |       |                                                                            
                                 |       +--> return allocated file                                                   
                                 |                                                                                    
                                 +--> install the file to allocated fd                                                
```

### sys_write()

(TBD)

### sys_read()

(TBD)

### sys_close()

(TBD)

The argument **family** in sys_socket() determines the first operation set for each network-related syscall.

| Syscall     | Family              |
| ---         | ---                 |
| sys_socket  | inet_create         |
| sys_bind    | inet_bind           |
| sys_listen  | inet_listen         |
| sys_connect | inet_stream_connect |
| sys_accept  | inet_accept         |
| sys_write   | (TBD)               |
| sys_read    | (TBD)               |
| sys_close   | (TBD)               |

## <a name="transport-layer"></a> Transport Layer

The transport layer provides services such as reliable transmission, flow control, connection concept.
Transmission Control Protocol (TCP) consists the above features, while User Datagram Protocol (UDP) provides a much simplified method for other transmission.

| Family              | Transport layer              |
| ---                 | ---                          |
| inet_create         | tcp_v4_init_sock             |
| inet_bind           | inet_csk_get_port            |
| inet_listen         | inet_csk_get_port, inet_hash |
| inet_stream_connect | tcp_v4_connect               |
| inet_accept         | inet_csk_accept              | 
| (TBD)               | (TBD)                        |
| (TBD)               | (TBD)                        |
| (TBD)               | (TBD)                        |

For the socket creation, not much to introduce.

```
 +-------------+                                                              
 | inet_create | install family operations to socket and call protocol->init()
 +---|---------+                                                              
     |                                                                        
     +--> call type->init()                                                   
                +------------------+                                          
          e.g., | tcp_v4_init_sock | init tcp socket                          
                +------------------+                                          
                     |    +---------------+                                   
                     |--> | tcp_init_sock |                                   
                     |    +---------------+                                   
                     |                                                        
                     +--> install ipv4 operations                                                                                              
```

```
 +-------------+                                                                             
 | __inet_bind |                                                                             
 +---|---------+                                                                             
     |                                                                                       
     +--> call type->get_port()                                                              
                +-------------------+                                                        
          e.g., | inet_csk_get_port | check if the given port is valid or prepare a valid one
                +-------------------+                                                        
                     |                                                                       
                     |--> if no given port, find a valid one and return                      
                     |                                                                       
                     +--> check if given port is valid                                       
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

(TBD)

## <a name="reference"></a> Reference

(TBD)
