> Study case: Linux version 5.15.0 on AST2500 emulation

## Index

- [Introduction](#introduction)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

Working on an embedded system, we are very likely to see the kernel panic or exception from time to time. 
The information includes register values, memory content around stack pointer, and function backtrace. 
It might not be helpful because it's all about the victim instead of the culprit. 
Since kernel space is a large shared memory region, everyone can be the suspect that pollutes the data on purpose or accident. 
But anyway, it's better to have some clues than to investigate from nowhere. 
We can issue the below command to object dump the abundant information of the vmlinux. (warning! ABUNDANT information!)

```
arm-linux-gnueabi-objdump -ldS vmlinux
```

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

Looking into the dumped file, we can see that:
1. Before doing anything, the regular function pushes some registers into the stack and recovers before returning.
2. The saved register set commonly includes **fp**, **ip**, **lr**, and **pc**.

| Alias | Register | Role                 |
| ---   | ---      | ---                  |
| fp    | r11      | frame pointer        |
| ip    | r12      | scratch register (?) |
| sp    | r13      | stack pointer        |
| lr    | r14      | link register        |
| pc    | r15      | program counter      |

Kernel stack on the 32-bit ARM architecture is a two-page space for each task to run the kernel logic, such as syscall and interrupt handler.

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

Generally speaking, functions utilize stack as storage for local variables and CPU registers. 
We can regard the stack as a series of frames, and each frame is the storage of a function.

```
                                        stack
                                        (mem)

                              high addr
                                      |   -   |
                                      |   -   |
+----------+                          |-------|
| function |  ─────────────────────►  | frame |
+--|-------+                          |-------|
   |    +----------+                  |       |
   +--> | function |  ─────────────►  | frame |
        +--|-------+                  |       |
           |    +----------+          |-------|
           +--> | function |  ─────►  |       |
                +----------+          | frame |
                                      |-------|
                                      |       |
                                      |       |
                                      |       |
                                      |       |
                               low addr
```
  
Given any frame pointer, we can quickly get the frame pointer of the previous frame and dump the information recursively till the very first frame.

```
               stack                                            fixed cast: info of previous frame                     
               (mem)              -                  +-------+  --+                                                    
                                 /     reg fp  --->  |  pc   |    | <--- where we are of the current frame             
     high addr                  /                    +-------+    |                                                    
             |   -   |         /                     |  lr   |    | <--- where we come from of the previous frame      
             |   -   |        /                      +-------+    |                                                    
             |-------|       /                       |  sp   |    | <--- where the stack end is of the previous frame  
 even older  | frame |      /                        +-------+    |                                                    
             |-------|     /                         |  fp   |    | <--- where the frame start is of the previous frame
             |       |    /                          +-------+  --+                                                    
   previous  | frame |   /                           |  r6   |    |                                                    
             |       |  /                            +-------+    |                                                    
             |-------|                               |  r5   |    | optional cast: data of previous frame              
             |       |                               +-------+    |                                                    
    current  | frame |                               |  r4   |    |                                                    
             |-------|                               +-------+  --+                                                    
             |       |  \                            |       |    |                                                    
             |       |   \                           |       |    |                                                    
             |       |    \                          |       |    | not relevant to us                                 
             |       |     \                         +-------+    |                                                    
      low addr              \          reg sp  --->  |  lr   |    |                                                    
                             -                       +-------+  --+         
```

<details>
  <summary> Code Trace </summary>

```
+----------------+                                                                                                
| dump_backtrace |                                                                                                
+---|------------+                                                                                                
    |                                                                                                             
    |--> determine frame pointer & mode                                                                           
    |                                                                                                             
    |    +-------------+                                                                                          
    +--> | c_backtrace |                                                                                          
         +---|---------+                                                                                          
             |                                                                                                    
             +--> save registers to stack                                                                         
             |                                                                                                    
             +--> for each frame                                                                                  
             |                                                                                                    
             |------> if frame pointer isn't valid, return                                                        
             |                                                                                                    
             |        +----------------------+                                                                    
             |------> | dump_backtrace_entry |                                                                    
             |        +-----|----------------+                                                                    
             |              |                                                                                     
             |              +--> print one line                                                                   
             |                                                                                                    
             |                   e.g., [<80942e80>] (dump_backtrace) from [<80109348>] (arch_cpu_idle+0x3c/0x4c)  
             |                                                                                                    
             |------> if condition met                                                                            
             |                                                                                                    
             |            +--------------------+                                                                  
             |----------> | dump_backtrace_stm |                                                                  
             |            +----|---------------+                                                                  
             |                 |                                                                                  
             |                 +--> dump registers                                                                
             |                                                                                                    
             |                      e.g., r10:00c5387d r9:410fb767 r8:8810c000 r7:ffffffff r6:00c0387d r5:00000051
             |                                                                                                    
             |------> if next frame pointer is null, return                                                       
             |                                                                                                    
             +--> restore registers from stack                                                                    
```

</details>
  
## <a name="reference"></a> Reference

- My friend Sneaker's slide
