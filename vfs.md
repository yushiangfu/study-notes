> Study case: Linux version 5.15.0 on AST2500 emulation

## Index

- [Introduction](#introduction)
- [To-Do List](#to-do-list)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

(TBD)

```
+-----------+                                                           
| sys_write |                                                           
+--|--------+                                                           
   |    +------------+                                                  
   +--> | ksys_write |                                                  
        +--|---------+                                                  
           |    +-----------+                                           
           |--> | file_ppos | get file offset                           
           |    +-----------+                                           
           |    +-----------+                                           
           |--> | vfs_write |                                           
           |    +--|--------+                                           
           |       |                                                    
           |       |--> if file has ->write                             
           |       |                                                    
           |       |         call ->write()                             
           |       |               +-----------+                        
           |       |         e.g., | write_mem |                        
           |       |               +-----------+                        
           |       |                                                    
           |       +--> else if file has ->write_iter                   
           |                                                            
           |                +----------------+                          
           |                | new_sync_write |                          
           |                +---|------------+                          
           |                    |    +-----------------+                
           |                    +--> | call_write_iter |                
           |                         +----|------------+                
           |                              |                             
           |                              +--> call ->write_iter()      
           |                                         +-----------------+
           |                                   e.g., | sock_write_iter |
           |                                         +-----------------+
           |                                                            
           +--> update file offset                                      
```

## <a name="to-do-list"></a> To-Do List

(TBD)

## <a name="reference"></a> Reference

(TBD)