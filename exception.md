> Study case: Linux version 5.10.46 on OpenBMC

## Index

- [Introduction](#introduction)
- [Vector Table](#vector-table)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

(TBD)

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

## <a name="reference"></a> Reference

[H. Nandish, Software Interrupt routine on ARM](https://lnxblog.github.io/2019/07/06/swi-routine-arm.html)


