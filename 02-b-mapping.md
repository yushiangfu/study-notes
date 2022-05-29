## Index

- [Introduction](#introduction)
- [Reference](#reference)
- [Kmap](#kmap)

## <a name="introduction"></a> Introduction

Virtual mapping relies on page table, MMU, and TLB:

- Page table: It's a collection of entries that specify the virtual-to-physical mapping.
- MMU: the hardware component that does the lookup behavior.
- TLB: cache for lookup
- 
Page table, a.k.a. PGD table, is a 16Ks-size area including 4096 PMD entries. Each PMD can represent one of the below meanings.

- Not mapped
- Section (1M) mapping
- A pointer to the 2nd level page table

If the mapped region is more significant than 1M, then the kernel adopts the section mapping or further allocates a PTE table for page mapping.

```
        1st level                                  2nd level
        PGD table                                  PTE table

        +-------+                                 ----------------+
        |   PMD0| physical addr (1M region)         |PTE0  |      |
   PGD0 |  ------                                   +------+      |
        |   PMD1|                                      -          |
        +-------+                                      -          |
        |   PMD0| ---+                              +------+      |
   PGD1 |  ------    |                              |PTE255|      |
        |   PMD1| -+ |                            ------------    | for SW (Linux) to check attributes
        +-------+  | |                              |PTE0  |      |
            -      | |                              +------+      |
            -      | |                                 -          |
            -      | |                                 -          |
            -      | |                              +------+      |
            -      | |                              |PTE255|      |
            -      | |                            ------------  --+
            -      | +--------------------------->  |PTE0  |      |
            -      |                                +------+      |
            -      |                                   -          |
            -      |                                   -          |
            -      |                                +------+      |
            -      |                                |PTE255|      |
            -      |                              ------------    | for HW (MMU) to look up
            -      +----------------------------->  |PTE0  |      |
            -                                       +------+      |
            -                                          -          |
        +-------+                                      -          |
        |   PMD0| none                              +------+      |
PGD2047 |  ------                                   |PTE255| physical addr (4K region)
        |   PMD1|                                 ----------------+
        +-------+
```

(Should I move the introduction from memory.md to here?)

```
               bit 31       20 19   12 11        0 
                  +-----------+-------+-----------+
 virtual address  |  12 bits  |8 bits |  12 bits  |
                  +-----------+-------+-----------+
                    1st index             offset   
                              2nd index            
```

There's the initial page table for the kernel to create mappings, and it's shared by all the kernel threads. 

```
bobfu@bobfu-Vostro-5402:~/workspace/oblinux$ head System.map
00000018 A cpu_v6_suspend_size
00000024 A cpu_ca15_suspend_size
00000024 A cpu_ca8_suspend_size
00000024 A cpu_v7_bpiall_suspend_size
00000024 A cpu_v7_suspend_size
0000002c A cpu_ca9mp_suspend_size
80004000 A swapper_pg_dir    <--------------------- fixed address
80008000 T _text
80008000 T stext
80008080 t __create_page_tables
```

For the regular userspace process, it has its own page table. 
For a multi-threaded process, all the threads share the same page table, and therefore they see the same virtual space. 
The kernel simply has the co-processor (TTBR0 or TTBR1?) point to the corresponding page table when a context switch happens. 
And that instructs MMU on where to lookup from.

```
                                  process                         process           
+---------+                     +---------+                     +---------+         
| kthread | ---+                |+-------+|                     |+-------+|         
+---------+    |                ||thread |-----+                ||thread |-----+    
+---------+    |                |+-------+|    |                |+-------+|    |    
| kthread | ---|                |         |    |                ||thread ------+    
+---------+    |                |         |    |                |+-------+|    |    
+---------+    |                |         |    |                ||thread ------+    
| kthread | ---|                |         |    |                |+-------+|    |    
+---------+    |                +---------+    |                +---------+    |    
               |                               |                               |    
               |                               |                               |    
               v                               v                               v    
          page table                      page table                      page table
          +--------+                      +--------+                      +--------+
          |        |                      |        |                      |        |
          |        |                      |        |                      |        |
          |        |                      |        |                      |        |
          |        |                      |        |                      |        |
          |        |                      |        |                      |        |
          |        |                      |        |                      |        |
          |        |                      |        |                      |        |
          +--------+                      +--------+                      +--------+
```

For mapping of kernel space, not much management is involved since it's not necessary. 
But a few points need to be considered for userspace mapping.

- lookup of the available virtual region for mapping
- region attributes, e.g., read, write, execute
- region extension, e.g., heap, stack

And the kernel utilizes **struct vma** (virtual memory area) to represent each region in user space for VM management. 
The common areas are:

- stack
  - for function call
  - where local variables exist
  - extend toward low address
- heap
  - for malloc()
  - ext end toward high address
- read & execute region of executable, linker, or library
  - e.g., compiled code
- read & write region of executable, linker, or library
  - e.g., global data
- read-only region of executable, linker, or library
  - e.g., constant?

  

```
                                                     virtual address                                
                                  +---------    +-----------------------+                           
                                  |             |                       |                           
                                  |             |                       |                           
                                  |             |||||||||||||||||||||||||  vma
                                  |             |                       |                           
                                  |             |                       |                           
                                  |             |                       |                           
            page table            |             |||||||||||||||||||||||||  vma                      
                                  |             |                       |                           
  low addr +---------+    --------+             |                       |                           
           |         |                          |                       |                           
           |   user  |                          |||||||||||||||||||||||||  vma                      
           |  space  |                          |                       |                           
           |         |                          |                       |                           
         ---------------  -----------------   -----------------------------                         
           |         |                          |                       |                           
           |  kernel |                          |                       |                           
           |  space  |                          |                       |                           
           |         |                          |                       |                           
 high addr +---------+    --------+             |                       |                           
                                  |             |                       |                           
                                  |             |                       |                           
                                  |             |                       |                           
                                  |             |                       |                           
                                  |             |                       |                           
                                  |             |                       |                           
                                  |             |                       |                           
                                  +---------    +-----------------------+                           
```

When a user executes the a.out, the shell first fork into two identical tasks and the following logic differs for parent and child tasks. 
The child task triggers execve() to load the target executable into memory without much surprise.

```
          parent task                       child task    
                                                          
                                                          
        fork()                                            
      +----------------+                +----------------+
      | if it's parent |                | if it's parent |
      |                |                |                |
----> |     whatever   |                |     whatever   |
      |                |                |                |
      | else           |                | else           |
      |                |                |                |
      |     execve()   |          ----> |     execve()   |
      |      ...       |                |      ...       |
      +----------------+                +----------------+
```

Once the kernel receives the execve() request, the child task switches to the newly generated mm structure and starts to install various mappings.

1. [kernel] prepare the stack
2. [kernel] map segments from a.out
3. [kernel] map segments from the specified interpreter
4. [kernel] install unique mappings (sigpage, vvar, and vdso)

And deliver the control to the interpreter instead of our target executable. 
The linker is actually also an executable rather than a regular shared library. 
Linker helps to load other shared libraries specified by the a.out during compilation.

6. [linker] map c library
7. [linker] extend heap size

Finally, the flow goes to the **main** function and prints out the famous 'Hello, World!' string.

```
                             +--- vma                    
                             |                           
                             |--- vma                    
                  old        |     -                     
                 +----+      |     -                     
                 | mm | -----+     -                     
                 +----+      +--- vma                    
 child                                                   
+------+                                                 
| task |  ---+            low addr                       
+------+     |    new        +--- vma  --|               
             |   +----+      |--- vma    |  a.out        
             +-- | mm | -----|--- vma  --+               
                 +----+      |                           
                             |--- vma  ---  heap         
                             |                           
                             |--- vma  --+               
                             |--- vma    |               
                             |--- vma    |  libc         
                             |--- vma    |               
                             |--- vma  --+               
                             |                           
                             |--- vma  --+               
                             |--- vma    |  linker       
                             |--- vma    |               
                             |--- vma  --+               
                             |                           
                             |--- vma  ---  stack        
                             |                           
                             |--- vma  --|               
                             |--- vma    |  arch specific
                             +--- vma  --+               
                         high addr                       
```

```
  +---------------------------------------- virtual address range      
  |                 +---------------------- p:private, s:share         
  |                 |    +----------------- page offset                
  |                 |    |        +-------- major:minor                
  |                 |    |        |     +-- inode                      
  |                 |    |        |     |                              
  |                 |    |        |     |                              
  +---------------- +--- +------- +---- +--                            
  00010000-00011000 r-xp 00000000 1f:05 915        /home/root/a.out    
  00020000-00021000 r--p 00000000 1f:05 915        /home/root/a.out    
  00021000-00022000 rw-p 00001000 1f:05 915        /home/root/a.out    
  000ac000-000cd000 rw-p 00000000 00:00 0          [heap]              
  76db8000-76f11000 r-xp 00000000 1f:04 426        /lib/libc.so.6      
  76f11000-76f20000 ---p 00159000 1f:04 426        /lib/libc.so.6      
  76f20000-76f22000 r--p 00158000 1f:04 426        /lib/libc.so.6      
  76f22000-76f24000 rw-p 0015a000 1f:04 426        /lib/libc.so.6      
  76f24000-76f2d000 rw-p 00000000 00:00 0                              
  76f30000-76f56000 r-xp 00000000 1f:04 421        /lib/ld-linux.so.3  
  76f64000-76f66000 rw-p 00000000 00:00 0                              
  76f66000-76f67000 r--p 00026000 1f:04 421        /lib/ld-linux.so.3  
  76f67000-76f68000 rw-p 00027000 1f:04 421        /lib/ld-linux.so.3  
  7e99e000-7e9bf000 rw-p 00000000 00:00 0          [stack]             
  7eb92000-7eb93000 r-xp 00000000 00:00 0          [sigpage]           
  7eb93000-7eb94000 r--p 00000000 00:00 0          [vvar]              
  7eb94000-7eb95000 r-xp 00000000 00:00 0          [vdso]              
  ffff0000-ffff1000 r-xp 00000000 00:00 0          [vectors]           
```

<details>
  <summary> Mapping and strace log </summary>

```
 00010000-00011000 r-xp 00000000 1f:05 915        /home/root/a.out    | 2. [kernel] map 1st segment of a.out
                                                                                                            
                                                                                                            
 00020000-00021000 r--p 00000000 1f:05 915        /home/root/a.out    | 3. [kernel] map 2nd segment of a.out     | 14. [linker] mprotect 'r'
 00021000-00022000 rw-p 00001000 1f:05 915        /home/root/a.out    |                                                                        
                                                                                                                                               
                                                                                                                                               
 000ac000-000cd000 rw-p 00000000 00:00 0          [heap]              | 16. [linker] extend heap size from 0
    
 76db8000-76f11000 r-xp 00000000 1f:04 426        /lib/libc.so.6      | 8. [linker] map file (libc, rx)                                        
                                                                      |                                                                        
 76f11000-76f20000 ---p 00159000 1f:04 426        /lib/libc.so.6      |      | 9. [linker] mprotect 'none'                                     
                                                                      |                                                                        
 76f20000-76f22000 r--p 00158000 1f:04 426        /lib/libc.so.6      |      | 10. [linker] map file (libc, rw)  | 13. [linker] mprotect 'r'
 76f22000-76f24000 rw-p 0015a000 1f:04 426        /lib/libc.so.6      |      |                                                                 
                                                                      |                                                                        
 76f24000-76f2d000 rw-p 00000000 00:00 0                              |      | 11. [linker] map anon (rw)                                      
                                                                                                                                               
 76f30000-76f56000 r-xp 00000000 1f:04 421        /lib/ld-linux.so.3  | 4. [kernel] map 1st segment of linker                                  
                                                                                                                                               
 76f64000-76f66000 rw-p 00000000 00:00 0                              | 7. [linker] map anon (rw)                | 12. [linker] set tls, tid, robust list
                                                                                                                                              
 76f66000-76f67000 r--p 00026000 1f:04 421        /lib/ld-linux.so.3  | 5. [kernel] map 2nd segment of linker    | 15. [linker] mprotect 'r'
 76f67000-76f68000 rw-p 00027000 1f:04 421        /lib/ld-linux.so.3  |                                                                        
                                                                                                                                               
 7e99e000-7e9bf000 rw-p 00000000 00:00 0          [stack]             | 1. [kernel] prepare stack                                              
                                                                                                                                               
 7eb92000-7eb93000 r-xp 00000000 00:00 0          [sigpage]           | 6. [kernel] prepare sigpage, vvar, and vdso                            
 7eb93000-7eb94000 r--p 00000000 00:00 0          [vvar]              |                                                                        
 7eb94000-7eb95000 r-xp 00000000 00:00 0          [vdso]              |                                                                        
                                                                                                                                               
 ffff0000-ffff1000 r-xp 00000000 00:00 0          [vectors]           | 0. [kernel] it's already there during boot time                        
```
    
```
execve("./a.out", ["./a.out"], [/* 16 vars */]) = 0
brk(0)                                  = 0xac000
mmap2(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x76f64000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_LARGEFILE|O_CLOEXEC) = 3
syscall_397(0x3, 0x76f52a54, 0x1800, 0x7ff, 0x7e9be058, 0x7e9be188) = 0
mmap2(NULL, 6905, PROT_READ, MAP_PRIVATE, 3, 0) = 0x76f60000
close(3)                                = 0
openat(AT_FDCWD, "/lib/tls/v6l/vfp/libc.so.6", O_RDONLY|O_LARGEFILE|O_CLOEXEC) = -1 ENOENT (No such file or directory)
syscall_397(0xffffff9c, 0x7e9be130, 0x800, 0x7ff, 0x7e9be000, 0x7e9be198) = -1 (errno 2)
openat(AT_FDCWD, "/lib/tls/v6l/libc.so.6", O_RDONLY|O_LARGEFILE|O_CLOEXEC) = -1 ENOENT (No such file or directory)
syscall_397(0xffffff9c, 0x7e9be130, 0x800, 0x7ff, 0x7e9be000, 0x7e9be198) = -1 (errno 2)
openat(AT_FDCWD, "/lib/tls/vfp/libc.so.6", O_RDONLY|O_LARGEFILE|O_CLOEXEC) = -1 ENOENT (No such file or directory)
syscall_397(0xffffff9c, 0x7e9be130, 0x800, 0x7ff, 0x7e9be000, 0x7e9be198) = -1 (errno 2)
openat(AT_FDCWD, "/lib/tls/libc.so.6", O_RDONLY|O_LARGEFILE|O_CLOEXEC) = -1 ENOENT (No such file or directory)
syscall_397(0xffffff9c, 0x7e9be130, 0x800, 0x7ff, 0x7e9be000, 0x7e9be198) = -1 (errno 2)
openat(AT_FDCWD, "/lib/v6l/vfp/libc.so.6", O_RDONLY|O_LARGEFILE|O_CLOEXEC) = -1 ENOENT (No such file or directory)
syscall_397(0xffffff9c, 0x7e9be130, 0x800, 0x7ff, 0x7e9be000, 0x7e9be198) = -1 (errno 2)
openat(AT_FDCWD, "/lib/v6l/libc.so.6", O_RDONLY|O_LARGEFILE|O_CLOEXEC) = -1 ENOENT (No such file or directory)
syscall_397(0xffffff9c, 0x7e9be130, 0x800, 0x7ff, 0x7e9be000, 0x7e9be198) = -1 (errno 2)
openat(AT_FDCWD, "/lib/vfp/libc.so.6", O_RDONLY|O_LARGEFILE|O_CLOEXEC) = -1 ENOENT (No such file or directory)
syscall_397(0xffffff9c, 0x7e9be130, 0x800, 0x7ff, 0x7e9be000, 0x7e9be198) = -1 (errno 2)
openat(AT_FDCWD, "/lib/libc.so.6", O_RDONLY|O_LARGEFILE|O_CLOEXEC) = 3
read(3, "\177ELF\1\1\1\0\0\0\0\0\0\0\0\0\3\0(\0\1\0\0\0\340\31\2\0004\0\0\0"..., 512) = 512
syscall_397(0x3, 0x76f52a54, 0x1800, 0x7ff, 0x7e9bdff0, 0x7e9be198) = 0
mmap2(NULL, 1525364, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x76db8000
mprotect(0x76f11000, 61440, PROT_NONE)  = 0
mmap2(0x76f20000, 16384, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x158000) = 0x76f20000
mmap2(0x76f24000, 34420, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x76f24000
close(3)                                = 0
set_tls(0x76f64d50, 0x76f654c8, 0x76f66000, 0x76f65448, 0x76f64d50) = 0
set_tid_address(0x76f64878)             = 1122
set_robust_list(0x76f64880, 12)         = 0
mprotect(0x76f20000, 8192, PROT_READ)   = 0
mprotect(0x20000, 4096, PROT_READ)      = 0
mprotect(0x76f66000, 4096, PROT_READ)   = 0
ugetrlimit(RLIMIT_STACK, {rlim_cur=8192*1024, rlim_max=RLIM_INFINITY}) = 0
munmap(0x76f60000, 6905)                = 0
syscall_397(0x1, 0x76f0a6e8, 0x1800, 0x7ff, 0x7e9bea58, 0x7e9beb80) = 0
getrandom("\221m9\347", 4, GRND_NONBLOCK) = 4
brk(0)                                  = 0xac000
brk(0xcd000)                            = 0xcd000
clock_nanosleep(CLOCK_REALTIME, 0, {1, 0}, {0, 135180}) = 0
clock_nanosleep(CLOCK_REALTIME, 0, {1, 0}, {0, 135180}) = 0
clock_nanosleep(CLOCK_REALTIME, 0, {1, 0}, {0, 135180}) = 0
clock_nanosleep(CLOCK_REALTIME, 0, {1, 0}, {0, 135180}) = 0
```

</details>
    
<details>
  <summary> Code trace </summary>

```
+------------+                                                                                              
| sys_execve |                                                                                              
+--|---------+                                                                                              
   |    +-----------+                                                                                       
   +--> | do_execve |                                                                                       
        +--|--------+                                                                                       
           |    +--------------------+                                                                      
           +--> | do_execveat_common |                                                                      
                +----|---------------+                                                                      
                     |    +------------+                                                                    
                     |--> | alloc_bprm |                                                                    
                     |    +--|---------+                                                                    
                     |       |                                                                              
                     |       |--> allocate 'bprm'                                                           
                     |       |                                                                              
                     |       |--> set filename and interp of bprm                                           
                     |       |                                                                              
                     |       |    +--------------+                                                          
                     |       +--> | bprm_mm_init |                                                          
                     |            +---|----------+                                                          
                     |                |    +----------+                                                     
                     |                |--> | mm_alloc | prepare mm and page table                           
                     |                |    +----------+                                                     
                     |                |    +----------------+                                               
                     |                +--> | __bprm_mm_init | prepare vma of stack, and link it to framework
                     |                     +----------------+                                               
                     |    +--------------+                                                                  
                     |--> | copy_strings | copy env to somewhere                                            
                     |    +--------------+                                                                  
                     |    +--------------+                                                                  
                     |--> | copy_strings | copy arg to somewhere                                            
                     |    +--------------+                                                                  
                     |    +-------------+                                                                   
                     +--> | bprm_execve | move to a proper rq, clear old mapping, map exec and interp       
                          +-------------+                                                                   
```

```
+-------------+                                                                                                                    
| bprm_execve | move to a proper rq, clear old mapping, map exec and interp                                                        
+---|---------+                                                                                                                    
    |    +----------------+                                                                                                        
    |--> | do_open_execat |                                                                                                        
    |    +---|------------+                                                                                                        
    |        |                                                                                                                     
    |        |--> determine flags                                                                                                  
    |        |                                                                                                                     
    |        |    +--------------+                                                                                                 
    |        +--> | do_filp_open | allocate 'file', find dentry/inode, install ops, and call ->open()                              
    |             +--------------+                                                                                                 
    |    +------------+                                                                                                            
    |--> | sched_exec | select rq and migrate task onto it                                                                         
    |    +------------+                                                                                                            
    |    +-------------+                                                                                                           
    +--> | exec_binprm |                                                                                                           
         +---|---------+                                                                                                           
             |                                                                                                                     
             |--> for 1 exec and at most 5 interpreters                                                                            
             |                                                                                                                     
             |        +-----------------------+                                                                                    
             |------> | search_binary_handler |                                                                                    
             |        +-----|-----------------+                                                                                    
             |              |    +----------------+                                                                                
             |              |--> | prepare_binprm | read file into buffer                                                          
             |              |    +----------------+                                                                                
             |              |                                                                                                      
             |              |--> for each registered format (e.g., script, elf)                                                    
             |              |                                                                                                      
             |              |------> call ->load_binary(), e.g.,                                                                   
             |              |        +-----------------+                                                                           
             |              |        | load_elf_binary | map segments of exec and interp, copy and env to stack, regs->pc to interp
             |              |        +-----------------+                                                                           
             |              |                                                                                                      
             |              +------> return if ok                                                                                  
             |                                                                                                                     
             +------> break if brpm->interpreter isn't set (though elf has interpreter, it doesn't set this field)                 
```

```
+-----------------+                                                                                      
| load_elf_binary | map segments of exec and interp, copy and env to stack, regs->pc to interp           
+----|------------+                                                                                      
     |                                                                                                   
     |--> check if magic "\177ELF" matches                                                               
     |                                                                                                   
     |    +----------------+                                                                             
     |--> | load_elf_phdrs | load program headers                                                        
     |    +----------------+                -                                                            
     |                                                                                                   
     |--> allocate 'file' for interpreter and read in its elf header                                     
     |                                                                                                   
     |--> if interpreter file exists                                                                     
     |                                                                                                   
     |        +----------------+                                                                         
     |------> | load_elf_phdrs | load program headers of linker                                          
     |        +----------------+                                                                         
     |    +----------------+                                                                             
     |--> | begin_new_exec | ensure no other threads, clone structures, set task name, reset sig handlers
     |    +----------------+                                                                             
     |    +----------------+                                                                             
     |--> | setup_new_exec |                                                                             
     |    +---|------------+                                                                             
     |        |    +-----------------------+                                                             
     |        +--> | arch_pick_mmap_layout | set mmap base and set ->get_unmapped_area()                 
     |             +-----------------------+                                                             
     |    +-----------------+                                                                            
     |--> | setup_arg_pages | finalize the stack vma                                                     
     |    +-----------------+                                                                            
     |                                                                                                   
     |--> for each program header with 'load' type (e.g., text & data)                                   
     |                                                                                                   
     |        +---------+                                                                                
     |------> | elf_map | map the segment to user space                                                  
     |        +---------+                                                                                
     |    +---------+                                                                                    
     |--> | set_brk | heap?                                                                              
     |    +---------+                                                                                    
     |                                                                                                   
     |--> if interpreter file exists                                                                     
     |                                                                                                   
     |        +-----------------+                                                                        
     |------> | load_elf_interp | map interpreter segments                                               
     |        +-----------------+                                                                        
     |    +-----------------------------+                                                                          
     |--> | ARCH_SETUP_ADDITIONAL_PAGES | install three vmas for 'sigpage', 'vvar', and 'vdso'
     |    +-----------------------------+  
     |    +-------------------+                                                                          
     |--> | create_elf_tables | copy elf info, arg, env to user space stack                              
     |    +-------------------+                                                                          
     |    +--------------+                                                                               
     +--> | START_THREAD |  set pc and sp of 'regs'                                                      
          +--------------+                                                                               
```

```
+-----------------------------+                                                     
| arch_setup_additional_pages | install three vmas for 'sigpage', 'vvar', and 'vdso'
+-------|---------------------+                                                     
        |                                                                           
        |--> ensure 'signal_page' is allocated                                      
        |                                                                           
        |--> count needed page size for vaddr (1 for sigpage, 2 for vdso)           
        |                                                                           
        |    +-------------------+                                                  
        |--> | get_unmapped_area | find an used vaddr                               
        |    +-------------------+                                                  
        |    +--------------------------+                                           
        |--> | _install_special_mapping | install vma for that vaddr                
        |    +--------------------------+                                           
        |    +------------------+                                                   
        +--> | arm_install_vdso |                                                   
             +----|-------------+                                                   
                  |    +--------------+                                             
                  |--> | install_vvar | install vma for 'vvar'                      
                  |    +--------------+                                             
                  |    +--------------------------+                                 
                  +--> | _install_special_mapping | install vma for 'vdso'          
                       +--------------------------+                                 
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

```
+----------------+                                                                             
| begin_new_exec | ensure no other threads, clone structures, set task name, reset sig handlers
+---|------------+                                                                             
    |    +-----------+                                                                         
    +--> | de_thread | ensure no other threads in group, and arg task assumes the leader       
    |    +-----------+                                                                         
    |    +---------------+                                                                     
    |--> | unshare_files | ensure the fdtable isn't shared                                     
    |    +---------------+                                                                     
    |    +-----------------+                                                                   
    |--> | set_mm_exe_file | update mm->exe_file to the new one (not interpreter)              
    |    +-----------------+                                                                   
    |    +-----------+                                                                         
    |--> | exec_mmap | replace current task's mm with bprm's (only one vma: stack)             
    |    +-----------+                                                                         
    |    +-----------------+                                                                   
    |--> | unshare_sighand | clone sighand for arg task                                        
    |    +-----------------+                                                                   
    |    +------------------+                                                                  
    |--> | do_close_on_exec | close files                                                      
    |    +------------------+                                                                  
    |    +-----------------+                                                                   
    |--> | __set_task_comm | set task name                                                     
    |    +-----------------+                                                                   
    |    +-----------------------+                                                             
    +--> | flush_signal_handlers | clear signal handlers                                       
         +-----------------------+                                                             
```

```
+-----------------+                                                            
| load_elf_interp | map interpreter segments                                   
+----|------------+                                                            
     |    +--------------------+                                               
     |--> | total_mapping_size | sum up the size of segments with 'load' type  
     |    +--------------------+                                               
     |                                                                         
     |--> for each segment with 'load' type                                    
     |                                                                         
     |        +---------+                                                      
     +------> | elf_map | map segment to user space                            
     |        +---------+                                                      
     |    +---------+                                                          
     |--> | padzero | memset 0 on bss                                          
     |    +---------+                                                          
     |    +--------------+                                                     
     +--> | vm_brk_flags | change vm flags of bss (this might create a new vma)
          +--------------+                                                     
```
   
</details>

<details>
  <summary> Code trace2 </summary>

```
+------------------------+                                                       
| generic_file_read_iter | read data from page of mapping
+-----|------------------+                                                       
      |                                                                          
      |--> if 'DIRECT' flag is set                                               
      |                                                                          
      |------> (skip, not my case here)                                          
      |                                                                          
      |    +--------------+                                                      
      +--> | filemap_read |                                                      
           +---|----------+                                                      
               |                                                                 
               |--> while we haven't read enough                                 
               |                                                                 
               |        +-------------------+                                    
               |------> | filemap_get_pages | read data from page to pvec        
               |        +-------------------+ (refer to the below)               
               |                                                                 
               |------> for each page in pvec                                    
               |                                                                 
               |            +-------------------+                                
               +----------> | copy_page_to_iter | copy data from page to iterator
                            +-------------------+                                
```

```
+-------------------+                                                                           
| filemap_get_pages | ensure data is there and up-to-date                                       
+----|--------------+                                                                           
     |    +------------------------+                                                            
     |--> | filemap_get_read_batch | save a bunch of page addresses in pvec                     
     |    +------------------------+                                                            
     |                                                                                          
     |--> if get nothing                                                                        
     |                                                                                          
     |        +---------------------------+                                                     
     |------> | page_cache_sync_readahead |                                                     
     |        +---------------------------+                                                     
     |        +------------------------+                                                        
     |------> | filemap_get_read_batch | save a bunch of page addresses in pvec                 
     |        +------------------------+                                                        
     |                                                                                          
     |--> if still get nothing                                                                  
     |                                                                                          
     |        +---------------------+                                                           
     |------> | filemap_create_page | allocate a page, read data to it, and add the page to cache
     |        +---------------------+                                                           
     |                                                                                          
     |------> return                                                                            
     |                                                                                          
     |                                                                                          
     |--> get the last page in pvec                                                             
     |                                                                                          
     |--> if it's labeled READAHEAD                                                             
     |                                                                                          
     |        +-------------------+                                                             
     |------> | filemap_readahead |                                                             
     |        +----|--------------+                                                             
     |             |    +----------------------------+                                          
     |             +--> | page_cache_async_readahead |                                          
     |                  +----------------------------+                                          
     |                                                                                          
     +--> ensure the pages are up-to-date                                                       
```

```
+---------------------------+                                                                                               
| page_cache_sync_readahead |                                                                                               
+------|--------------------+                                                                                               
       |                                                                                                                    
       +--> prepare readahead parameters                                                                                    
       |                                                                                                                    
       |    +--------------------+                                                                                          
       +--> | page_cache_sync_ra |                                                                                          
            +----|---------------+                                                                                          
                 |    +--------------------+                                                                                
                 +--> | ondemand_readahead |                                                                                
                      +----|---------------+                                                                                
                           |                                                                                                
                           |--> adjust readahead parameters                                                                 
                           |                                                                                                
                           |    +------------------+                                                                        
                           +--> | do_page_cache_ra |                                                                        
                                +----|-------------+                                                                        
                                     |    +-------------------------+                                                       
                                     +--> | page_cache_ra_unbounded |                                                       
                                          +------|------------------+                                                       
                                                 |                                                                          
                                                 |--> for each index in the mapping range                                   
                                                 |                                                                          
                                                 |------> if page exist already                         -+                  
                                                 |                                                       |                  
                                                 |            +------------+                             |                  
                                                 |----------> | read_pages | (traced separately)         |                  
                                                 |            +------------+                             |                  
                                                 |                                                       |                  
                                                 |------> else                                           |  simplified
                                                 |                                                       |  logic
                                                 |            +--------------------+                     |                  
                                                 |----------> | __page_cache_alloc |                     |                  
                                                 |            +--------------------+                     |                  
                                                 |            +----------+                               |                  
                                                 |----------> | list_add | add the page to a local pool  |                  
                                                 |            +----------+                              -+                  
                                                 |    +------------+                                                        
                                                 +--> | read_pages | (traced separately)                                    
                                                      +------------+                                                        
```

```
+------------+                                                
| read_pages |                                                
+--|---------+                                                
   |    +----------------+                                    
   |--> | blk_start_plug | prepare a plug for the current task
   |    +----------------+                                    
   |                                                          
   |--> if ->readahead() exists                               
   |                                                          
   |------> call ->readahead(), e.g.,                         
   |        +------------------+                              
   |        | blkdev_readahead | (refer to the below)         
   |        +------------------+                              
   |                                                          
   |--> else if ->readpages() exists                          
   |                                                          
   |------> call ->readpages(), e.g.,                         
   |        +---------------+                                 
   |        | nfs_readpages |                                 
   |        +---------------+                                 
   |                                                          
   |--> else                                                  
   |                                                          
   |------> loop                                              
   |                                                          
   |----------> call ->readpage(), e.g.,                      
   |            +-----------------+                           
   |            | blkdev_readpage | (refer to the below)      
   |            +-----------------+                           
   |    +-----------------+                                   
   +--> | blk_finish_plug | (refer to the below)              
        +-----------------+                                   
```

```
+---------------------+                                                         
| filemap_create_page |                                                         
+-----|---------------+                                                         
      |    +------------------+                                                 
      |--> | page_cache_alloc | get a free page                                 
      |    +------------------+                                                 
      |    +-----------------------+                                            
      |--> | add_to_page_cache_lru | add the page to mapping and percpu lru list
      |    +-----------------------+                                            
      |    +-------------------+                                                
      |--> | filemap_read_page |                                                
      |    +----|--------------+                                                
      |         |                                                               
      |         +--> call ->readpage(), e.g.,                                   
      |              +-----------------+                                        
      |              | blkdev_readpage |                                        
      |              +-----------------+                                        
      |    +-------------+                                                      
      +--> | pagevec_add | add the page to pvec                                 
           +-------------+                                                      
```

```
+---------------+                                                                                                                   
| sync_blockdev | sync bdev mapping to back storage                                                                                 
+---|-----------+                                                                                                                   
    |    +-----------------+                                                                                                        
    +--> | __sync_blockdev |                                                                                                        
         +----|------------+                                                                                                        
              |                                                                                                                     
              |--> if caller doesn't wait                                                                                           
              |                                                                                                                     
              |        +---------------+                                                                                            
              |------> | filemap_flush |                                                                                            
              |        +---------------+                                                                                            
              |                                                                                                                     
              |--> else                                                                                                             
              |                                                                                                                     
              |        +------------------------+                                                                                   
              +------> | filemap_write_and_wait |                                                                                   
                       +-----|------------------+                                                                                   
                             |    +------------------------------+                                                                  
                             +--> | filemap_write_and_wait_range |                                                                  
                                  +-------|----------------------+                                                                  
                                          |    +----------------------------+                                                       
                                          |--> | __filemap_fdatawrite_range |                                                       
                                          |    +------|---------------------+                                                       
                                          |           |    +------------------------+                                               
                                          |           +--> | filemap_fdatawrite_wbc |                                               
                                          |                +-----|------------------+                                               
                                          |                      |    +---------------+                                             
                                          |                      +--> | do_writepages | with specified range, write dirty pages back
                                          |                           +---------------+                                             
                                          |    +-------------------------+                                                          
                                          +--> | filemap_fdatawait_range | wait till WRITEBACK pages finish                         
                                               +-------------------------+                                                          
```

```
+---------------+                                                                                                         
| do_writepages | with specified range, write dirty pages back                                                            
+---|-----------+                                                                                                         
    |                                                                                                                     
    |--> if ->writepages() exists                                                                                         
    |                                                                                                                     
    +------> call ->writepages(), e.g.,                                                                                   
    |        +-------------------+      +--------------------+                                                            
    |        | blkdev_writepages | ---> | generic_writepages |                                                            
    |        +-------------------+      +--------------------+                                                            
    |                                                                                                                     
    |--> else                                                                                                             
    |                                                                                                                     
    |        +--------------------+                                                                                       
    +------> | generic_writepages |                                                                                       
             +----|---------------+                                                                                       
                  |    +----------------+                                                                                 
                  |--> | blk_start_plug | prepare plug for current task                                                   
                  |    +----------------+                                                                                 
                  |    +-------------------+                                                                              
                  |--> | write_cache_pages | within range, call mapping->writepage() for each dirty page                  
                  |    +-------------------+                                                                              
                  |    +-----------------+                                                                                
                  +--> | blk_finish_plug | for each entity in list, add to a queue (e.g., io scheduler queue or mtd queue)
                       +-----------------+                                                                                
```

```
+---------------------------+                                                      
| __generic_file_write_iter | copy data to pages of mapping, mark them dirty       
+------|--------------------+                                                      
       |                                                                           
       |--> if 'DIRECT' flag is set                                                
       |                                                                           
       |------> (skip, not my case here)                                           
       |                                                                           
       |--> else                                                                   
       |                                                                           
       |        +-----------------------+                                          
       +------> | generic_perform_write |                                          
                +-----|-----------------+                                          
                      |                                                            
                      |--> while we haven't written enough                         
                      |                                                            
                      |------> call ->write_begin(), e.g.,                         
                      |        +--------------------+                              
                      |        | blkdev_write_begin | get a page from mapping      
                      |        +--------------------+                              
                      |        +----------------------------+                      
                      |------> | copy_page_from_iter_atomic | copy data to the page
                      |        +----------------------------+                      
                      |                                                            
                      +------> call ->write_end(), e.g.,                           
                               +------------------+                                
                               | blkdev_write_end | mark buffer dirty              
                               +------------------+                                
```

```
+---------------------+                                          
| unmap_mapping_range |                                          
+-----|---------------+                                          
      |                                                          
      |--> ensure the start and len is page-size aligned         
      |                                                          
      |    +---------------------+                               
      +--> | unmap_mapping_pages | unmap all the vma within range
           +---------------------+                               
```

```
+---------------+                                                                        
| release_pages | drop ref for each page in array, free the pages whose ref reaches 0
+---|-----------+                                                                        
    |                                                                                    
    |--> for each page in array                                                          
    |                                                                                    
    |------> page ref--                                                                  
    |                                                                                    
    |------> if ref != 0 (it's still used), continue                                     
    |                                                                                    
    |------> if page is set 'lru'                                                        
    |                                                                                    
    |            +------------------------+                                              
    |----------> | del_page_from_lru_list | remove page from list (lruvec of memory node)
    |            +------------------------+                                              
    |            +------------------------+                                              
    |----------> | __clear_page_lru_flags | clear 'lru', 'active', 'unevictable' on page 
    |            +------------------------+                                              
    |                                                                                    
    |------> add page to a local list                                                    
    |                                                                                    
    |    +----------------------+                                                        
    +--> | free_unref_page_list | free each page in the local list                       
         +----------------------+                                                        
```

```
+-------------------+                                                                                           
| __pagevec_lru_add | move pages in pvec to lru list of memory node and release them                            
+----|--------------+                                                                                           
     |                                                                                                          
     |--> for each page in pvec                                                                                 
     |        +----------------------------+                                                                    
     |        | relock_page_lruvec_irqsave | lock and get lruvec from memory node                               
     |        +----------------------------+                                                                    
     |        +----------------------+                                                                          
     |        | __pagevec_lru_add_fn |                                                                          
     |        +-----|----------------+                                                                          
     |              |                                                                                           
     |              |--> label 'lru' on page                                                                    
     |              |                                                                                           
     |              |    +----------------------+                                                               
     |              +--> | add_page_to_lru_list | add page to the proper list in lruvec based on page attributes
     |                   +----------------------+                                                               
     |    +-------------------------------+                                                                     
     |--> | unlock_page_lruvec_irqrestore | unlock                                                              
     |    +-------------------------------+                                                                     
     |    +---------------+                                                                                     
     +--> | release_pages | drop ref for each page in array, free the pages whose ref reaches 0                 
          +---------------+                                                                                     
```

```
+---------------+                                                                                         
| lru_add_drain | collect pages from other lru lists to memory node, and release them                     
+---|-----------+                                                                                         
    |                                                                                                     
    |--> get pvec from 'lru_pvecs.lru_add'                                                                
    |                                                                                                     
    |    +-------------------+                                                                            
    |--> | __pagevec_lru_add | move pages in pvec to lru list of memory node and release them             
    |    +-------------------+                                                                            
    |                                                                                                     
    |--> get pvec from 'lru_rotate.pvec'                                                                  
    |                                                                                                     
    |    +---------------------+                                                                          
    |--> | pagevec_lru_move_fn | move pages in pvec to lru list of memory node and release them           
    |    +---------------------+                                                                          
    |                                                                                                     
    |--> get pvec from 'lru_pvecs.lru_deactivate_file'                                                    
    |                                                                                                     
    |    +---------------------+                                                                          
    +--> | pagevec_lru_move_fn | move pages in pvec to lru list of memory node and release them           
    |    +---------------------+                                                                          
    |                                                                                                     
    |--> get pvec from 'lru_pvecs.lru_deactivate'                                                         
    |                                                                                                     
    |    +---------------------+                                                                          
    +--> | pagevec_lru_move_fn | move pages in pvec to lru list of memory node and release them           
    |    +---------------------+                                                                          
    |                                                                                                     
    |--> get pvec from 'lru_pvecs.lru_lazyfree'                                                           
    |                                                                                                     
    |    +---------------------+                                                                          
    |--> | pagevec_lru_move_fn | move pages in pvec to lru list of memory node and release them           
    |    +---------------------+                                                                          
    |    +---------------------+                                                                          
    +--> | activate_page_drain |                                                                          
         +-----|---------------+                                                                          
               |                                                                                          
               |--> get pvec from 'lru_pvecs.activate_page'                                               
               |                                                                                          
               |    +---------------------+                                                               
               +--> | pagevec_lru_move_fn | move pages in pvec to lru list of memory node and release them
                    +---------------------+                                                               
```

```
+--------------------------+                                                                                                
| unmap_mapping_range_tree | for each vma within range, clear page table entries within vma                                 
+------|-------------------+                                                                                                
       |    +---------------------------+                                                                                   
       |--> | vma_interval_tree_foreach |                                                                                   
       |    +---------------------------+                                                                                   
       |                                                                                                                    
       |------> adjust the start and end to ensure they are in the vma ragne                                                
       |                                                                                                                    
       |        +-------------------------+                                                                                 
       +------> | unmap_mapping_range_vma |                                                                                 
                +------|------------------+                                                                                 
                       |    +-----------------------+                                                                       
                       +--> | zap_page_range_single |                                                                       
                            +-----|-----------------+                                                                       
                                  |    +---------------+                                                                    
                                  +--> | lru_add_drain | collect pages from other lru lists to memory node, and release them
                                  |    +---------------+                                                                    
                                  |    +------------------+                                                                 
                                  +--> | unmap_single_vma | clear page table entries within range                           
                                       +------------------+                                                                 
```

```
+---------------------+                                                                           
| try_to_release_page | detatch bh list from page and free them                                   
+-----|---------------+                                                                           
      |                                                                                           
      |--> if page has the 'writeback' label, return                                              
      |                                                                                           
      |--> if mapping has ->releasepage()                                                         
      |                                                                                           
      |------> call ->releasepage()                                                               
      |                                                                                           
      |--> else                                                                                   
      |                                                                                           
      |        +---------------------+                                                            
      +------> | try_to_free_buffers |                                                            
               +-----|---------------+                                                            
                     |    +--------------+                                                        
                     |--> | drop_buffers | move the bh list to a local head, and detatch from page
                     |    +--------------+                                                        
                     |    +-------------------+                                                   
                     |--> | cancel_dirty_page | clear 'dirty' bit of page                         
                     |    +-------------------+                                                   
                     |                                                                            
                     |--> for each bh in local list                                               
                     |                                                                            
                     |        +------------------+                                                
                     +------> | free_buffer_head | return bh to kmem cache                        
                              +------------------+                                                
```

```
+----------------------+                                                      
| block_invalidatepage | clear some flags for those truncated bh              
+-----|----------------+                                                      
      |                                                                       
      |--> get buffer head from page private                                  
      |                                                                       
      |--> for each buffer in the linked list                                 
      |                                                                       
      |------> if the buffer is above the arg 'offset'                        
      |                                                                       
      |            +----------------+                                         
      |----------> | discard_buffer | clear some flags of the bh              
      |            +----------------+                                         
      |                                                                       
      |--> if arg 'length' is page size                                       
      |                                                                       
      |        +---------------------+                                        
      +------> | try_to_release_page | detatch bh list from page and free them
               +---------------------+                                        
```

```
+---------------------+                                                                                           
| truncate_inode_page | unmap and invalidate page, detatch it from mapping, and has its ref--                     
+-----|---------------+                                                                                           
      |    +-----------------------+                                                                              
      |--> | truncate_cleanup_page | unmpa and invalidate page                                                    
      |    +-----|-----------------+                                                                              
      |          |                                                                                                
      |          |--> if page is set 'mapped'                                                                     
      |          |                                                                                                
      |          |        +--------------------+                                                                  
      |          |------> | unmap_mapping_page | for each vma within range, clear page table entries within vma   
      |          |        +--------------------+                                                                  
      |          |                                                                                                
      |          |--> if page has private (private what?)                                                         
      |          |                                                                                                
      |          |        +-------------------+                                                                   
      |          +------> | do_invalidatepage | e.g., clear some flags for those truncated bh                     
      |                   +-------------------+                                                                   
      |    +------------------------+                                                                             
      +--> | delete_from_page_cache | detatch page from mapping and page ref--                                    
           +-----|------------------+                                                                             
                 |                                                                                                
                 |--> get page mapping                                                                            
                 |                                                                                                
                 |    +--------------------------+                                                                
                 |--> | __delete_from_page_cache | detatch page from mapping                                      
                 |    +--------------------------+                                                                
                 |    +----------------------+                                                                    
                 +--> | page_cache_free_page |                                                                    
                      +-----|----------------+                                                                    
                            |                                                                                     
                            |--> if mapping has ->freepage()                                                      
                            |                                                                                     
                            |------> call ->freepage()                                                            
                            |                                                                                     
                            |    +----------+                                                                     
                            +--> | put_page |                                                                     
                                 +----------+                                                                     
```

```
+-------------------+                                                                                                                  
| get_unmapped_area | lookup a region that satisfies the range ('addr' isn't guaranteed)                                               
+----|--------------+                                                                                                                  
     |                                                                                                                                 
     |--> determine get_area(): 1. from mm, 2. from file, 3. from shm                                                                  
     |                                                                                                                                 
     +--> call ->get_area(), e.g.,                                                                                                     
          +--------------------------------+                                                                                           
          | arch_get_unmapped_area_topdown | lookup a region that satisfies the range ('addr' isn't guaranteed)                        
          +-------|------------------------+                                                                                           
                  |                                                                                                                    
                  |--> if user has specified the 'addr'                                                                                
                  |                                                                                                                    
                  |        +----------+                                                                                                
                  |------> | find_vma | return  the first vma that meets 'addr < vm_end'                                               
                  |        +----------+                                                                                                
                  |                                                                                                                    
                  |------> if no such vma, or it doesn't intersect our target range                                                    
                  |                                                                                                                    
                  |----------> return                                                                                                  
                  |                                                                                                                    
                  |--> set up info (flag = top_down)                                                                                   
                  |                                                                                                                    
                  |    +------------------+                                                                                            
                  |--> | vm_unmapped_area | find a suitable region and return start addr                                               
                  |    +----|-------------+                                                                                            
                  |         |                                                                                                          
                  |         |--> if flag is top_down                                                                                   
                  |         |                                                                                                          
                  |         |        +-----------------------+                                                                         
                  |         |------> | unmapped_area_topdown | starting from high address, find a suitable region and return start addr
                  |         |        +-----------------------+                                                                         
                  |         |                                                                                                          
                  |         |--> else                                                                                                  
                  |         |                                                                                                          
                  |         |        +---------------+                                                                                 
                  |         +------> | unmapped_area | starting from low address, find a suitable region and return start addr         
                  |                  +---------------+                                                                                 
                  |                                                                                                                    
                  +--> if it fails, try bottom-up lookup                                                                               
```

```
+-------------+                                                                               
| mmap_region | ensure there's a vma covering this region, link that vma with framework       
+---|---------+                                                                               
    |    +------------------+                                                                 
    |--> | munmap_vma_range | unmap overlapped region                                         
    |    +------------------+                                                                 
    |    +-----------+                                                                        
    |--> | vma_merge | try to expand from existing vma                                        
    |    +-----------+                                                                        
    |                                                                                         
    |--> return if it's successful                                                            
    |                                                                                         
    |    +---------------+                                                                    
    |--> | vm_area_alloc | allocate 'vma'                                                     
    |    +---------------+                                                                    
    |                                                                                         
    |--> set up 'vma' with region info                                                        
    |                                                                                         
    |--> if it's a file mapping                                                               
    |                                                                                         
    |------> call ->mmap(), e.g.,                                                             
    |        +----------+                                                                     
    |        | ovl_mmap |                                                                     
    |        +----------+                                                                     
    |                                                                                         
    |--> else if it's a 'shared' mapping                                                      
    |                                                                                         
    |        +------------------+                                                             
    |------> | shmem_zero_setup |                                                             
    |        +------------------+                                                             
    |                                                                                         
    |--> else                                                                                 
    |                                                                                         
    |        +-------------------+                                                            
    |------> | vma_set_anonymous | set vma ops = NULL                                         
    |        +-------------------+                                                            
    |    +----------+                                                                         
    +--> | vma_link |                                                                         
         +--|-------+                                                                         
            |    +------------+                                                               
            |--> | __vma_link | insert vma into vma list and tree                             
            |    +------------+                                                               
            |    +-----------------+                                                          
            +--> | __vma_link_file | if it's a file mapping, insert vma into file mapping tree
                 +-----------------+                                                          
```

```
+-----------+                                                                                                                       
| sys_mmap2 |                                                                                                                       
+--|--------+                                                                                                                       
   |    +----------------+                                                                                                          
   +--> | sys_mmap_pgoff |                                                                                                          
        +---|------------+                                                                                                          
            |    +-----------------+                                                                                                
            +--> | ksys_mmap_pgoff |                                                                                                
                 +----|------------+                                                                                                
                      |    +---------------+                                                                                        
                      +--> | vm_mmap_pgoff |                                                                                        
                           +---|-----------+                                                                                        
                               |    +---------+                                                                                     
                               |--> | do_mmap |                                                                                     
                               |    +--|------+                                                                                     
                               |       |    +-------------------+                                                                   
                               |       |--> | get_unmapped_area | lookup a region that satisfies the range ('addr' isn't guaranteed)
                               |       |    +-------------------+                                                                   
                               |       |                                                                                            
                               |       |--> determine vm flags                                                                      
                               |       |                                                                                            
                               |       |    +-------------+                                                                         
                               |       |--> | mmap_region | ensure there's a vma covering this region, link that vma with framework 
                               |       |    +-------------+                                                                         
                               |       |                                                                                            
                               |       +--> determine populate len if the flags ask so                                              
                               |                                                                                                    
                               |--> if populate len is determined                                                                   
                               |                                                                                                    
                               |        +-------------+                                                                             
                               +------> | mm_populate | fault in the pages                                                          
                                        +-------------+                                                                             
```

```
+------------+
| sys_munmap |
+--|---------+
   |    +-------------+
   +--> | __vm_munmap |
        +---|---------+
            |    +-------------+
            +--> | __do_munmap | clear pte, free page table, and release vma
                 +---|---------+
                     |    +-----------------------+
                     |--> | find_vma_intersection | find the vma that intersects with arg range
                     |    +-----------------------+
                     |
                     |--> if the arg range doens't match the found vma
                     |
                     |        +-------------+
                     |------> | __split_vma |
                     |        +-------------+
                     |    +----------------------------+
                     |--> | detach_vmas_to_be_unmapped | detatch from rb tree
                     |    +----------------------------+
                     |    +--------------+
                     |--> | unmap_region |
                     |    +---|----------+
                     |        |    +---------------+
                     |        |--> | lru_add_drain | collect pages from other lru lists to memory node, and release them
                     |        |    +---------------+
                     |        |    +------------+
                     |        +--> | unmap_vmas | clear pte within range
                     |        |    +------------+
                     |        |    +---------------+
                     |        +--> | free_pgtables | free 2nd-level page table?
                     |             +---------------+
                     |    +-----------------+
                     +--> | remove_vma_list | free vma
                          +-----------------+                                                                     
```

```
+--------------+                                                                                          
| sys_mprotect |                                                                                          
+---|----------+                                                                                          
    |    +------------------+                                                                             
    +--> | do_mprotect_pkey |                                                                             
         +----|-------------+                                                                             
              |    +----------+                                                                           
              |--> | find_vma | return  the first vma that meets 'addr < vm_end'                          
              |    +----------+                                                                           
              |                                                                                           
              |--> return if vma doesn't cover the start                                                  
              |                                                                                           
              |--> endless loop                                                                           
              |                                                                                           
              |------> if ->mprotect exists (only on x86?)                                                
              |                                                                                           
              +----------> call ->mprotect()                                                              
              |                                                                                           
              |        +----------------+                                                                 
              |------> | mprotect_fixup |                                                                 
              |        +---|------------+                                                                 
              |            |    +-----------+                                                             
              |            |--> | vma_merge | check if we can merge with prev/next vma with our new flags 
              |            |    +-----------+                                                             
              |            |                                                                              
              |            |--> if the specified range doens't match the whole vma                        
              |            |                                                                              
              |            |        +-----------+                                                         
              |            |------> | split_vma |                                                         
              |            |        +-----------+                                                         
              |            |                                                                              
              |            +--> change attributes in both sw and hw                                       
              |                                                                                           
              +--> break loop if we reach the end of range                                                
```

```
+---------+                                                                                              
| sys_brk |                                                                                              
+--|------+                                                                                              
   |                                                                                                     
   |--> if it's a shrinking request                                                                      
   |                                                                                                     
   |        +-------------+                                                                              
   |------> | __do_munmap | clear pte, free page table, and release vma                                  
   |        +-------------+                                                                              
   |                                                                                                     
   |        go to 'success'                                                                              
   |                                                                                                     
   |    +--------------+                                                                                 
   |--> | do_brk_flags |                                                                                 
   |    +---|----------+                                                                                 
   |        |    +-------------------+                                                                   
   |        |--> | get_unmapped_area | lookup a region that satisfies the range ('addr' isn't guaranteed)
   |        |    +-------------------+                                                                   
   |        |    +------------------+                                                                    
   |        |--> | munmap_vma_range | clear old record                                                   
   |        |    +------------------+                                                                    
   |        |    +-----------+                                                                           
   |        |--> | vma_merge | try to expand existing vma to cover our region                            
   |        |    +-----------+                                                                           
   |        |                                                                                            
   |        |--> return if it's successful                                                               
   |        |                                                                                            
   |        |    +---------------+                                                                       
   |        |--> | vm_area_alloc |                                                                       
   |        |    +---------------+                                                                       
   |        |                                                                                            
   |        |--> set up vma based on region info                                                         
   |        |                                                                                            
   |        |    +----------+                                                                            
   |        +--> | vma_link | link vma into framework                                                    
   |             +----------+                                                                            
   |                                                                                                     
   |--> if need to populate                                                                              
   |                                                                                                     
   |        +-------------+                                                                              
   +------> | mm_populate |                                                                              
            +-------------+                                                                              
```

```
+-----------------------+                                                    
| flush_all_zero_pkmaps | release unused pages from pkmap                    
+-----|-----------------+                                                    
      |                                                                      
      |--> for each pkmap (512)                                              
      |                                                                      
      |------> if nothing we can do, or it's in use                          
      |                                                                      
      |----------> continue                                                  
      |                                                                      
      |------> pkmap_count[i] = 0                                            
      |                                                                      
      |        +----------+                                                  
      |------> | pte_page | get struct page from pte value (physical address)
      |        +----------+                                                  
      |        +-----------+                                                 
      |------> | pte_clear | clear pte                                       
      |        +-----------+                                                 
      |                                                                      
      +------> remove page from kmap list                                    
```

## <a name="kmap"></a> Kmap

```
+------+                                                             
| kmap | ensure the page has a mapped virtual address                
+-|----+                                                             
  |                                                                  
  |--> if arg page isn't from highmem                                
  |                                                                  
  |        +--------------+                                          
  |------> | page_address | return mapped virtual address of arg page
  |        +--------------+                                          
  |                                                                  
  |--> else                                                          
  |                                                                  
  |        +-----------+                                             
  +------> | kmap_high | map a highmem page                          
           +-----------+                                             
```

```
+-----------+                                                            
| kmap_high | map a highmem page                                         
+--|--------+                                                            
   |    +--------------+                                                 
   |--> | page_address | return mapped virtual address of arg page       
   |    +--------------+                                                 
   |                                                                     
   |--> if the page doesn't have mapped virtual address yet              
   |                                                                     
   |--> endless loop                                                     
   |                                                                     
   |        +-------------------+                                        
   |------> | get_next_pkmap_nr | get next index                         
   |        +-------------------+                                        
   |                                                                     
   |------> if no more pkmaps                                            
   |                                                                     
   |            +-----------------------+                                
   |----------> | flush_all_zero_pkmaps | release unused pages from pkmap
   |            +-----------------------+                                
   |                                                                     
   |------> if pkmap_count[idx] == 0 (available)                         
   |                                                                     
   |----------> break                                                    
   |                                                                     
   |------> if can retry (at most 512 times)                             
   |                                                                     
   +----------> continue                                                 
   |                                                                     
   |------> sleep to see if anyone unmap some slots                      
   |                                                                     
   |--> prepare vaddr based on index                                     
   |                                                                     
   |    +------------+                                                   
   |--> | set_pte_at | set pte                                           
   |    +------------+                                                   
   |                                                                     
   |--> label the index as used                                          
   |                                                                     
   |    +------------------+                                             
   +--> | set_page_address | relate 'page' and 'vaddr'                   
        +------------------+                                             
```

```
+--------+                                                               
| kunmap |                                                               
+-|------+                                                               
  |                                                                      
  |--> if arg page isn't from high mem, return                           
  |                                                                      
  |    +-------------+                                                   
  +--> | kunmap_high |                                                   
       +---|---------+                                                   
           |    +--------------+                                         
           |--> | page_address | get virtual addr that the page maps from
           |    +--------------+                                         
           |    +----------+                                             
           |--> | PKMAP_NR | get index from virtual addr                 
           |    +----------+                                             
           |                                                             
           |--> pkmap_count[nr]--                                        
           |                                                             
           |--> if there's any task waiting for the unmap, wake it up    
           |                                                             
           |        +---------+                                          
           +------> | wake_up |                                          
                    +---------+                                          
```
      
</details>

## <a name="reference"></a> Reference

(TBD)



