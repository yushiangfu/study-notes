> Study case: Linux version 5.15.0 on AST2500 emulation

## Index

- [Introduction](#introduction)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

```
+--------------+                                                                                        
| shmem_create |  prepare an inode of regular file and link it with given dentry                                        
+---|----------+                                                                                        
    |    +-------------+                                                                                
    +--> | shmem_mknod | (regular file)                                                                 
         +---|---------+                                                                                
             |    +-----------------+                                                                   
             |--> | shmem_get_inode |                                                                   
             |    +----|------------+                                                                   
             |         |    +---------------------+                                                     
             |         |--> | shmem_reserve_inode | get a free inode number                             
             |         |    +---------------------+                                                     
             |         |    +-----------+                                                               
             |         |--> | new_inode | allocate a complex or regular inode, init and add to sb's list
             |         |    +-----------+                                                               
             |         |                                                                                
             |         +--> further set up inode and install operation sets of different file types     
             |                                                                                          
             |    +---------------+                                                                     
             +--> | d_instantiate | fill inode info in dentry and link them                             
                  +---------------+                                                                                                                                   
```

## <a name="reference"></a> Reference

(TBD)
