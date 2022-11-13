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
    
## <a name="reference"></a> Reference

(TBD)




