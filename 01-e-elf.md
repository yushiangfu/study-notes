## Index

- [Introduction](#introduction)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

While EXE is the executable format on Windows OS, ELF format applies to applications and other files on Linux systems.

- Object file: base compiled unit
- Executable: running program
- Shared lib: dynamic library
- Coredump: analysis of a task

Unlike the shared library, a static library works as a collection of object files, and it's not usable unless compiled into an executable.

```
+--------+         +--------+                                                                                  
| source |  ---->  | object |    ---+                             |        |                                   
|  code  |         |  file  |       |                             |        |                                   
+--------+         +--------+       |                             |        |                                   
                                    |                             |        |                                   
+--------+         +--------+       |       +------------+        +--------+                                   
| source |  ---->  | object |  --------->   | executable |        |exe text|                                   
|  code  |         |  file  |       |       |    app     |        +--------+                                   
+--------+         +--------+       |       +------------+        |exe data|                                   
                                    |                             +--------+                                   
+--------+         +--------+       |                             |        |                                   
| source |  ---->  | object |  -----+                             |        |                       +----------+
|  code  |         |  file  |                                     |        |    program crashes    | coredump |
+--------+         +--------+                                     |        |   ---------------->   |   file   |
                                                                  |        |                       +----------+
+--------+         +--------+                                     |        |                                   
| source |  ---->  | object |    ---+                             |        |                                   
|  code  |         |  file  |       |                             |        |                                   
+--------+         +--------+       |                             +--------+                                   
                                    |         +--------+          |lib text|                                   
+--------+         +--------+       |         | shared |          +--------+  e.g. libc                        
| source |  ---->  | object |  --------->     |   lib  |          |lib data|                                   
|  code  |         |  file  |       |         +--------+          +--------+                                   
+--------+         +--------+       |                             |        |                                   
                                    |                             |        |                                   
+--------+         +--------+       |                             |        |                                   
| source |  ---->  | object |  -----+                             +--------+                                   
|  code  |         |  file  |                                     |lib text|                                   
+--------+         +--------+                                     +--------+  e.g. libpthread                  
                                                                  |lib data|                                   
                                                                  +--------+                                   
                                                                  |        |                                   
                                                                  |        |                                   
```



Let's introduce the ELF format by the famous 'Hello, World!' example.

```
#include <stdio.h>
#include <unistd.h>

void main()
{   
    printf("Hello, World!\n");
    while (1)
        sleep(1); // so I can observe /proc/$PID/maps
}
```

Usually, our logic and global variables will be compiled into .text, .data, and .bss within the ELF executables. 
The ELF format consists of ELF header, programer headers, many sections, and section headers. 
Multiple sections will further group into one segment of different types, e.g., INTERP, LOAD, DYNAMIC. 
When we load an executable into memory, the kernel only loads segments of LOAD type but not all of them.

```
                                     +------------------+                                         
                                     |     elf hdr      |                                         
                                     +------------------+                                         
                                     |   program hdrs   | ---------------------------------------+
                                     +------------------+                                        |
         +-  section hdr  -------->  |   .ARM.exidx     ||     <-------- program hdr  -+         |
         |                           |                  |                              |         |
         |   section hdr  -------->  |     .interp      ||     <-------- program hdr   |         |
         |                           |                  |                              |         |
         |   section hdr  -------->  |     .dynsym      ||                             |         |
         |   section hdr  -------->  |     .dynstr      ||                             |         |
         |   section hdr  -------->  |      .text       ||     <-------- program hdr   |         |
         |   section hdr  -------->  |     .rodata      ||                             |         |
         |   section hdr  -------->  |       ...        ||                             |         |
         |                           |                  |                              |         |
+------- |   section hdr  -------->  |      .got        ||                             |         |
|        |   section hdr  -------->  |      .data       ||     <-------- program hdr   |---------+
|        |   section hdr  -------->  |      .bss        ||                             |          
|        |   section hdr  -------->  |       ...        ||                             |          
|        |                           |                  |                              |          
|        |   section hdr  -------->  |     .dynamic     ||     <-------- program hdr   |          
|        |                           |                  |                              |          
|        |   section hdr  -------->  |.note.gnu.build-id||     <-------- program hdr   |          
|        |   section hdr  -------->  |   .note.ABI-tag  ||                             |          
|        |                           |                  |                              |          
|        |   section hdr  -------->  |   .init_array    ||     <-------- program hdr  -+          
|        +-  section hdr  -------->  |       ...        ||                                        
|                                    +------------------+                                         
+----------------------------------- |   section hdrs   |                                         
                                     +------------------+                                         
```

<details>
  <summary> 'readelf' usage </summary>

ELF header

```
bobfu@bobfu-Vostro-5402:~/workspace/oblinux$ readelf -h a.out
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           ARM
  Version:                           0x1
  Entry point address:               0x1033c
  Start of program headers:          52 (bytes into file)
  Start of section headers:          6908 (bytes into file)
  Flags:                             0x5000200, Version5 EABI, soft-float ABI
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         9
  Size of section headers:           40 (bytes)
  Number of section headers:         29
  Section header string table index: 28
```

Program headers of a.out

```
bobfu@bobfu-Vostro-5402:~/workspace/oblinux$ readelf -l a.out

Elf file type is EXEC (Executable file)
Entry point 0x1033c
There are 9 program headers, starting at offset 52

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  EXIDX          0x0004cc 0x000104cc 0x000104cc 0x00008 0x00008 R   0x4
  PHDR           0x000034 0x00010034 0x00010034 0x00120 0x00120 R   0x4
  INTERP         0x000154 0x00010154 0x00010154 0x00013 0x00013 R   0x1
      [Requesting program interpreter: /lib/ld-linux.so.3]
  LOAD           0x000000 0x00010000 0x00010000 0x004d8 0x004d8 R E 0x10000
  LOAD           0x000f10 0x00020f10 0x00020f10 0x0011c 0x00120 RW  0x10000
  DYNAMIC        0x000f18 0x00020f18 0x00020f18 0x000e8 0x000e8 RW  0x4
  NOTE           0x000168 0x00010168 0x00010168 0x00044 0x00044 R   0x4
  GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RW  0x10
  GNU_RELRO      0x000f10 0x00020f10 0x00020f10 0x000f0 0x000f0 R   0x1

 Section to Segment mapping:
  Segment Sections...
   00     .ARM.exidx 
   01     
   02     .interp 
   03     .interp .note.gnu.build-id .note.ABI-tag .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rel.dyn .rel.plt .init .plt .text .fini .rodata .ARM.exidx .
eh_frame 
   04     .init_array .fini_array .dynamic .got .data .bss 
   05     .dynamic 
   06     .note.gnu.build-id .note.ABI-tag 
   07     
   08     .init_array .fini_array .dynamic 
```

Sections headers of a.out

```
bobfu@bobfu-Vostro-5402:~/workspace/oblinux$ readelf -S a.out
There are 29 section headers, starting at offset 0x1afc:

Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .interp           PROGBITS        00010154 000154 000013 00   A  0   0  1
  [ 2] .note.gnu.build-i NOTE            00010168 000168 000024 00   A  0   0  4
  [ 3] .note.ABI-tag     NOTE            0001018c 00018c 000020 00   A  0   0  4
  [ 4] .gnu.hash         GNU_HASH        000101ac 0001ac 000030 04   A  5   0  4
  [ 5] .dynsym           DYNSYM          000101dc 0001dc 000060 10   A  6   1  4
  [ 6] .dynstr           STRTAB          0001023c 00023c 000047 00   A  0   0  1
  [ 7] .gnu.version      VERSYM          00010284 000284 00000c 02   A  5   0  2
  [ 8] .gnu.version_r    VERNEED         00010290 000290 000020 00   A  6   1  4
  [ 9] .rel.dyn          REL             000102b0 0002b0 000008 08   A  5   0  4
  [10] .rel.plt          REL             000102b8 0002b8 000028 08  AI  5  21  4
  [11] .init             PROGBITS        000102e0 0002e0 00000c 00  AX  0   0  4
  [12] .plt              PROGBITS        000102ec 0002ec 000050 04  AX  0   0  4
  [13] .text             PROGBITS        0001033c 00033c 000174 00  AX  0   0  4
  [14] .fini             PROGBITS        000104b0 0004b0 000008 00  AX  0   0  4
  [15] .rodata           PROGBITS        000104b8 0004b8 000012 00   A  0   0  4
  [16] .ARM.exidx        ARM_EXIDX       000104cc 0004cc 000008 00  AL 13   0  4
  [17] .eh_frame         PROGBITS        000104d4 0004d4 000004 00   A  0   0  4
  [18] .init_array       INIT_ARRAY      00020f10 000f10 000004 04  WA  0   0  4
  [19] .fini_array       FINI_ARRAY      00020f14 000f14 000004 04  WA  0   0  4
  [20] .dynamic          DYNAMIC         00020f18 000f18 0000e8 08  WA  6   0  4
  [21] .got              PROGBITS        00021000 001000 000024 04  WA  0   0  4
  [22] .data             PROGBITS        00021024 001024 000008 00  WA  0   0  4
  [23] .bss              NOBITS          0002102c 00102c 000004 00  WA  0   0  1
  [24] .comment          PROGBITS        00000000 00102c 00002b 01  MS  0   0  1
  [25] .ARM.attributes   ARM_ATTRIBUTES  00000000 001057 00002a 00      0   0  1
  [26] .symtab           SYMTAB          00000000 001084 000670 10     27  80  4
  [27] .strtab           STRTAB          00000000 0016f4 000302 00      0   0  1
  [28] .shstrtab         STRTAB          00000000 0019f6 000105 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  y (purecode), p (processor specific)
```

Program headers of ld-linux.so.3

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

</details>

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
    
## <a name="reference"></a> Reference

(TBD)




