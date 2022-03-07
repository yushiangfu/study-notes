> Study case: Linux version 5.15.0 on AST2500 emulation

## Index

- [Introduction](#introduction)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

By observing the dumped **vmlinux**, we can see that
1. Before doing anything, it pushes some registers into the stack and recovers at the end of the function.
2. The saved register set commonly includes **fp**, **ip**, **lr**, and **pc**.

| Alias | Register | Role                 |
| ---   | ---      | ---                  |
| fp    | r11      | frame pointer        |
| ip    | r12      | scratch register (?) |
| sp    | r13      | stack pointer        |
| lr    | r14      | link register        |
| pc    | r15      | program counter      |

```
8085d9d0 <ip_rcv>:
ip_rcv():
/home/bobfu/workspace/oblinux/net/ipv4/ip_input.c:533
{
8085d9d0:   e1a0c00d    mov ip, sp
8085d9d4:   e92dd870    push    {r4, r5, r6, fp, ip, lr, pc} <========== (regular regs +) fp, ip, lr, pc
8085d9d8:   e24cb004    sub fp, ip, #4
8085d9dc:   e24dd024    sub sp, sp, #36 ; 0x24
8085d9e0:   e52de004    push    {lr}        ; (str lr, [sp, #-4]!)
8085d9e4:   ebe2cbf6    bl  801109c4 <__gnu_mcount_nc>
~~~
8085da88:   e89da870    ldm sp, {r4, r5, r6, fp, sp, pc}



static void __net_exit ipv4_frags_pre_exit_net(struct net *net)
{
8085dbe8:   e1a0c00d    mov ip, sp
8085dbec:   e92dd800    push    {fp, ip, lr, pc} <========== (regular regs +) fp, ip, lr, pc
8085dbf0:   e24cb004    sub fp, ip, #4
8085dbf4:   e52de004    push    {lr}        ; (str lr, [sp, #-4]!)
8085dbf8:   ebe2cb71    bl  801109c4 <__gnu_mcount_nc>
~~~
8085dc10:   e89da800    ldm sp, {fp, sp, pc}



80d00b78 <start_kernel>:
start_kernel():
/home/bobfu/workspace/oblinux/init/main.c:932
{
80d00b78:   e1a0c00d    mov ip, sp
80d00b7c:   e92ddff0    push    {r4, r5, r6, r7, r8, r9, sl, fp, ip, lr, pc} <========== (regular regs +) fp, ip, lr, pc
80d00b80:   e24cb004    sub fp, ip, #4
80d00b84:   e24dd024    sub sp, sp, #36 ; 0x24
80d00b88:   e52de004    push    {lr}        ; (str lr, [sp, #-4]!)
80d00b8c:   ebd03f8c    bl  801109c4 <__gnu_mcount_nc>
~~~
80d01188:   e89daff0    ldm sp, {r4, r5, r6, r7, r8, r9, sl, fp, sp, pc}
```

```
  high addr  +--+-----------+
             |  |  reserved | 8 bytes
stack ---+   |  +-----------+
         |   |  |  pt_regs  |
         |   |  +-----------+
         |   |           |
         |   |           |
         |   |           |
         |   +-----------+
         |   |           |
         |   |           |
         |   |           |
         |   |  +-----------+
         v   |  |0x57AC6E9D | stack end magic
             |  +-----------+
             |  |thread_info|
   low addr  +--+-----------+
```

```
               stack                                                                                     
               (mem)          +----                  +-------+  --+                                      
                              |        reg fp  --->  |  pc   |    |                                      
     high addr                |                      +-------+    |                                      
             |   -   |        |                      |  lr   |    |                                      
             |   -   |        |                      +-------+    | fixed cast: info of previous frame   
             |-------|        |                      |  sp   |    |                                      
 even older  | frame |        |                      +-------+    |                                      
             |-------|        |                      |  fp   |    |                                      
             |       |        |                      +-------+  --+                                      
   previous  | frame |        |                      |  r6   |    |                                      
             |       |        |                      +-------+    |                                      
             |-------|    ----+                      |  r5   |    | optional cast: data of previous frame
             |       |                               +-------+    |                                      
    current  | frame |                               |  r4   |    |                                      
             |-------|    ----+                      +-------+  --+                                      
             |       |        -                      |       |    |                                      
             |       |        |                      |  32B  |    |                                      
             |       |        |                      |       |    | not relevant to us                   
             |       |        |                      +-------+    |                                      
      low addr                |        reg sp  --->  |  lr   |    |                                      
                              +----                  +-------+  --+                                      
```

## <a name="reference"></a> Reference

(TBD)
