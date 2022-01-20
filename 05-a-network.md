> Study case: Linux version 5.15.0 on AST2500 emulation

## Index

- [Introduction](#introduction)
- [Network Layers](#network-layer)
- [Syscalls](#syscalls)
- [To-Do List](#to-do-list)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

(TBD)

## <a name="network-layers"></a> Network Layers

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

## <a name="syscalls"></a> Syscalls

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

### socket

```
prototype:
int socket(int domain, int type, int protocol);
```

Function **socket** takes three arguments: domain, type, and protocol.
- **domain**
  - It suffices to know that well-known protocols such as TCP, UDP, IP, ICMP belong to the family AF_INET, a.k.a. **IPv4 Internet Protocol**.
- type
  - Usually, the type is either TCP (SOCK_STREAM) or UDP (SOCK_DGRAM) in the above family.
- protocol
  - With the family **IPv4 Internet Protocol** and type TCP or UDP selected, this field can only be IP whether the caller specifies it or not.

## <a name="to-do-list"></a> To-Do List

(TBD)

## <a name="reference"></a> Reference

(TBD)
