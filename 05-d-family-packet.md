> Study case: Linux version 5.15.0 on AST2500 emulation

## Index

- [Introduction](#introduction)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

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
            |                                +------> | __register_prot_hook |
            |                                         +----------------------+
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
                                              |--> | dev_hard_header | build mac header
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

## <a name="reference"></a> Reference

(TBD)
