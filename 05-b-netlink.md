> Study case: Linux version 5.15.0 on AST2500 emulation

## Index

- [Introduction](#introduction)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

The original design of socket transmission applies to the route between localhost and remote machine. 
Kernel later adopts this mechanism (Netlink) and benefits kernel space since it lacks initializing the communication with userspace. 
It's one of the network families and focuses on bi-direction communication between any ends on the same system. 
The kernel initializes the Netlink table (nl_table) during boot time, and each element contains a hash table for socket lookup.

```
         +--                                 hash table                      
         |                                   +-+------+                      
         |                                   +-+------+      multicast list  
         |      nl_table[NETLINK_ROUTE]      +-+------+        +-+-+-+-+-+   
         |                                   +-+------+                      
         |                                   +-+------+                      
         |    ---------------------------------------------------------------
         |                                   hash table                      
         |                                   +-+------+                      
         |                                   +-+------+      multicast list  
         |      nl_table[NETLINK_UNUSED]     +-+------+        +-+-+-+-+-+   
         |                                   +-+------+                      
         |                                   +-+------+                      
 n = 32  |    ---------------------------------------------------------------
         |                                   hash table                      
         |                                   +-+------+                      
         |                                   +-+------+      multicast list  
         |      nl_table[NETLINK_USERSOCK]   +-+------+        +-+-+-+-+-+   
         |                                   +-+------+                      
         |                                   +-+------+                      
         |    ---------------------------------------------------------------
         |                                                                   
         |                                   -                               
         |                                   -                               
         |                                   -                               
         +--                                 -                               
```

Among the 32 entries, different subsystems create Netlink sockets and insert them into six hash tables in my study case.

| Name                   | Table Index | Receive Function |
| ---                    | ---         | ---              |
| NETLINK_ROUTE          | 0           | rtnetlink_rcv    |
| NETLINK_SOCK_DIAG      | 4           | sock_diag_rcv    |
| NETLINK_FIB_LOOKUP     | 10          | nl_fib_input     |
| NETLINK_NETFILTER      | 12          | nfnetlink_rcv    |
| NETLINK_KOBJECT_UEVENT | 15          | uevent_net_rcv   |
| NETLINK_GENERIC        | 16          | genl_rcv         |

<details>
  <summary> Code Trace </summary>

```
+-----------------+                                                                                         
| uevent_net_init |                                                                                         
+----|------------+                                                                                         
     |                                                                                                      
     |--> prepare netlink table config (e.g., callback = uevent_net_rcv)                                    
     |                                                                                                      
     |    +-----------------------+                                                                         
     +--> | netlink_kernel_create |                                                                         
          +-----|-----------------+                                                                         
                |    +-------------------------+                                                            
                +--> | __netlink_kernel_create |                                                            
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
    
</details>

In the **AF_INET** family, we can allocate a TCP/IP socket by specifying the **type** as **SOCK_STREAM** and **protocol** as **IPPROTO_IP**. 
In the **AF_NETLINK** family, the **type** seems unimportant while **protocol** behaves as the index in the Netlink table.

| Family     | Type        | Protocol               | Note           |
| ---        | ---         | ---                    | ---            |
| AF_INET    | SOCK_STREAM | IPPROTO_IP             | TCP/IP socket  |
| AF_NETLINK | SOCK_RAW    | NETLINK_KOBJECT_UEVENT | udev mechanism |

The flow also differs a bit compared to TCP/IP model, with no client and no server concept, and it's just an endpoint sending and receiving messages. 
In function **bind()**, the caller can control whether it'd like to receive the broadcast message (nl_groups = mask) or not (nl_groups = 0).

```
               AF_INET                        AF_NETLINK
                                       │
                                       │
    client                 server      │        anyone
 -----------------------------------   │   --------------
   socket()               socket()     │       socket()         prepare handle for network operations
       |                      |        │           |
       |                      v        │           v
       |                   bind()      │        bind()          add socket to netlink table[protocol]
       |                      |        │           |
       |                      v        │           |
       |                  listen()     │           |
       v                      |        │           |
   connect()                  |        │           |
       |                      v        │           |
       |                  accept()     │           |
       |                      |        │           |
       v                      v        │           v
read()/write()         read()/write()  │  sendmsg()/recvmsg()   regular data operation
       |                      |        │           |
       v                      v        │           v
    close()                close()     │        close()         close the connection
                                       │
                                       │
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
            |                            +----------------+
            |                      e.g., | netlink_create |
            |                            +---|------------+
            |                                |    +------------------+
            |                                |--> | __netlink_create | install 'netlink_ops' to socket, and init sock
            |                                |    +------------------+
            |                                |
            |                                +--> get bind() and unbine() from netlink table and install to netlink sock
            |
            |    +-------------+
            +--> | sock_map_fd | allocate a file handle for the socket, and install it to fd table
                 +-------------+
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
                      +--------------+
                e.g., | netlink_bind |
                      +---|----------+
                          |
                          |--> if the user specifies 'nl_groups' (means it'd like to receive broadcast)
                          |
                          |        +------------------------+
                          |------> | netlink_realloc_groups | prepare groups for the sock
                          |        +------------------------+
                          |    +----------------+
                          |--> | netlink_insert | insert the socke into netlink table[protocol]
                          |    +----------------+
                          |
                          |--> if the user specifies 'nl_groups' (means it'd like to receive broadcast)
                          |
                          |        +------------------------------+
                          |------> | netlink_update_subscriptions |   add the socket to the multicast list
                          |        +------------------------------+
                          |        +--------------------------+
                          +------> | netlink_update_listeners | update listener mask of table[protocol]
                                   +--------------------------+
```
    
</details>

The **udev** mechanism utilizes the Netlink to broadcast the event of device addition or removal in kernel side to sockets on the multicast list. 
The task that binds a socket to the list is surprisingly **systemd** instead of **systemd-udevd**. 
To figure that out, we need to trace its source code, and I'm not going to do that now.

```
root@romulus:~# cat /proc/net/netlink
sk               Eth Pid        Groups   Rmem     Wmem     Dump  Locks    Drops    Inode
458e1447 0   280        00000111 0        0        0     2        0        6810
8b76dec9 0   1          800405d5 0        0        0     2        0        5098
19934dfb 0   272        00000554 0        0        0     2        0        6565
4ea1821f 0   170        00000111 0        0        0     2        0        5970
6d6cc46d 0   181        00020ff8 0        0        0     2        0        5861
6e260864 0   0          00000000 0        0        0     2        0        6
80e246cb 4   0          00000000 0        0        0     2        0        4325
62642523 10  0          00000000 0        0        0     2        0        4186
5f474388 12  0          00000000 0        0        0     2        0        4326
c6d85e2d 15  138   ???  00000002 0        0        0     2        0        5430
d5950c6f 15  0      |   00000000 0        0        0     2        0        9
bdf4fdf7 15  1      v   00000002 0        0        0     2        0        5074
2bebd6b3 15  2540912388 00000001 0        0        0     2        0        5101
b68c8d8c 16  138        00000000 0        0        0     2        0        5429
9828ad2d 16  0          00000000 0        0        0     2        0        7
         +-  --+        -------+
         |     |               |
         |     |          group bitmap
         |     |
         |     |
         |     |
         |     +--------------- pid: dynamically allocated
         |                      0  : kernel
         |                      1  : systemd
netlink protocol                138: systemd-networkd
0 : NETLINK_ROUTE               170: systemd-resolved
4 : NETLINK_SOCK_DIAG           181: nscd
10: NETLINK_FIB_LOOKUP          272: phosphor-network-manager
12: NETLINK_NETFILTER           280: avahi-daemon
15: NETLINK_KOBJECT_UEVENT
16: NETLINK_GENERIC                                                                 
```

Both kernel and userspace tasks have put their sockets in the hash table of Netlink table[NETLINK_KOBJECT_UEVENT]. 
Each side can initiate the message delivery, and the device framework calls **kobject_uevent** to prepare skb and broadcast the event from the kernel side. 
Any message from userspace toward this protocol will trigger the registered receive function **uevent_net_rcv**, and eventually, it also broadcasts.

```
 nl_table[NETLINK_KOBJECT_UEVENT]



  hash table
  +-+------+
  +-+------+      multicast list
  +-+------+        +-+-+-+-+-+
  +-+------+
  +-+------+             ▲
                         │
       ▲          to be broadcast
       │
to be looked up
```

<details>
  <summary> Code Trace </summary>

```
+------------+                                                                                                                  
| device_add |                                                                                                                  
+--|---------+                                                                                                                  
   |    +----------------+                                                                                                      
   +--> | kobject_uevent |                                                                                                      
        +---|------------+                                                                                                      
            |    +--------------------+                                                                                         
            +--> | kobject_uevent_env |                                                                                         
                 +----|---------------+                                                                                         
                      |                                                                                                         
                      |--> prepare buffer for environment variables                                                             
                      |                                                                                                         
                      |--> add 'ACTION', 'DEVPATH', 'SUBSYSTEM', and 'SEQNUM' into the buffer                                   
                      |                                                                                                         
                      |    +------------------------------+                                                                     
                      +--> | kobject_uevent_net_broadcast |                                                                     
                           +-------|----------------------+                                                                     
                                   |                                                                                            
                                   |--> if net namespace isn't determined                                                       
                                   |                                                                                            
                                   |        +-------------------------------+                                                   
                                   |------> | uevent_net_broadcast_untagged | broadcast to each sock on list                    
                                   |        +-------------------------------+                                                   
                                   |                                                                                            
                                   |--> else                                                                                    
                                   |                                                                                            
                                   |        +-----------------------------+                                                     
                                   +------> | uevent_net_broadcast_tagged | broadcast to each sock on list (net namespace aware)
                                            +-----------------------------+                                                     
```
         
```
+----------------+                                                                                
| uevent_net_rcv |                                                                                
+---|------------+                                                                                
    |    +-----------------+                                                                      
    +--> | netlink_rcv_skb |                                                                      
         +----|------------+                                                                      
              |                                                                                   
              +--> call ->cb() to process the msg                                                 
                         +--------------------+                                                   
                   e.g., | uevent_net_rcv_skb |                                                   
                         +----|---------------+                                                   
                              |    +----------------------+                                       
                              +--> | uevent_net_broadcast |                                       
                                   +-----|----------------+                                       
                                         |                                                        
                                         |--> prepare another larger skb = arg skb + 'SEQNUM' info
                                         |                                                        
                                         |    +-------------------+                               
                                         +--> | netlink_broadcast |                               
                                              +-------------------+                               
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

</details>

From the user side, the regular function **sendmsg()** and **recvmsg()** serve the purpose of communication. 
The difference is that the task can aim for the target socket by specifying the PID and deciding whether to broadcast.

<details>
  <summary> Code Trace </summary>

```
+-------------+           +-----------------+
| sys_sendmsg |  ------>  | netlink_sendmsg |
+-------------+           +-----------------+
                                             
+--------------+          +-----------------+
| sys_recvmmsg | ------>  | netlink_recvmsg |
+--------------+          +-----------------+
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
+-----------------+
| netlink_recvmsg |
+----|------------+
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

</details>

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

[mwarning, netlink-examples](https://github.com/mwarning/netlink-examples)
