## Index

- [Introduction](#introduction)
- [Boot Flow](#boot-flow)

## <a name="introduction"></a> Introduction

Memory management is one of the kernel's most essential and fundamental infrastructures and is used to satisfy kernel space and user space requirements.
So why does memory need to be managed?
- If a simple logic runs without using any memory, we don't need the management.
- If an application only writes data to a pre-defined area and doesn't exceed the boundary, we don't need the management.
- If the system reaches financial freedom in terms of memory and the program won't consume them entirely before exiting, we don't need the management.

Unfortunately, on a 32-bit machine, it supports up to 4 GB RAM only.
Worst of all, dynamic memory allocation is so typical. Also, instead of one process, hundreds or thousands of them run altogether.
Besides, the application's metadata is also generated dynamically in OS.

```
+----------------+                                                
|                |  <------- request of large-size memory (>= 4KB)
|                |                                                
|     memory     |                                                
|   management   |  <------- request of small-size memory (< 4KB) 
|                |                                                
|                |                                                
|                |  <------- return memory                        
+----------------+                                                
```

## <a name="boot-flow"></a> Boot Flow

```
+------------+                                                                                                                          
| setup_arch |                                                                                                                          
+--|---------+                                                                                                                          
   |    +-------------------+                                                                                                           
   +--> | setup_machine_fdt | add memblock for the manageable region                             0x8000_0000 ~ 0xA000_0000, based on DTB
   |    +-------------------+                                                                                                           
   |                                                                                                                                    
   +--> reserve memblock for DTB                                                                 0x8810_C000 ~ 0x8812_12B8              
   |                                                                                                                                    
   |    +-------------------+                                                                                                           
   +--> | arm_memblock_init |                                                                                                           
        +----|--------------+                                                                                                           
             |                                                                                                                          
             +--> reserve memblock for kernel                                                    0x8010_0000 ~ 0x80EF_7BA0              
             |                                                                                                                          
             |    +-----------------+                                                                                                   
             +--> | arm_initrd_init | reserve memblock for initramfs                             0x8800_0000 ~ 0x8810_C000              
             |    +-----------------+                                                                                                   
             |    +-------------------------+                                                                                           
             +--> | arm_mm_memblock_reserve | reserve memblock for the initial page table        0x8000_4000 ~ 0x8000_8000              
             |    +-------------------------+                                                                                           
             |    +----------------------------------+                                                                                  
             +--> | early_init_fdt_scan_reserved_mem |                                                                                  
             |    +--------|-------------------------+                                                                                  
             |             |    +-------------------------+                                                                             
             |             +--> | __fdt_scan_reserved_mem | add memblock for VGA                 0x9F00_0000 ~ 0xA000_0000, based on DTB
             |             |    +-------------------------+                                                                             
             |             |    +-------------------------+                                                                             
             |             +--> | __fdt_scan_reserved_mem | add memblock for flash               0x9800_0000 ~ 0x9C00_0000, based on DTB
             |             |    +-------------------------+                                                                             
             |             |    +-------------------------+                                                                             
             |             +--> | __fdt_scan_reserved_mem | add memblock for coldfire            0x9EF0_0000 ~ 0x0010_0000, based on DTB
             |             |    +-------------------------+                                                                             
             |             |    +-----------------------+                                                                               
             |             +--> | fdt_init_reserved_mem | 1. reserve memblock for GFX            0x9D00_0000 ~ 0x9E00_0000, based on DTB
             |                  +-----------------------+ 2. reserve memblock for video engine   0x9600_0000 ~ 0x9800_0000, based on DTB
             |    +------------------------+                                                                                            
             +--> | dma_contiguous_reserve | reserve memblock for the DMA/CMA                    0x9C00_0000 ~ 0x9D00_0000              
                  +------------------------+                                                                                                  
```

```
[    0.000000] Reserved memory: created CMA memory pool at 0x9d000000, size 16 MiB
[    0.000000] OF: reserved mem: initialized node framebuffer, compatible id shared-dma-pool
[    0.000000] Reserved memory: created CMA memory pool at 0x96000000, size 32 MiB
[    0.000000] OF: reserved mem: initialized node jpegbuffer, compatible id shared-dma-pool
[    0.000000] cma: Reserved 16 MiB at 0x9c000000
[    0.000000] Ignoring RAM at 0x9ee00000-0xa0000000
[    0.000000] Consider using a HIGHMEM enabled kernel.
[    0.000000] Zone ranges:
[    0.000000]   Normal   [mem 0x0000000080000000-0x000000009eefffff]
[    0.000000]   HighMem  empty
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000080000000-0x0000000097ffffff]
[    0.000000]   node   0: [mem 0x0000000098000000-0x000000009bffffff]
[    0.000000]   node   0: [mem 0x000000009c000000-0x000000009edfffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000080000000-0x000000009edfffff]
[    0.000000] CPU: All CPU(s) started in SVC mode.
[    0.000000] percpu: Embedded 16 pages/cpu s35148 r8192 d22196 u65536
[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 125476
[    0.000000] Kernel command line: console=ttyS4,115200 earlycon
[    0.000000] Dentry cache hash table entries: 65536 (order: 6, 262144 bytes, linear)
[    0.000000] Inode-cache hash table entries: 32768 (order: 5, 131072 bytes, linear)
[    0.000000] mem auto-init: stack:off, heap alloc:off, heap free:off
[    0.000000] Memory: 354620K/505856K available (9216K kernel code, 805K rwdata, 2192K rodata, 1024K init, 185K bss, 85700K reserved, 65536K cma-reserved, 0K highmem
)
```
