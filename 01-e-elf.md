## Index

- [Introduction](#introduction)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

While EXE is the executable format on Windows OS, ELF format applies to applications and other files on Linux systems.

- Object file: base compiled unit
- Executable: running program
- Shared lib: dynamic library
- Coredump: analysis of a task

Let's introduce the ELF format by the famous 'Hello, World!' example.

```
#include <stdio.h>

void main()
{
    printf("Hello, World!\n");
}
```

Program header of a.out

```
bobfu@bobfu-Vostro-5402:~/workspace/oblinux$ readelf -l a.out

Elf file type is EXEC (Executable file)
Entry point 0x1030c
There are 9 program headers, starting at offset 52

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  EXIDX          0x000498 0x00010498 0x00010498 0x00008 0x00008 R   0x4
  PHDR           0x000034 0x00010034 0x00010034 0x00120 0x00120 R   0x4
  INTERP         0x000154 0x00010154 0x00010154 0x00013 0x00013 R   0x1
      [Requesting program interpreter: /lib/ld-linux.so.3]
  LOAD           0x000000 0x00010000 0x00010000 0x004a4 0x004a4 R E 0x10000
  LOAD           0x000f10 0x00020f10 0x00020f10 0x00118 0x0011c RW  0x10000
  DYNAMIC        0x000f18 0x00020f18 0x00020f18 0x000e8 0x000e8 RW  0x4
  NOTE           0x000168 0x00010168 0x00010168 0x00044 0x00044 R   0x4
  GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RW  0x10
  GNU_RELRO      0x000f10 0x00020f10 0x00020f10 0x000f0 0x000f0 R   0x1
...
```

Program header of ld-linux.so.3

```
obfu@bobfu-Vostro-5402:~/workspace/study-notes$ readelf -l ld-linux.so.3 

Elf file type is DYN (Shared object file)
Entry point 0xa10
There are 7 program headers, starting at offset 52

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  EXIDX          0x0253c0 0x000253c0 0x000253c0 0x000c8 0x000c8 R   0x4
  LOAD           0x000000 0x00000000 0x00000000 0x25488 0x25488 R E 0x10000
  LOAD           0x026200 0x00036200 0x00036200 0x016d8 0x017c0 RW  0x10000
  DYNAMIC        0x026ef4 0x00036ef4 0x00036ef4 0x000c8 0x000c8 RW  0x4
  NOTE           0x000114 0x00000114 0x00000114 0x00024 0x00024 R   0x4
  GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RW  0x10
  GNU_RELRO      0x026200 0x00036200 0x00036200 0x00e00 0x00e00 R   0x1
  ...
```

```
+-----------+                                                                               
| de_thread | ensure no other threads in group, and arg task assumes the leader             
+--|--------+                                                                               
   |                                                                                        
   |--> return if no other thread in group                                                  
   |                                                                                        
   |    +-------------------+                                                               
   |--> | zap_other_threads | for each other thread in group, set KILL signal and wake it up
   |    +-------------------+                                                               
   |                                                                                        
   |--> wait till other non-leader tasks exit                                               
   |                                                                                        
   |--> if arg task isn't group leader                                                      
   |                                                                                        
   |------> wait till group leader exits as well                                            
   |                                                                                        
   |------> assume its pid and become the group leader                                      
   |                                                                                        
   |------> if there's a tracer tracking the original leader                                
   |                                                                                        
   |            +------------------+                                                        
   +----------> | __wake_up_parent | notify tracer of the death                             
                +------------------+                                                        
```

```
+------------+                                                                                 
| unshare_fd | clone fdtable from current task, and assign to current task?                    
+--|---------+                                                                                 
   |                                                                                           
   |--> if flash has specified 'clone'                                                         
   |                                                                                           
   |        +--------+                                                                         
   +------> | dup_fd | clone fdtable from current task (both fd arrays point to the same files)
            +-|------+                                                                         
              |    +------------------+                                                        
              |--> | kmem_cache_alloc | allocate 'files_struct' which includes fdtable         
              |    +------------------+                                                        
              |                                                                                
              |--> set up the new fdtable                                                      
              |                                                                                
              +--> copy fds from old to new                                                    
```

## <a name="reference"></a> Reference

(TBD)




