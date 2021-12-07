## Index

- [Terminology](#terminology)
- [Introduction](#introduction)
- [Boot Flow](#boot-flow)
- [Memblock Allocator](#memblock-allocator)
- [Virtual Space](#virtual-space)
- [Page Table](#page-table)
- [Paging Init](#paging-init)
- [Node and Zone](#node-and-zone)

## <a name="terminology"></a> Terminology

- Device Tree Source (DTS)
   - It's a structured configuration that we use to control kernel behavior.
- Device Tree Blob (DTB)
   - The binary compiled from DTS
   - The format that kernel parses from during boot flow.

## <a name="introduction"></a> Introduction

Memory management is one of the kernel's most essential and fundamental infrastructures and is used to satisfy kernel space and user space requirements.
So why does memory need to be managed?
- If a simple logic runs without memory, we don't need the management.
- If an application only writes data to a pre-defined area and doesn't exceed the boundary, we don't need the management.
- If the system reaches financial freedom in memory and the program won't consume them entirely before exiting, we don't need the management.

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

0. Set up barely enough mapping for the kernel to run normally.
1. Add and reserve memory regions based on DTS/DTB, and then 'memblock allocator' works as the temporary memory management.
      - Allocation becomes available, but no page or object concept here.
2. Clear the page table and map the free regions from the above allocator.
3. Keep those reserved areas and transfer available pages to the 'buddy system.'
      - Page(s) allocation becomes available.
4. Build 'kmem cache' to satisfy the request of a smaller-sized structure.
      - Object(s) allocation becomes available.

## <a name="memblock-allocator"></a> Memblock Allocator

Memblock allocator is the temporary memory management handling the add and reserve of memory blocks during boot time.
First, in DTS/DTB, we specify the target memory region that we want the kernel to help manage and the kernel parses that config and passes to the memblock module.
Then we reserve a few areas occupied by DTB, kernel, initramfs, and initial page table in case they got allocated for other purposes by accident.
DTS/DTB also specified many ranges that needed to be added or reserved in the memblock allocator.
Lastly, the kernel reserves an area for DMA/CMA, and then the allocator is good to go with these significant regions saved in its static data structure.

```                                               
              +-----add------+                     
              |              |                     
                                                   
   physical           +----reserve---+             
   address            |              |             
                                                   
 0x8000_0000  +--------------+                     
              |              |                     
              |       +--------------+  0x8000_4000
              |       |  page table  |             
              |       +--------------+  0x8000_8000
              |              |                     
              |       +--------------+  0x8010_0000
              |       |    kernel    |             
              |       +--------------+  0x80EF_7BA0
              |              |                     
              |       +--------------+  0x8800_0000
              |       |  initramfs   |             
              |       +--------------+  0x8810_C000
              |       |      DTB     |             
              |       +--------------+  0x8812_12B8
              |              |                     
              |       +--------------+  0x9600_0000
              |       | video engine |             
 0x9800_0000  +-------+--------------+  0x9800_0000
              |     flash    |                     
 0x9C00_0000  +-------+--------------+  0x9C00_0000
              |       |   DMA/CMA    |             
              |       +--------------+  0x9D00_0000
              |       |     GFX      |             
              |       +--------------+  0x9E00_0000
              |              |                     
 0x9EF0_0000  +--------------+                     
              |   coldfire   |                     
 0x9F00_0000  +--------------+                     
              |     VGA      |                     
 0xA000_0000  +--------------+                     
```

- Code flow

```
+------------+                                                                                                                                            
| setup_arch |                                                                                                                                            
+--|---------+                                                                                                                                            
   |    +-------------------+                                                                                                                             
   +--> | setup_machine_fdt | add memblock for the manageable region (0x8000_0000 ~ 0xA000_0000)                                                          
   |    +-------------------+                                                                                                                             
   |                                                                                                                                                      
   +--> reserve memblock for DTB (0x8810_C000 ~ 0x8812_12B8)                                                                                              
   |                                                                                                                                                      
   |    +-------------------+                                                                                                                             
   |--> | arm_memblock_init |                                                                                                                             
   |    +----|--------------+                                                                                                                             
   |         |                                                                                                                                            
   |         +--> reserve memblock for kernel (0x8010_0000 ~ 0x80EF_7BA0)                                                                                 
   |         |                                                                                                                                            
   |         |    +-----------------+                                                                                                                     
   |         +--> | arm_initrd_init | reserve memblock for initramfs (0x8800_0000 ~ 0x8810_C000)                                                          
   |         |    +-----------------+                                                                                                                     
   |         |    +-------------------------+                                                                                                             
   |         +--> | arm_mm_memblock_reserve | reserve memblock for the initial page table (0x8000_4000 ~ 0x8000_8000)                                     
   |         |    +-------------------------+                                                                                                             
   |         |    +----------------------------------+                                                                                                    
   |         +--> | early_init_fdt_scan_reserved_mem |                                                                                                    
   |         |    +--------|-------------------------+                                                                                                    
   |         |             |    +-------------------------+                                                                                               
   |         |             +--> | __fdt_scan_reserved_mem | add memblock for VGA （0x9F00_0000 ~ 0xA000_0000）                                              
   |         |             |    +-------------------------+                                                                                               
   |         |             |    +-------------------------+                                                                                               
   |         |             +--> | __fdt_scan_reserved_mem | add memblock for flash （0x9800_0000 ~ 0x9C00_0000）                                            
   |         |             |    +-------------------------+                                                                                               
   |         |             |    +-------------------------+                                                                                               
   |         |             +--> | __fdt_scan_reserved_mem | add memblock for coldfire （0x9EF0_0000 ~ 0x0010_0000）                                         
   |         |             |    +-------------------------+                                                                                               
   |         |             |    +-----------------------+                                                                                                 
   |         |             +--> | fdt_init_reserved_mem | 1. reserve memblock for GFX （0x9D00_0000 ~ 0x9E00_0000）                                         
   |         |                  +-----------------------+ 2. print log                                                                                    
   |         |                                                [    0.000000] Reserved memory: created CMA memory pool at 0x9d000000, size 16 MiB          
   |         |                                                [    0.000000] OF: reserved mem: initialized node framebuffer, compatible id shared-dma-pool
   |         |                                            3. reserve memblock for video engine （0x9600_0000 ~ 0x9800_0000）                                
   |         |                                            4. print log                                                                                    
   |         |                                                [    0.000000] Reserved memory: created CMA memory pool at 0x96000000, size 32 MiB          
   |         |                                                [    0.000000] OF: reserved mem: initialized node jpegbuffer, compatible id shared-dma-pool 
   |         |    +------------------------+                                                                                                              
   |         +--> | dma_contiguous_reserve | 1. reserve memblock for the DMA/CMA （0x9C00_0000 ~ 0x9D00_0000）                                              
   |              +------------------------+ 2. print log                                                                                                 
   |                                             [    0.000000] cma: Reserved 16 MiB at 0x9c000000                                                        
   |    +-------------+                                                                                                                                   
   +--> | paging init | we'll introduce this function later                                                                                               
        +-------------+       
```

## <a name="virtual-space"></a> Virtual Space

So far, the memory blocks are all about physical addresses, but actually, we are running in virtual space already.
The logic written in assembly code before entering 'start_kerner' sets up simple mapping for the kernel itself and enables MMU.

Why do we need virtual space?
All the physical addresses we've mentioned indicate their position on the bus, and it's totally up to hardware designers' preference to decide the layout.
For example, on AST2500, physical address 0x0000_0000 maps to SPI flash and 0x8000_0000 maps to DRAM.
Since the position of DRAM differs from machine to machine, even with the same processor ARM, chip designers can still place DRAM at different bus addresses.
Introducing virtual space can solve this problem by consistently presenting the same layout to running kernel and applications.
Different DRAM positions affect how the kernel maps its virtual address to the destination.
The typical ratio between user and kernel space on ARM system is 3:1 or 2:2, and our study case uses the latter one.

Involved hardware components:
- Memory Management Unit (MMU)
   - Once enabled, the address CPU handles become virtual, and it takes translation to get the physical address.
- Table Look-aside Buffer (TLB)
   - A hardware cache that helps speed up the translation of MMU.
- Translation Table Base Register (TTBR)
   - It tells MMU the physical address of the page table. Whenever a context switch happens, it changes accordingly as well.

Here's how it works:
1. CPU tries to access the virtual address.
2. MMU does the real translation work.
3. Check with TLB first since it's the cache. If it hits, the job finishes.
4. If not hit, check with the page table pointed by TTBR.
5. And update TLB as well.

Much knowledge and expertise lie here, such as L1 cache, L2 cache, i-cache, d-cache, synchronization, VIPT, etc...
But I have no plan to delve into any of them at all.

```
                            physical space                           virtual space                              
                                                                                                                
                          +----------------+                       +----------------+                           
                          |                |                       |                |                           
                          |                |                       |                |                           
                          |                |                       |                |                           
                          |                |                       |                |                           
                          |                |                       |                |                           
                          |                |                       |                |                           
                          |                |          2            |----------------| 0x7F00_0000, MODULES_VADDR
 PHYS_OFFSET, 0x8000_0000 |----------------|       +-----+    1    |----------------| 0x8000_0000, PAGE_OFFSET  
                          |      DRAM      | <---- | MMU | <------ |                |                           
                          |                |       +-----+         |----------------| 0x9F00_0000, VMALLOC_START
              0xA000_0000 |----------------|      3 |   | 4        |                |                           
                          |                |        v   v          |                |                           
                          |                |  +-----+   +-------+  |                |                           
                          |                |  | TLB |<--| page  |  |                |                           
                          |                |  +-----+ 5 | table |  |                |                           
                          +----------------+            +-------+  +----------------+                           
```

## <a name="page-table"></a> Page Table

The minimum mapped area is a 4K-sized page frame.
However, a page table isn't a structure that prepares an entry for each page frame but a multiple-level hierarchy that can support up to 5 levels.
They are shown as the PGD, P4D, PUD, PMD, PTE from top to bottom in the source code.
Thankfully, AST2500@ARM utilizes only PGD, PMD, and PTE, and here's a brief introduction.

- PGD
   - It represents a 2M mapping.
   - One PGD consists of two PMDs in place.
- PMD
   - It represents a 1M mapping or points to the next level.
   - a.k.a. section
- PTE
   - Represent a 4K mapping.

Let's ignore PGD for simplicity and focus on a few possible combinations of PMD and PTE.

- Case 1: PMD is zero (unmapped).
- Case 2: PMD has a value (attribute determines the interpretation).
   - Case 2.1: it's a physical address for 1M mapping.
   - Case 2.2: it's a physical address pointing to the 2nd level page table.
      - Case 2.2.1: PTE is zero (unmapped).
      - Case 2.2.2: it's a physical address for 4K mapping.

As a result, we can save lots of 2nd level tables if they are section mapped or not mapped at all.
Please note that every 2nd level table is for two PMDs.

```
        1st level                2nd level
        PGD table                PTE table

        +-------+               ------------  --+
        |   PMD0|                 |PTE0  |      |
   PGD0 |  ------                 +------+      |
        |   PMD1|                    -          |
        +-------+                    -          |
        |   PMD0| ---+            +------+      |
   PGD1 |  ------    |            |PTE255|      |
        |   PMD1| -+ |          ------------    | for SW (Linux) to check attributes
        +-------+  | |            |PTE0  |      |
            -      | |            +------+      |
            -      | |               -          |
            -      | |               -          |
            -      | |            +------+      |
        +-------+  | |            |PTE255|      |
        |   PMD0|  | |          ------------  --+
PGD2047 |  ------  | +--------->  |PTE0  |      |
        |   PMD1|  |              +------+      |
        +-------+  |                 -          |
                   |                 -          |
                   |              +------+      |
                   |              |PTE255|      |
                   |            ------------    | for HW (MMU) to look up
                   +----------->  |PTE0  |      |
                                  +------+      |
                                     -          |
                                     -          |
                                  +------+      |
                                  |PTE255|      |
                                ------------  --+    
```

Given a virtual address, we can do the 'page walk' by following the below steps:
1. Start from the 1st-level page table assigned to the task.
2. Use bit 20~31 as the index to find the entry.
   - The value means physical address if it's a section mapping.
   - Otherwise, it points to the 2nd-level table. (Assume it's our case)
3. Use bit 12~19 as the index to find the entry.
4. Add the offset to the found physical address, and that's it!

```                                   
               bit 31       20 19   12 11        0 
                  +-----------+-------+-----------+
 virtual address  |  12 bits  |8 bits |  12 bits  |
                  +-----------+-------+-----------+
                    1st index             offset   
                              2nd index            
```

## <a name="paging-init"></a> Paging Init

Knowing the necessity of virtual space and the hierarchy of page tables, we progress to introduce the flow of paging init.
This logic prepares several mappings from managed memory to vmalloc, pkmap, fixmap, etc.
Here's the list of them sorted in the order of ascending virtual address:

- Module
   - It's an area reserved for loaded modules.
- Pkmap
   - I don't know.
- Kernel
   - it further separates into executable and non-executable parts.
- Manageable memory
   - Exclude kernel since it has its mapping.
   - Exclude 'flash,' coldfire,' and 'VGA' because of their 'no map' attribute specified in DTS.
- DMA regions
   - 'video engine', 'DMA/CMA', and 'GFX'
   - Though they are in manageable memory range, they get remapped with the DMA attribute.
- Vmalloc area
   - The memory from this region is guaranteed to be virtually consecutive.
- DTB
   - 2M-sized mapping for DTB
- Fixmap
   - I don't know.
- Vector
   - A table for exception handling

```
                                                                                              virtual                  
                                                                                              address                  
                                                                                                                       
                                                                          +--------------+  0x7F00_0000, MOUDLES_VADDR 
    physical                                                              |    module    |                             
    address                                                               +--------------+  0x7FE0_0000                
                                                                          |     pkmap    |                             
  0x8000_0000  +------+--------------+   --+                +--           +--------------+  0x8000_0000, PAGE_OFFSET   
               |      |  page table  |     |  map_kernel()  |             |   kernel-x   |                             
  0x8000_4000  |      |       +      |     | <------------> |             +--------------+  0x80E0_0000                
               |      |    kernel    |     |                |             |   kernel-nx  |                             
  0x80F0_0000  |      +--------------+   --+                +--           +--------------+  0x80F0_0000                
               |              |            |                |             |              |                             
  0x8800_0000  |      +--------------+     |                |             |              |                             
               |      |  initramfs   |     |                |             |              |                             
  0x8810_C000  |      +--------------+     |                |             |              |                             
               |      |      DTB     |     |  map_lowmem()  |             |              |                             
  0x8812_12B8  |      +--------------+     | <------------> |             |              |                             
               |              |            |                |             |              |                             
  0x9600_0000  |      +--------------+     |                |     +--------------+       |  0x9600_0000                
               |      | video engine |     |                |     | video engine |       |                             
  0x9800_0000  +------+--------------+   --+                +--   +--------------+-------+  0x9800_0000                
               |     flash    |                                                                                        
  0x9C00_0000  +------+--------------+   --+                +--   +--------------+-------+  0x9C00_0000                
               |      |   DMA/CMA    |     |                |     |    DMA/CMA   |       |                             
  0x9D00_0000  |      +--------------+     |                |     +--------------+       |  0x9D00_0000                
               |      |     GFX      |     |  map_lowmem()  |     |      GFX     |       |                             
  0x9E00_0000  |      +--------------+     | <------------> |     +--------------+       |  0x9E00_0000                
               |              |            |                |             |              |                             
  0x9EE0_0000  |              |          --+                +--           +--------------+  0x9EE0_0000                
               |              |                                                                                        
  0x9EF0_0000  +--------------+                                                                                        
               |   coldfire   |                                                                                        
  0x9F00_0000  +--------------+                                           +--------------+  0x9F00_0000, VMALLOC_START 
               |     VGA      |                                           |    vmalloc   |                             
  0xA000_0000  +--------------+                                           |              |                             
                                                                          |              |                             
                                                                          |              |                             
                                                                          +--------------+  0xFF80_0000, FDT_FIXED_BASE
                                                                          |      DTB     |                             
                                                                          +--------------+  0xFFA0_0000                
                                                                                                                       
                                                                          +--------------+  0xFFC8_0000, FIXADDR_START 
                                                                          |    fixmap    |                             
                                                                          +--------------+  0xFFF0_0000, FIXADDR_END   
                                                                                                                       
                                                                          +--------------+  0xFFFF_0000                
                                                                          |    vector    |                             
                                                                          +--------------+  0xFFFF_1000                                
```

- Code flow

```
+-------------+                                                                                           
| paging_init |                                                                                           
+---|---------+                                                                                           
    |    +--------------------+                                                                           
    +--> | prepare_page_table | reset all the mappings except the 1st memblock so the kernel can still run
    |    +--------------------+                                                                           
    |    +------------+                                                                                   
    +--> | map_lowmem | map all the available memblocks but avoid kernel region                           
    |    +------------+                                                                                   
    |    +------------+                                                                                   
    +--> | map_kernel |  map kernel for its executable and non-executable parts separately                
    |    +------------+                                                                                   
    |    +----------------------+                                                                         
    +--> | dma_contiguous_remap | remap DMA regions: video engine, DMA/CMA, and GFX                       
    |    +----------------------+                                                                         
    |    +-----------------------+                                                                        
    +--> | early_fixmap_shutdown | map 'fixmap'                                                           
    |    +-----------------------+                                                                        
    |    +-----------------+                                                                              
    +--> | devicemaps_init | map other regions: vmalloc, DTB, vector, etc                                 
    |    +-----------------+                                                                              
    |    +-----------+                                                                                    
    +--> | kmap_init | map 'kmap'                                                                        
    |    +-----------+                                                                                    
    |    +--------------+                                                                                 
    +--> | bootmem_init | we'll introduce this function later                                             
         +--------------+                                                                                 
```

## <a name="node-and-zone"></a> Node and Zone

Before introducing the function _bootmem_init_, let's talk about node and zone first.
For multiple processors sharing the memory that doesn't serve a specific CPU, we call this memory a 'node,' and it's the typical structure on most devices.
Rumor has those large systems that I've never seen one by myself have multiple 'nodes,' and each CPU has its preferred one.
Accessing nearby memory costs less time while visiting remote ones takes longer.
Here are the formal names for the above two designs, and our study case belongs to the first one.

- UMA
   - Uniform Memory Access
   - There's only node 0
- NUMA
   - Non-Uniform Memory Access
   - Multiple nodes exist

Each node consists of a few zones that serve different purposes.

- DMA zone
   - Some old fashion devices have limitations that can only access memory below a specific threshold, and the memory range below this line is the DMA zone.
   - Let's ignore this one since it got disabled in our study case. Hooray!
- Normal zone
   - AST2500 adopts ARM32, which supports up to 4G of user and kernel space combined.
   - Though it's 2G/2G for user/kernel space in our case, 3G/1G is also pervasive.
   - It's pretty straightforward that we can't map more than 2G into kernel space, and it's even less since Linux reserves some virtual ranges for other usages.
   - These direct-mapped page frames belong to 'DMA' or 'normal' zones.
- Highmem zone
   - If we ask the kernel to manage much memory more than it can accommodate, it falls into 'highmem.'
   - This zone might be empty if we disable the CONFIG_HIGHMEM or provide not much DRAM for memory management.
   - Our case is kind of tricky. With enabled config and enough memory, kernel forcibly drops highmem because of the VIPT cache type. No idea why.
   - Relative to highmem, we call the 'DMA' and 'normal' zones together as lowmem.

Function _bootmem_init_ initializes the node and zone structure,  and in our study case, it's one node and two zones (normal + empty highmem).

```
+--------------+                                                                                                                               
| bootmem_init |                                                                                                                               
+---|----------+                                                                                                                               
    |    +-----------------+                                                                                                                   
    +--> | zone_sizes_init |                                                                                                                   
         +----|------------+                                                                                                                   
              |    +----------------+                                                                                                          
              +--> | free_area_init |                                                                                                          
                   +---|------------+                                                                                                          
                       |                                                                                                                       
                       +--> print log                                                                                                          
                       |        [    0.000000]   Normal   [mem 0x0000000080000000-0x000000009eefffff]                                          
                       |        [    0.000000]   HighMem  empty                                                                                
                       |        [    0.000000] Movable zone start for each node                                                                
                       |        [    0.000000] Early memory node ranges                                                                        
                       |        [    0.000000]   node   0: [mem 0x0000000080000000-0x0000000097ffffff]                                         
                       |        [    0.000000]   node   0: [mem 0x0000000098000000-0x000000009bffffff]                                         
                       |        [    0.000000]   node   0: [mem 0x000000009c000000-0x000000009edfffff]                                         
                       |                                                                                                                       
                       |    +---------------------+                                                                                            
                       +--> | free_area_init_node |                                                                                            
                            +-----|---------------+                                                                                            
                                  |                                                                                                            
                                  |--> set up the node                                                                                         
                                  |                                                                                                            
                                  +--> print log                                                                                               
                                  |        [    0.000000] Initmem setup node 0 [mem 0x0000000080000000-0x000000009edfffff]                     
                                  |                                                                                                            
                                  |    +--------------------+                                                                                  
                                  |--> | alloc_node_mem_map | prepare 'struct page' array and each represents its corresponding '4K page frame'
                                  |    +--------------------+                                                                                  
                                  |    +---------------------+                                                                                 
                                  +--> | free_area_init_core | set up zones                                                                    
                                       +---------------------+                                                                                 
```


### Vmalloc area
   - Memory from this region is guaranteed to be virtually consecutive.
### Fixmap
   - I don't know.



```
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
