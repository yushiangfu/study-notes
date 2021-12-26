> Study case: Linux version 5.10.46 on OpenBMC

## Index

- [Introduction](#introduction)
- [Registers](#registers)
- [Vector Table](#vector-table)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

A userspace program may contain hundreds or thousands of instructions that manipulate processor registers and system memory. 
Instead of simple operation, usually, applications need to send the request to the kernel for other system-level services run in another CPU mode. 
External hardware like timer triggers interrupts in the meantime, and the CPU enters the corresponding processor mode for further handling. 
These are called _USER_, _SUPERVISOR_, and _INTERRUPT_ modes respectively on ARM processors and there are also _ABORT_, _UNDEFINED_, and _FIQ_ modes. 
Though it's highly related to architecture design, other processors have a similar concept of the exception mechanism.

## <a name="registers"></a> Registers

On 32-bit ARM, there are thirteen generic registers (r0 ~ r12), the stack pointer (r13 or sp), link register (r14 or lr), and the program counter (pc).
- r# - serves general purposes such as mathematic operation and memory access.
- sp - points to the current stack. (Each CPU mode has its banked sp)
- lr - stores the return address in conventional function calling or CPU mode switch. (Each CPU mode has its banked lr)
- pc - the processor fetches the instruction from where pc points.
- cpsr - current program status register
- spsr - saved program status register

Let's compare the steps between function calling and mode switch:
- function calling
  - _lr_ saves the return address 
  - _pc_ jumps to target function
  - _sp_ adjusts to lower address if stack growth direction is downward.
- mode switch
  - backup _cpsr_ to banked _spsr_ (processor behavior)
  - backup _pc_ to banked _lr_ (processor behavior)
  - meanwhile, banked _sp_ points to the stack prepared for the switched mode

There are still numerous registers and co-processors, but we won't delve into them.

- Figure - mode switch

```
    User               Supervisor
                                 
 +--------+                      
 |   r0   |                      
 +--------+                      
      -                          
      -                          
      -                          
      -                          
      -                          
 +--------+                      
 |  r12   |                      
 +--------+            +--------+
 |   sp   |            | sp_svc |
 +--------+            +--------+
 |   lr   |     +-->   | lr_svc |
 +--------+     |      +--------+
 |   pc   |  ---|                
 +--------+                      
                                 
 +--------+                      
 |  cpsr  |  ---|                
 +--------+     |      +--------+
                +-->   |spsr_svc|
                       +--------+
```

## <a name="vector-table"></a> Vector Table

(TBD)

```                                                                                                                      
 virtual addr                                                                             
                                                                                          
 0xFFFF_0000  +------------------------+                                                  
              |    goto vector_rst     |  \                                               
 0xFFFF_0004  +------------------------+   -                                              
              |    goto vector_und     |   |                                              
 0xFFFF_0008  +------------------------+   |                                              
              |    goto vector_swi     |   |                                              
 0xFFFF_000C  +------------------------+   |                                              
              |    goto vector_pabt    |   |                                              
 0xFFFF_0010  +------------------------+   | vector                                       
              |    goto vector_dabt    |   | table                                        
 0xFFFF_0014  +------------------------+   |                                              
              |    goto vector_xxx     |   |                                              
 0xFFFF_0018  +------------------------+   |                                              
              |    goto vector_irq     |   |                                              
 0xFFFF_001C  +------------------------+   -                                              
              |    goto vector_fiq     |  /                                               
 0xFFFF_0020  +------------------------+                                                  
                                                                                          
 0xFFFF_0F60  +------------------------+                                                  
              | __kuser_helper_start   |  \                                               
 0xFFFF_0FA0  +------------------------+   |                                              
              | __kuser_memory_barrier |   |                                              
 0xFFFF_0FC0  +------------------------+   |                                              
              | __kuser_cmpxchg        |   | kuser                                        
 0xFFFF_0FE0  +------------------------+   | helpers                                      
              |   HW TLS instruction   |   |                                              
 0xFFFF_0FFC  +------------------------+   |                                              
              | __kuser_helper_version |  /                                               
 0xFFFF_1000  +------------------------+                                                  
              |     __stubs_start      | the address of vector_swi is saved at 0xFFFF_1000
 0xFFFF_12AC  +------------------------+                                                                 
```

- Figure

```
  low addr  +--+-----------+        
            |  |thread_info|        
            |  +-----------+        
        ^   |           |           
        |   |           |           
        |   |           |           
        |   |           |           
        |   |           |           
        |   +-----------+           
        |   |           |           
        |   |           |           
 stack  |   |           |           
            |  +-----------+        
            |  |  pt_regs  |        
            |  +-----------+        
            |  |  reserved | 8 bytes
 high addr  +--+-----------+        
```

## <a name="reference"></a> Reference

[H. Nandish, Software Interrupt routine on ARM](https://lnxblog.github.io/2019/07/06/swi-routine-arm.html)


