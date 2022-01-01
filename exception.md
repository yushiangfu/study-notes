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
| function calling              | mode switch                          |
| ---                           | ---                                  |
|                               | banked _spsr_ saves _cpsr_           |
| _lr_ saves the return address | banked _lr_ saves the return address |
| _pc_ jumps to function        | _pc_ jumps to vector table           |
| _sp_ adjusts to lower address | banked _sp_ replaces _sp_            |

The left-hand steps are software instructions, while the right-hand side ones are hardware (processor) behaviors.

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
 |   sp   |            | sp_svc | banked register
 +--------+            +--------+
 |   lr   |     +-->   | lr_svc | banked register
 +--------+     |      +--------+
 |   pc   |  ---|                
 +--------+                      
                                 
 +--------+                      
 |  cpsr  |  ---|                
 +--------+     |      +--------+
                +-->   |spsr_svc| banked register
                       +--------+
```

Every exception has its banked stack pointer, and kernel prepares stack space for IRQ/ABT/UND/FIQ, 12 bytes each. 
Like a signal triggers the signal handler, interrupt also behaves the same except the handler works in kernel space. 
Interrupt service routine (ISR) is the formal name, but how can it perform the complicated logic in such a small space? It doesn't. 
Kernel quickly switches to SVC mode for ISR after a few simple register operations executed in that exception mode..
Let's introduce the vector table first in the next section.

- Code flow

```
+------------+                
| setup_arch |                
+--|---------+                
   |    +-----------------+   
   +--> | setup_processor |   
        +----|------------+   
             |    +----------+
             +--> | cpu_init | prepare stack space for irq/abt/und/fiq for one processor
                  +----------+
```

- Figure

```
               stacks                                                     
                                     +-- irq[0] = for r0                      
               +-----+               |                                    
               | irq | 4 bytes * 3 --|-- irq[1] = for lr_irq (backup of pc)
               +-----+               |                                    
               | abt |               +-- irq[2] = for spsr_irq, (backup of cpsr)
   CPU0        +-----+                                                    
               | und |                                                    
               +-----+                                                    
               | fiq |                                                    
-----------------------------------                                       
               | irq |                                                    
               +-----+                                                    
               | abt |                                                    
   CPU1        +-----+                                                    
               | und |                                                    
               +-----+                                                    
               | fiq |                                                    
               +-----+                                                             
```

## <a name="vector-table"></a> Vector Table

The kernel allocates two pages during boot time, copies the vector table from the kernel section, and sets up the mapping. 
Whenever an exception happens, the processor automatically has the _pc_ point to the corresponding entry in the vector table, located at 0xFFFF_0000. 
Ignore the _kuser helper_ part in the below figure because I haven't looked into it yet.

- Figure - vector table in memory

```                                                                                                                      
 virtual addr                                                                               
                                                                                            
 0xFFFF_0000  +--------------------------+                                                  
              |    goto vector_rst       |  \                                               
 0xFFFF_0004  +--------------------------+   -                                              
              |    goto vector_und       |   |                                              
 0xFFFF_0008  +--------------------------+   |                                              
              |    goto vector_swi       |   |                                              
 0xFFFF_000C  +--------------------------+   |                                              
              |    goto vector_pabt      |   |                                              
 0xFFFF_0010  +--------------------------+   | vector                                       
              |    goto vector_dabt      |   | table                                        
 0xFFFF_0014  +--------------------------+   |                                              
              |    goto vector_addrexcptn|   |                                              
 0xFFFF_0018  +--------------------------+   |                                              
              |    goto vector_irq       |   |                                              
 0xFFFF_001C  +--------------------------+   -                                              
              |    goto vector_fiq       |  /                                               
 0xFFFF_0020  +--------------------------+                                                  
                                                                                            
 0xFFFF_0F60  +--------------------------+                                                  
              | __kuser_helper_start     |  \                                               
 0xFFFF_0FA0  +--------------------------+   |                                              
              | __kuser_memory_barrier   |   |                                              
 0xFFFF_0FC0  +--------------------------+   |                                              
              | __kuser_cmpxchg          |   | kuser                                        
 0xFFFF_0FE0  +--------------------------+   | helpers                                      
              |   HW TLS instruction     |   |                                              
 0xFFFF_0FFC  +--------------------------+   |                                              
              | __kuser_helper_version   |  /                                               
 0xFFFF_1000  +--------------------------+                                                  
              |     __stubs_start        | the address of vector_swi is saved at 0xFFFF_1000
 0xFFFF_12AC  +--------------------------+                                                  
```

- Code flow

```
+-------------+                                                                                  
| paging_init |                                                                                  
+---|---------+                                                                                  
    |    +-----------------+                                                                     
    +--> | devicemaps_init |                                                                     
         +----|------------+                                                                     
              |                                                                                  
              |--> allocate two page frames                                                      
              |                                                                                  
              |    +-----------------+                                                           
              |--> | early_trap_init | copy vector table and kuser helper to that allocated space
              |    +-----------------+                                                           
              |    +----------------+                                                            
              +--> | create_mapping | specify the virtual address started from 0xFFFF_0000       
                   +----------------+                                                            
```

Take _vector_irq_,_vector_pabt_,_vector_dabt_, _vector_und_ have similar logic at the beginning, and let's take _vector_irq_ for example:
- After _pc_ moves to _vector_irq_ from vector table, kernel saves current _r0_, _lr_irq_, and _spsr_irq_ to the small stack.
- Determine which mode we came from by checking _spsr_irq_ and going to either *__irq_usr* or *__irq_sve*.

- Code flow

```
+------------+                                                     
| vector_irq |                                                     
+--|---------+                                                     
   |                                                               
   |---> save r0, lr_irq, spsr_irq to irq stack (size: 4 bytes * 3)
   |                                                               
   |---> switch to SVC mode                                        
   |                                                               
   |---> determine which mode we comes from                        
   |                                                               
   |---> save irq stack address to r0                              
   |                                                               
   |---> if we were at USR mode                                    
   |                                                               
   |         +-----------+                                         
   |         | __irq_usr |                                         
   |         +-----------+                                         
   |                                                               
   +---> elif we were at SVC mode                                  
                                                                   
             +-----------+                                         
             | __irq_svc |                                         
             +-----------+                                                                                                                     
```

- Code flow of ___irq_usr_

```
+-----------+                                                                                                          
| __irq_usr |                                                                                                          
+--|--------+                                                                                                          
   |    +-----------+                                                                                                  
   +--> | usr_entry | store r0 ~ r12, sp_usr, lr_usr, old pc to sp_svc                                                 
   |    +-----------+                                                                                                  
   |    +-------------+                                                                                                
   |--> | irq_handler |                                                                                                
   |    +---|---------+                                                                                                
   |        |    +-----------------+                                                                                   
   |        +--> | handle_arch_irq | e.g. avic_handle_irq                                                              
   |             +-----------------+                                                                                   
   |    +-----------------+                                                                                            
   |--> | get_thread_info | get thread_info                                                                            
   |    +-----------------+                                                                                            
   |    +----------------------+                                                                                       
   +--> | ret_to_user_from_irq |                                                                                       
        +-----|----------------+                                                                                       
              |                                                                                                        
              |--> check thread_info->flags to see if there's pending work                                             
              |                                                                                                        
              |--> if there is                                                                                         
              |                                                                                                        
              |        +-------------------+                                                                           
              |        | slow_work_pending |                                                                           
              |        +----|--------------+                                                                           
              |             |    +-----------------+                                                                   
              |             |--> | do_work_pending | schedule or handle signal based the flags                         
              |             |    +-----------------+                                                                   
              |             |                                                                                          
              |             |--> if everything's fine                                                                  
              |             |                                                                                          
              |             |        +-----------------+                                                               
              |             |        | no_work_pending | restore registers for USR mode, and switch to it              
              |             |        +-----------------+                                                               
              |             |                                                                                          
              |             +--> else (syscall needs restart)                                                          
              |                                                                                                        
              |                      +---------------+                                                                 
              |                      | local_restart | handle syscall, restore registers for USR mode, and switch to it
              |                      +---------------+                                                                 
              |                                                                                                        
              +--> else                                                                                                
                       +-----------------+                                                                             
                       | no_work_pending | restore registers for USR mode, and switch to it                            
                       +-----------------+                                                                             
```

- Code flow of ___irq_svc_

```
+-----------+                                            
| __irq_svc |                                            
+--|--------+                                            
   |    +-----------+                                    
   |--> | svc_entry |                                    
   |    +--|--------+                                    
   |       |                                             
   |       |--> save registers to stack for later restore
   |       |                                             
   |       |    +-----------------+                      
   |       +--> | get_thread_info | get thread_info      
   |            +-----------------+                      
   |    +-------------+                                  
   +--> | irq_handler |                                  
        +---|---------+                                  
            |    +-----------------+                     
            +--> | handle_arch_irq | e.g. avic_handle_irq
                 +-----------------+                     
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


