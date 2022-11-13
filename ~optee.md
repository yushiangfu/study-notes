> Study case: Linux version 5.15.0 on OpenBMC

## Index

- [Introduction](#introduction)
- [To-Do List](#to-do-list)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

(None)

```
+---------------------------+                                                                    
| TEEC_RegisterSharedMemory |                                                                    
+------|--------------------+                                                                    
       |                                                                                         
       |--> if context has reg_mem?                                                              
       |                                                                                         
       |        +-------------------+                                                            
       |        | teec_shm_register | set up shm@kernel for shm@user                             
       |        +-------------------+ (read-only buffer might fail and fallback to shadow buffer)
       |                                                                                         
       +--> else                                                                                 
                                                                                                 
                +----------------+                                                               
                | teec_shm_alloc | instead of the caller, the pool manager helps allocate buffer 
                +----------------+                                                               
                                                                                                 
                map to shadow buffer                                                             
```

## <a name="to-do-list"></a> To-Do List

(None)

## <a name="reference"></a> Reference

(None)

