> Study case: Linux version 5.15.0 on AST2500 emulation

## Index

- [Introduction](#introduction)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

Utility **udhcpc** is the typical DHCP client on an embedded system that helps us get the IP dynamically from the DHCP server. 
Because it's at the stage before obtaining an IP, the interaction between the DHCP client and server differs from what we commonly know.

1. The client broadcasts a packet to discover the server.
2. The server replies with an available IP address and other information such as gateway IP that the client can configure.
3. The client broadcasts again to indicate that it will take the offer.
4. Server: cool.

```
            client                                                       server           
                                                                                          
                                                                                          
+---------------------------+      broadcast                                              
|       dhcp discover       | ------------------->                                        
+---------------------------+                                                             
 src mac = aa:aa:aa:bb:bb:bb                                                              
 dst mac = ff:ff:ff:ff:ff:ff                                                              
  src ip = 0.0.0.0                                                                        
  dst ip = 255.255.255.255                                                                
                                               unicast       +---------------------------+
                                        <------------------- |        dhcp offer         |
                                                             +---------------------------+
                                                              src mac = cc:cc:cc:dd:dd:dd 
                                                              dst mac = aa:aa:aa:bb:bb:bb 
                                                               src ip = 192.168.0.1       
                                                               dst ip = 255.255.255.255   
+---------------------------+      broadcast                                              
|       dhcp request        | ------------------->                                        
+---------------------------+                                                             
 src mac = aa:aa:aa:bb:bb:bb                                                              
 dst mac = ff:ff:ff:ff:ff:ff                                                              
  src ip = 0.0.0.0                                                                        
  dst ip = 255.255.255.255                                                                
                                               unicast       +---------------------------+
                                        <------------------- |         dhcp ack          |
                                                             +---------------------------+
                                                              src mac = cc:cc:cc:dd:dd:dd 
                                                              dst mac = aa:aa:aa:bb:bb:bb 
                                                               src ip = 192.168.0.1       
                                                               dst ip = 255.255.255.255   
```

Tracing the code of **udhcpc**, it's worth noting that the packet from the client is a self-made UDP datagram using a highly customizable socket.

```
socket(PF_PACKET, SOCK_DGRAM, htons(ETH_P_IP))
```

Doing so allows **udhcpc** to freely compose IP header, UDP header, and payload (DHCP message). 
Though it's still the kernel that builds the MAC header, the arguments come from **udhcpc**, which means it controls all the major fields of a packet.

```
+---------+
| mac hdr | src mac (dev addr) + dst mac (broadcast) + higher protocol (ip)
+---------+
|  ip hdr | src ip (any)       + dst ip (broadcast)  + higher protocol (udp)
+---------+
| udp hdr | src port (68)      + dst port (67)
+---------+
|         |
|  data   | dhcp message
|         |
+---------+
```

When creating a socket of packet family, the framework automatically registers the **packet_rcv** for specified **protocol**, e.g., IP. 
Whenever an incoming packet indicates that its higher protocol is IP, the network subsystem calls not only **ip_rcv** but also **packet_rcv**. 
The latter further wakes up the waiting task, such as **udhcpc**, to advance the DHCP procedure.

```
                           write                          read


                    1. prep payload              1. parse internet hdr
                    2. prep transport hdr        2. parse transport hdr
                    3. prep internet hdr         3. parse payload

                       +------------+                 +----------+
application            | sys_sendto |                 | sys_read |
   layer               +------------+                 +----------+
                              |                             |
              ----------------|-----------------------------v----------------
                              v                    +----------------+
                     +----------------+            | packet_recvmsg | get data from queue
                     | packet_sendmsg |            +----------------+
                     +----------------+              | packet_rcv |   put data onto queue
                              |                      +------------+
              ----------------|-----------------------------^----------------
                              v                             |
  network       +---------------------------+    +---------------------+
 interface      | ftgmac100_hard_start_xmit |    | ftgmac100_interrupt |
   layer        +---------------------------+    +---------------------+
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
            |                            +---------------+
            |                      e.g., | packet_create |
            |                            +---|-----------+
            |                                |
            |                                |--> alloc sock and install 'packet_ops'
            |                                |
            |                                |--> if 'protocol' is specified
            |                                |
            |                                |        +----------------------+
            |                                +------> | __register_prot_hook | register packet_rcv() for arg 'protocol'
            |                                         +----------------------+
            |    +-------------+
            +--> | sock_map_fd | allocate a file handle for the socket, and install it to fd table
                 +-------------+
```
  
```
+------------+                                                          
| packet_rcv |                                                          
+--|---------+                                                          
   |    +-----------+                                                   
   |--> | skb_clone | copy the skb                                      
   |    +-----------+                                                   
   |    +------------------+                                            
   |--> | __skb_queue_tail |                                            
   |    +------------------+                                            
   |                                                                    
   +--> call ->sk_data_ready()                                          
              +-------------------+                                     
        e.g., | sock_def_readable | wake up the task waiting on the data
              +-------------------+                                     
```

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
                      +-------------+
                e.g., | packet_bind |
                      +---|---------+
                          |    +----------------+
                          +--> | packet_do_bind |
                               +---|------------+
                                   |
                                   |--> get net device by either name or interface index
                                   |
                                   +--> ensure the hook is up to date
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
                                          +----------------+
                                    e.g., | packet_sendmsg |
                                          +---|------------+
                                              |
                                              |--> get previously saved protocol and net device
                                              |
                                              |    +------------------+
                                              |--> | packet_alloc_skb |
                                              |    +------------------+
                                              |    +-----------------+
                                              |--> | dev_hard_header | build mac header based on user parameters
                                              |    +-----------------+
                                              |    +-----------------------------+
                                              |--> | skb_copy_datagram_from_iter | copy data to skb
                                              |    +-----------------------------+
                                              |
                                              +--> call ->xmit()
                                                         +----------------+
                                                   e.g., | dev_queue_xmit | transfer skb to driver and send out
                                                         +----------------+
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
             |                             +----------------+
             |                       e.g., | packet_recvmsg |
             |                             +---|------------+
             |                                 |    +--------------------+
             |                                 |--> | skb_recv_datagram  | receive skb from queue, might block
             |                                 |    +--------------------+
             |                                 |    +-----------------------+
             |                                 |--> | skb_copy_datagram_msg | copy data to msg
             |                                 |    +-----------------------+
             |                                 |
             |                                 |--> copy addr info to msg
             |                                 |
             |                                 |    +-------------------+
             |                                 +--> | skb_free_datagram | free the skb
             |                                      +-------------------+
             |
             |--> if user space 'addr' is provided
             |
             |        +-------------------+
             +------> | move_addr_to_user |
                      +-------------------+
```
   
</details>

## <a name="reference"></a> Reference

- [Dynamic Host Configuration Protocol](https://en.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol)
