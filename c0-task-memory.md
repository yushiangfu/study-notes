> The note is based on Linux version 5.15.43 in OpenBMC.

## Index

- [Introduction](#introduction)
- [Memory Layout](#memory-layout)
- [Page Table](#page-table)
- [Page Fault](#page-fault)
- [Task Startup](#task-startup)
- [Others](#others)
  - Reverse Mapping
  - Kmap
- [Reference](#reference)

## <a name="introduction"></a> Introduction

(TBD)

## <a name="memory-layout"></a> Memory Layout

The executable primarily consists of code, data, and other segments. 
Prior to running the program's main parts in memory, additional mappings are necessary.

- Main program:
  - The main program consists of three mappings: `text`, `data`, and `bss`. 
  - Debug information and other segments are unnecessary and won't be loaded into a mapping.
- Stack:
  - The stack is utilized for the function call mechanism and is downward expandable, meaning its mapping size can change. 
  - Each thread within a process has its own stack.
- Heap:
  - The heap is utilized by functions like `malloc()` and similar operations. 
  - It is upward expandable, allowing its mapping size to change.
- Libraries:
  - Libraries, such as the standard C library or thread library, also occupy three mappings, similar to the main program.
- Linker:
  - The linker assists in loading a series of specified libraries into memory. 
  - It occupies three mappings, although it is expected to be an executable rather than a shared library, commonly known as `ld.so` or `ld-linux.so`.
- Others:
  - The `sigpage`, `vvar`, and `vdso` segments have specific functions that are currently unknown to me

Each region has its own set of permissions and flags. 
The kernel utilizes the virtual memory area (`vma`) structure to describe each region. 
From the content source's perspective, regions can be either file mappings if backed by a file or anonymous mappings, as seen in the stack and heap.

<p align="center"><img src="images/task-memory/memory-layout.png" /></p>

<details><summary> More Details </summary>

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
 
```c
struct mm_struct {
    struct {
        unsigned long start_code, end_code, start_data, end_data;
        unsigned long start_brk,    // the start addr of heap
                      brk,          // current end addr of heap, but it's changeable
                      start_stack;
        unsigned long arg_start, arg_end, env_start, env_end;
    }
}
```

```
+-----------------------+                                                       
| arch_pick_mmap_layout | : decide mmap_base & ->get_unmapped_area() of given mm
+-----|-----------------+                                                       
      |                                                                         
      |--> if flag has specified RANDOMIZE                                      
      |                                                                         
      |------> get a random factor                                              
      |                                                                         
      |--> if it's legacy mmap (not our case)                                   
      |                                                                         
      |        +-----------+                                                    
      |------> | mmap_base | determine mmap base                                
      |        +-----------+                                                    
      |                                                                         
      +------> set ->get_unmapped_area = arch_get_unmapped_area_topdown         
```
  
```
+--------------------------------+
| arch_get_unmapped_area_topdown | : lookup a region that satisfies the range ('addr' isn't guaranteed)
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
        |--> | vm_unmapped_area | : find a suitable region and return start addr
        |    +----|-------------+
        |         |
        |         |--> if flag is top_down
        |         |
        |         |        +-----------------------+
        |         |------> | unmapped_area_topdown | starting from high address,
        |         |        +-----------------------+ find a suitable region and return start addr
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
+-------------------+
| get_unmapped_area | : lookup a region that satisfies the range ('addr' isn't guaranteed)
+----|--------------+
     |
     |--> determine get_area(): 1. from mm, 2. from file, 3. from shm
     |
     +--> call ->get_area(), e.g.,
          +--------------------------------+
          | arch_get_unmapped_area_topdown | lookup a region that satisfies the range ('addr' isn't guaranteed)
          +--------------------------------+
```
  
```c
struct vm_area_struct {
    unsigned long vm_start;           // vaddr start
    unsigned long vm_end;             // vaddr end
    struct vm_area_struct *vm_next,   // list node
                          *vm_prev;
    struct rb_node vm_rb;             // tree node
    struct mm_struct *vm_mm;          // points to mm
    pgprot_t vm_page_prot;            // protection permissions
    unsigned long vm_flags;           // flags
    struct {
        struct rb_node rb;
        unsigned long rb_subtree_last;
    } shared;                         // priority tree node
    struct list_head anon_vma_chain;  // reverse mapping: list node
    struct anon_vma *anon_vma;        // reverse mapping: points to management structure
    const struct vm_operations_struct *vm_ops;  // operations, including ->fault()
    unsigned long vm_pgoff;           // file mapping: offset within file (unit: page size)
    struct file * vm_file;            // file mapping: points to file
    void * vm_private_data;           // only a few audio/vidoe drivers use it
}
```
  
```c
struct address_space {
    struct inode        *host;      // can be file or block dev
    struct rb_root_cached   i_mmap; // tree of private and shared mappings
}
```
  
```c
struct file {
    struct address_space    *f_mapping;
} 
```
  
```c
struct inode {
    struct address_space    *i_mapping;
}
```
  
```
+----------+                                                
| find_vma | : lookup the first vma that meets addr < vm_end
+--|-------+                                                
   |    +---------------+                                   
   |--> | vmacache_find | check if target is in the cache   
   |    +---------------+                                   
   |                                                        
   |--> return if found                                     
   |                                                        
   |--> search in tree                                      
   |                                                        
   +--> if found, update cache                              
```
  
```
+-----------------------+                                                    
| find_vma_intersection | : given the range, return the first intersected vma
+-----|-----------------+                                                    
      |    +----------+                                                      
      |--> | find_vma | lookup the first vma that meets addr < vm_end        
      |    +----------+                                                      
      |                                                                      
      |--> return null if they don't intersect                               
      |                                                                      
      +--> return the found vma                                              
```
  
```
+------------------+                                                              
| insert_vm_struct | : insert vma into process's mm and inode's i_mmap tree       
+----|-------------+                                                              
     |    +----------------+                                                      
     |--> | find_vma_links | given range, return its list prev, tree parent & link
     |    +----------------+                                                      
     |                                                                            
     |--> if it's an anonymous vma                                                
     |                                                                            
     |------> set ->vm_pgoff to be consistent with /proc/pid/maps?                
     |                                                                            
     |    +----------+                                                            
     +--> | vma_link | link vma into mm and address_space                         
          +----------+                                                            
```
  
```
+----------+                                                                                   
| vma_link | : link vma into mm and address_space                                              
+--|-------+                                                                                   
   |    +------------+                                                                         
   |--> | __vma_link | :
   |    +--|---------+                                                                         
   |       |    +-----------------+                                                            
   |       |--> | __vma_link_list | link vma into list in mm                                   
   |       |    +-----------------+                                                            
   |       |    +---------------+                                                              
   |       +--> | __vma_link_rb | link vma into tree in mm                                     
   |            +---------------+                                                              
   |    +-----------------+                                                                    
   +--> | __vma_link_file | link vma into tree of address space, for private or shared mappings
        +-----------------+                                                                    
```
                    
```
+-----------+
| sys_mmap2 | :
+--|--------+
   |    +----------------+
   +--> | sys_mmap_pgoff | :
        +---|------------+
            |    +-----------------+
            +--> | ksys_mmap_pgoff | :
                 +----|------------+
                      |    +---------------+
                      +--> | vm_mmap_pgoff | :
                           +---|-----------+
                               |    +---------+
                               |--> | do_mmap | :
                               |    +--|------+
                               |       |    +-------------------+
                               |       |--> | get_unmapped_area | lookup a region that satisfies the range,
                               |       |    +-------------------+ but it's not guaranteed
                               |       |
                               |       |--> determine vm flags
                               |       |
                               |       |    +-------------+
                               |       |--> | mmap_region | ensure there's a vma covering this region,
                               |       |    +-------------+ link that vma with framework
                               |       |
                               |       +--> determine populate len if the flags (e.g., VM_LOCKED) ask so
                               |
                               |--> if populate len is determined
                               |
                               |        +-------------+
                               +------> | mm_populate | fault in the pages
                                        +-------------+
```

```
+-------------+                                                                               
| mmap_region | : ensure there's a vma covering this region, link that vma with framework       
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
    |        +----------+    +-------------------+
    |        | ovl_mmap | or | generic_file_mmap |
    |        +----------+    +-------------------+
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
    +--> | vma_link | link vma into mm and address_space                                              
         +----------+
```
  
```
+------------+
| sys_munmap | :
+--|---------+
   |    +-------------+
   +--> | __vm_munmap | :
        +---|---------+
            |    +-------------+
            +--> | __do_munmap | : clear pte, free page table, and release vma
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
                     |--> | unmap_region | :
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
| sys_mlockall | : apply flags to mm and all vma of the current task               
+---|----------+                                                                   
    |                                                                              
    |--> check if the request is allowed                                           
    |                                                                              
    |    +----------------------+                                                  
    +--> | apply_mlockall_flags | apply flags to mm and all vma of the current task
         +----------------------+                                                  
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
  
</details>

## <a name="page-table"></a> Page Table

Please note that memory management can vary across different platforms, and the example provided is specific to the AST2500 kernel. 
In this system, memory mapping information is stored in the page table, which enables the hardware (MMU) to translate virtual addresses into physical addresses. 
The page table is designed as a multi-level structure, and our case study employs a two-level structure.

The first-level table consists of 4096 entries, dividing the 4GB virtual space into 1MB regions, with each region corresponding to one entry in the table. 
The possible values for entries are as follows:

- Unmapped: 
  - Accessing this entry triggers a page fault, which is the expected behavior at the hardware level.
- Section mapping: 
  - The value stored in the entry represents the destination physical address. 
  - This mapping size is 1MB and is only used in the kernel space.
- Pointer to a second-level table: 
  - When necessary, a second-level table is created to save memory usage.

The second-level table consists of 256 entries, where each 4KB area within a section has a corresponding entry in the table. 
The possible values for entries in the second-level table are as follows:

- Unmapped: 
  - Accessing this entry triggers a page fault as per the hardware behavior.
- Page mapping: 
  - The value stored in the entry represents the destination physical address. 
  - This mapping size is 4KB and is used in both the kernel and user spaces.

<p align="center"><img src="images/task-memory/page-table.png" /></p>
  
The memory management (`mm`) structure manages the virtual memory area (`vma`) and page table. 
When creating a task, the `mm` structure can be shared or cloned. 
Threads within a process, whether single-threaded or multi-threaded, share the same mm structure, resulting in an identical virtual space. 
Kernel threads also share the initial `mm` structure, but they don't have user space mappings and, therefore, no `vma`. 
During a context switch, the Translation Table Base Register (`TTBR`) is updated to point to the next task's page table, enabling the `MMU` to perform address lookup correctly.

<p align="center"><img src="images/task-memory/page-table-and-task.png" /></p>

<details><summary> More Details </summary>
  
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

</details>
                                                    
## <a name="page-fault"></a> Page Fault
  
When a processor encounters an inaccessible area and fails to access it, a data abort exception occurs, which is then trapped by the kernel as a page fault. 
A common example of a page fault is a segmentation fault, often caused by dereferencing a null or incorrect pointer, indicating a genuine issue. 
However, it's important to note that a page fault does not always indicate an error. 
Sometimes, the virtual memory mechanism relies on page faults to complete the mapping process.

To illustrate this with a 4K mapping example, three components are required for a successful virtual address lookup:

- Virtual address: 
  - The user specifies the address either through the `mmap` argument or the kernel finds a suitable region that meets the size requirement.
- Physical page frame: 
  - A new page frame is requested from the buddy memory allocation to serve as the destination for the mapping.
- Page table: 
  - Using the virtual and physical addresses, the kernel determines the appropriate entries in both the first-level and second-level page tables to populate with the corresponding values. 
  - If the second-level page table doesn't exist yet, it is created.

In practice, when a user calls `mmap` to create an anonymous or file mapping, only the virtual memory area (`vma`) is set up initially. 
It is not until the region is accessed that the page table entry and physical page are prepared in time for subsequent read or write actions.
  
<p align="center"><img src="images/task-memory/page-fault.png" /></p>
  
<details><summary> More Details </summary>

As the CPU exception, there are two types of abort on ARM architecture.

- Data abort: data fault
- Prefetch abort: instruction fault

And two separate registers are designed to save the exception status when the corresponding abort happens.

- FSR: fault status register
- IFSR: instruction fault status register

A fault isn't strictly equivalent to invalid memory access. Instead, a few cases are designed to be that way.

- Copy on write
- Fault on VMA of anonymous type
- Fault on VMA of file type

```
              +-  from user    --+                                                      
 Prefetch     |                  | (refer to IFSR)
  Abort  -----+                  |                                                      
              +-  from kernel  --|                             1. copy on write         
                                 |                             2. valid fault: anonymous
                                 +-------- handle page fault   3. valid fault: file     
                                 |                             4. genuine fault            
              +-  from user    --|                             
  Data        |                  |                                                      
  Abort  -----+                  | (refer to FSR)                                                    
              +-  from kernel  --+                                                      
```

Based on the index fetched from FSR or IFSR, the corresponding function, mainly **do_page_fault**, will be called to handle the fault correctly.

```
arch/arm/mm/fsr-2level.c
static struct fsr_info fsr_info[] = { 
    { do_bad,               SIGSEGV, 0,             "vector exception"         },  
    { do_bad,               SIGBUS,  BUS_ADRALN,    "alignment exception"          },  
    { do_bad,               SIGKILL, 0,             "terminal exception"           },  
    { do_bad,               SIGBUS,  BUS_ADRALN,    "alignment exception"          },  
    { do_bad,               SIGBUS,  0,             "external abort on linefetch"      },  
    { do_translation_fault, SIGSEGV, SEGV_MAPERR,   "section translation fault"    },  
    { do_bad,               SIGBUS,  0,             "external abort on linefetch"      },  
    { do_page_fault,        SIGSEGV, SEGV_MAPERR,   "page translation fault"       },  
    { do_bad,               SIGBUS,  0,             "external abort on non-linefetch"  },  
    { do_bad,               SIGSEGV, SEGV_ACCERR,   "section domain fault"         },  
    { do_bad,               SIGBUS,  0,             "external abort on non-linefetch"  },  
    { do_bad,               SIGSEGV, SEGV_ACCERR,   "page domain fault"        },  
    { do_bad,               SIGBUS,  0,             "external abort on translation"    },  
    { do_sect_fault,        SIGSEGV, SEGV_ACCERR,   "section permission fault"     },  
    { do_bad,               SIGBUS,  0,             "external abort on translation"    },  
    { do_page_fault,        SIGSEGV, SEGV_ACCERR,   "page permission fault"        },  
...
}

static struct fsr_info ifsr_info[] = {
    { do_bad,               SIGBUS,  0,             "unknown 0"            },
    { do_bad,               SIGBUS,  0,             "unknown 1"            },
    { do_bad,               SIGBUS,  0,             "debug event"              },
    { do_bad,               SIGSEGV, SEGV_ACCERR,   "section access flag fault"    },
    { do_bad,               SIGBUS,  0,             "unknown 4"            },
    { do_translation_fault, SIGSEGV, SEGV_MAPERR,   "section translation fault"    },
    { do_bad,               SIGSEGV, SEGV_ACCERR,   "page access flag fault"       },
    { do_page_fault,        SIGSEGV, SEGV_MAPERR,   "page translation fault"       },
    { do_bad,               SIGBUS,  0,             "external abort on non-linefetch"  },
    { do_bad,               SIGSEGV, SEGV_ACCERR,   "section domain fault"         },
    { do_bad,               SIGBUS,  0,             "unknown 10"               },
    { do_bad,               SIGSEGV, SEGV_ACCERR,   "page domain fault"        },
    { do_bad,               SIGBUS,  0,             "external abort on translation"    },
    { do_sect_fault,        SIGSEGV, SEGV_ACCERR,   "section permission fault"     },
    { do_bad,               SIGBUS,  0,             "external abort on translation"    },
    { do_page_fault,        SIGSEGV, SEGV_ACCERR,   "page permission fault"        },
...
}
```

```
 +-------------+                                                                       
 | vector_pabt | :
 +--|----------+                                                                       
    |                                                                                  
    +---> save r0, lr_pabt, spsr_pabt to abt stack (size: 4 bytes * 3)                 
    |                                                                                  
    +---> switch to SVC mode                                                           
    |                                                                                  
    +---> determine which mode we comes from                                           
    |                                                                                  
    +---> save abt stack address to r0                                                 
    |                                                                                  
    +---> if we were at USR mode                                                       
    |                                                                                  
    |         +------------+                                                           
    +-------> | __pabt_usr | get ifsr and handle fault accordingly, return to user mode
    |         +------------+                                                           
    |                                                                                  
    +---> elif we were at SVC mode                                                     
    |                                                                                  
    |         +------------+                                                           
    +-------> | __pabt_svc | get ifsr and handle fault accordingly                     
              +------------+                                                           
```

```
+------------+                                                                             
| __pabt_usr | : get ifsr and handle fault accordingly, return to user mode                  
+--|---------+                                                                             
   |    +-----------+                                                                      
   |--> | usr_entry | store r0 ~ r12, sp_usr, lr_usr, old pc to sp_svc                     
   |    +-----------+                                                                      
   |    +-------------+                                                                    
   |--> | pabt_helper | :                                                                    
   |    +---|---------+                                                                    
   |        |                                                                              
   |        +--> call v6_processor_functions->_prefetch_abort(), e.g.,                     
   |             +-----------+                                                             
   |             | v6_pabort | get ifsr and handle the fault accordingly                   
   |             +-----------+                                                             
   |    +--------------------+                                                             
   +--> | ret_from_exception | handle pending work, restore user mode regs and switch to it
        +--------------------+                                                             
```

```
+-----------+                                                
| v6_pabort | : get ifsr and handle the fault accordingly      
+--|--------+                                                
   |                                                         
   |--> get ifsr from coprocessor                            
   |                                                         
   |    +------------------+                                 
   +--> | do_PrefetchAbort | :
        +----|-------------+                                 
             |                                               
             |--> get target ifsr_info using ifsr as index   
             |                                               
             |--> call ->fn(), e.g.,                         
             |    +---------------+                          
             |    | do_page_fault | handle page fault        
             |    +---------------+                          
             |                                               
             |--> if it's handled properly, return           
             |                                               
             |    +----------------+                         
             +--> | arm_notify_die |                         
                  +----------------+                         
```

```
+------------+                                                                            
| __pabt_svc | : get ifsr and handle fault accordingly                                      
+--|---------+                                                                            
   |    +-----------+                                                                     
   |--> | svc_entry | save registers to stack for later restore                           
   |    +-----------+                                                                     
   |    +-------------+                                                                   
   |--> | pabt_helper | call v6_processor_functions->_prefetch_abort() to handle the fault
   |    +-------------+                                                                   
   |    +----------+                                                                      
   +--> | svc_exit | restore registers from stack                                         
        +----------+                                                                      
```

```
 +-------------+                                                                      
 | vector_dabt | :
 +--|----------+                                                                      
    |                                                                                 
    +---> save r0, lr_dabt, spsr_dabt to abt stack (size: 4 bytes * 3)                
    |                                                                                 
    +---> switch to SVC mode                                                          
    |                                                                                 
    +---> determine which mode we comes from                                          
    |                                                                                 
    +---> save abt stack address to r0                                                
    |                                                                                 
    +---> if we were at USR mode                                                      
    |                                                                                 
    |         +------------+                                                          
    +-------> | __dabt_usr | get fsr and handle fault accordingly, return to user mode
    |         +------------+                                                          
    |                                                                                 
    +---> elif we were at SVC mode                                                    
    |                                                                                 
    |         +------------+                                                          
    +-------> | __dabt_svc | get fsr and handle fault accordingly                     
              +------------+                                                          
```

```
+------------+                                                                             
| __dabt_usr | : get fsr and handle fault accordingly, return to user mode                   
+--|---------+                                                                             
   |    +-----------+                                                                      
   |--> | usr_entry | store r0 ~ r12, sp_usr, lr_usr, old pc to sp_svc                     
   |    +-----------+                                                                      
   |    +-------------+                                                                    
   |--> | dabt_helper | :
   |    +---|---------+                                                                    
   |        |                                                                              
   |        +--> call v6_processor_functions->_data_abort(), e.g.,                         
   |             +----------------+                                                        
   |             | v6_early_abort | get fsr and handle the fault accordingly               
   |             +----------------+                                                        
   |    +--------------------+                                                             
   +--> | ret_from_exception | handle pending work, restore user mode regs and switch to it
        +--------------------+                                                             
```

```
+----------------+                                         
| v6_early_abort | : get fsr and handle the fault accordingly
+---|------------+                                         
    |                                                      
    |--> get 'fsr' and 'far' from coprocessor              
    |                                                      
    |    +--------------+                                  
    +--> | do_DataAbort | :
         +---|----------+                                  
             |                                             
             |--> get target fsr_info using fsr as index   
             |                                             
             |--> call ->fn(), e.g.,                       
             |    +---------------+                        
             |    | do_page_fault | handle page fault      
             |    +---------------+                        
             |                                             
             |--> if it's handled properly, return
             |                                             
             |    +----------------+                       
             +--> | arm_notify_die |                       
                  +----------------+                       
```

```
+------------+                                                                     
| __dabt_svc | : get fsr and handle fault accordingly                                
+--|---------+                                                                     
   |    +-----------+                                                              
   |--> | svc_entry | save registers to stack for later restore                    
   |    +-----------+                                                              
   |    +-------------+                                                            
   |--> | dabt_helper | call processor specific ->_data_abort() to handle the fault
   |    +-------------+                                                            
   |    +----------+                                                               
   +--> | svc_exit | restore registers from stack                                  
        +----------+                                                               
```

</details>
  
### Page Fault Handling
  
The following cases are genuine errors that can lead to the termination of a task with the `SIGSEGV` signal:

- Accessing the kernel space from a user space task.
- Dereferencing a pointer that points to an unmapped area, including null pointers.

When a page fault occurs, the kernel takes the following steps:

1. Checks if the faulted address belongs to any virtual memory area (`vma`).
2. Allocates a page from the buddy system if the address is within a `vma`.
3. Handles the page content based on the fault type.
4. Updates the first-level and second-level page table entries to establish the virtual-to-physical address mapping.
5. Resumes the task's execution and allows the data operation to continue.
  
The involved task continues to encounter faults in other regions without being aware of the underlying procedure. 
Regarding step 3, here are some common fault types and their corresponding data handling:

- Anonymous mapping: The allocated page is simply zeroed out.
- File mapping: Data is read from the backed file into the page.
- Copy on write (COW): This occurs when a parent task forks a child, which belongs to a new process. The page table and vma list are duplicated as memory resources for the new task, while the page frames remain unchanged. Both tasks share the same physical pages because their page table entries are identical. This technique saves significant effort in copying data, especially since the child often immediately calls execve(). If either the parent or child attempts to write data into a shared and writable mapping, the kernel detects the specific type of page fault. The data is then duplicated from the old page to the newly allocated one, and the task can continue its execution.
   
<p align="center"><img src="images/task-memory/copy-on-write.png" /></p>
  
<details><summary> More Details </summary>

### Fault on Anonymous Area

```
                                  accessing any of                                      
                                  these areas triggers                                  
                                  a valid fault                                         
 physical address                              |           virtual address              
                                               |                                        
 |              |                              |           |              |low          
 |              |                              |           |              |             
 |%%%%%%%%%%%%%%| ------------+                |-------->  |##############| <---- vma   
 |%%%%%%%%%%%%%%|             |                |           |              |             
 |%%%%%%%%%%%%%%|             |                |           |              |             
 |%%%%%%%%%%%%%%|             |                |-------->  |##############| <---- vma   
 |              |             |                |           |              |             
 |              |             |                +-------->  |##############| <---- vma   
 |              |             |   +------+                 |              |             
 |              |             |   |      |                 |              | user space  
 |              |             |   |      |            -------------------------         
 |              |             |   |      |                 |              | kernel space
 |              |             |   |      |         |-----  |##############|             
 |              |             +-- |------|  -------+       |              |             
 |              |                 +------+                 |              |             
 |              |                                          |              |             
 |              |                                          |              |             
 |              |                                          |              |             
 |              |                                          |              |             
 |              |                                          |              |             
 |              |                                          |              | high        
```
        
### Copy on Write

```
   parent                                                                       child                  
    task                                                                         task                  
  +------+                                                                     +------+                
+--- mm  |                                                                     |  mm ---+              
| +------+                                                                     +------+ |              
|                                                                                       |              
+--> mm                                                                           mm <--+              
  +------+                                                                     +------+                
+---pgd  |                             page frames                             | pgd----+              
| +------+                              +-------+                              +------+ |              
|                                       +-------+                                       |              
| 1st level   +-> 2nd level      +->    +-------+    <--+     2nd level <--+  1st level |              
+->+-----+    |    +-----+       |      +-------+       |      +-----+     |   +-----<--+              
   |-----|    |    |-----|       |      +-------+       -      |-----|     -   |-----|                 
   |-----| ---|    |-----|    ---|      +-------+       +--    |-----|     +-- |-----|                 
   |-----|         |-----|              +-------+              |-----|         |-----|                 
   |-----| ---|    |-----|    ----->    +-------+    <-----    |-----|     +-- |-----|                 
   +-----+    |    +-----+              +-------+              +-----+     |   +-----+                 
              |                         +-------+                          |                           
              +-> 2nd level             +-------+             2nd level <--|                           
                   +-----+              +-------+              +-----+                                 
                   |-----|    ----->    +-------+    <-----    |-----|                                 
                   |-----|              +-------+              |-----|                                 
                   |-----|              +-------+              |-----|                                 
                   |-----|    ----->    +-------+       ---    |-----|                                 
                   +-----+              +-------+    <-/       +-----+                                 
                                        +-------+                                                      
                                            -                                                          
                                            -        e.g.,                                             
                                            -        1. the child tries to write the page              
                                                     2. fault happens and a duplicated page is prepared
                                                     3. update page table entry to point to it         
```
          
```
                                                        +-------------------+         +---------------+  
                                                   +--- | do_anonymous_page |    +--- | do_read_fault |  
                                                   |    +-------------------+    |    +---------------+  
                                                   |    +----------+             |    +--------------+   
                                                   |--- | do_fault | ------------|--- | do_cow_fault |   
                                                   |    +----------+             |    +--------------+   
                                                   |    +------------+           |    +-----------------+
                                                   |--- | do_wp_page |           +--- | do_shared_fault |
                                                   |    +------------+                +-----------------+
                                              +------------------+                                       
                                         +--- | handle_pte_fault |                                       
                                         |    +------------------+                                       
                                    +-----------------+                                                  
                               +--- | handle_mm_fault |                                                  
                               |    +-----------------+                                                  
                               |    +--------------+                                                     
                               |--- | expand_stack |                                                     
                               |    +--------------+                                                     
                          +-----------------+                                                            
                     +--- | __do_page_fault |                                                            
                     |    +-----------------+                                                            
+---------------+    |    +-----------------+                                                            
| do_page_fault | ---|--- | __do_user_fault |                                                            
+---------------+    |    +-----------------+                                                            
                     |    +-------------------+     +-----------------+                                  
                     +--- | __do_kernel_fault | --- | fixup_exception |                                  
                          +-------------------+     +-----------------+                                  
```
                                                        
```
+---------------+
| do_page_fault | :
+---|-----------+
    |    +-----------------+
    |--> | __do_page_fault | handle mm fault, or simply expand downward if it's stack
    |    +-----------------+
    |
    |--> return 0 for valid case
    |
    |--> if it's a real fault from user context
    |
    |        +-----------------+
    |------> | __do_user_fault | raise signal SIGSEGV
    |        +-----------------+
    |
    |--> else if it's a real fault from kernel context
    |
    |        +-------------------+
    +------> | __do_kernel_fault | dump info and send KILL to current task
             +-------------------+
```
  
```
+-----------------+
| __do_page_fault | : handle mm fault, or simply expand downward if it's stack
+----|------------+
     |    +----------+
     |--> | find_vma | try to find target vma based on faulted addr
     |    +----------+
     |
     |--> if faulted addr < vma start
     |
     |------> go to 'check_stack'
     |
     |--> return error if not
     |
     |    +-----------------+
     +--> | handle_mm_fault |
     |    +-----------------+
     |
     +--> return
     |
     +--> if the vma is stack
check_stack
     |        +--------------+
     +------> | expand_stack |
              +--------------+
```
  
```
+-----------------+
| handle_mm_fault | :
+----|------------+
     |    +---------------------+
     |--> | __set_current_state | set state = running
     |    +---------------------+
     |    +-------------------+
     +--> | __handle_mm_fault | :
          +----|--------------+
               |    +------------+
               |--> | pgd_offset | get target pgd based on addr
               |    +------------+
               |    +-----------+
               |--> | p4d_alloc | return arg pgd
               |    +-----------+
               |    +-----------+
               |--> | pud_alloc | return arg p4d
               |    +-----------+
               |    +-----------+
               |--> | pmd_alloc |  return arg pud
               |    +-----------+
               |    +------------------+
               +--> | handle_pte_fault | ensure 2nd-level table exists,
                    +------------------+ and either call ->fault() or simply update pte entry
```

```
 +------------------+                                                                                     
 | handle_pte_fault | : ensure 2nd-level table exists, and either call ->fault() or simply update pte entry 
 +----|-------------+                                                                                     
      |                                                                                                   
      |--> if no pte yet                                                                                  
      |                                                                                                   
      |------> if vma is anon type                                                                        
      |                                                                                                   
      |            +-------------------+                                                                  
      |----------> | do_anonymous_page | ensure 2nd-level table exists, prepare pte value and update entry
      |            +-------------------+ (aka demand allocation)
      |                                                                                                   
      |------> else                                                                                       
      |                                                                                                   
      |            +----------+                                                                           
      +----------> | do_fault | ensure 2nd-level table exists, call ->fault()                             
      |            +----------+ (aka demand paging)
      |                                                                                                   
      |--> if flag has specified 'write'                                                                  
      |                                                                                                   
      |        +------------+                                                                             
      +------> | do_wp_page | handle the fault of write-protect?
               +------------+                                                                             
```

```
+-------------------+                                                                  
| do_anonymous_page | : ensure 2nd-level table exists, prepare pte value and update entry
+----|--------------+                                                                  
     |    +-----------+                                                                
     |--> | pte_alloc | ensure the pte table exists                                    
     |    +-----------+                                                                
     |    +------------------------------------+                                       
     |--> | alloc_zeroed_user_highpage_movable | allocate a highmem page               
     |    +------------------------------------+                                       
     |    +--------+                                                                   
     |--> | mk_pte | prepare pte value                                                 
     |    +--------+                                                                   
     |    +------------------------+                                                   
     |--> | page_add_new_anon_rmap | label 'swap backed' on page, and save rmap info to it
     |    +------------------------+                                                   
     |    +---------------------------------------+                                    
     |--> | lru_cache_add_inactive_or_unevictable | (skip for now)                     
     |    +---------------------------------------+                                    
     |    +------------+                                                               
     +--> | set_pte_at | update pte entry                                              
          +------------+                                                               
```

```
+----------+                                                                                   
| do_fault | : ensure 2nd-level table exists, call ->fault()                                     
+--|-------+                                                                                   
   |                                                                                           
   |--> if vm ops doesn't have ->fault(), return error                                         
   |                                                                                           
   |--> else if it's not a 'write'                                                             
   |                                                                                           
   |        +---------------+                                                                  
   |------> | do_read_fault | call ->map_pages(), ensure 2nd-level table exists, call ->fault()
   |        +---------------+                                                                  
   |                                                                                           
   |--> else if it's a non-shared mapping                                                      
   |                                                                                           
   |        +--------------+                                                                   
   |------> | do_cow_fault | ensure 2nd-level table exists, call ->fault(), copy data to it    
   |        +--------------+                                                                   
   |                                                                                           
   |--> else                                                                                   
   |                                                                                           
   |        +-----------------+                                                                
   +------> | do_shared_fault | ensure 2nd-level table exists, call ->fault(), dirty the page  
            +-----------------+                                                                
```

```
+---------------+                                              
| do_read_fault | : ensure 2nd-level table exists, call ->fault()
+---|-----------+                                              
    |                                                          
    |--> if vm ops has ->map_pages()                           
    |                                                          
    |        +-----------------+                               
    |------> | do_fault_around | :                              
    |        +----|------------+                               
    |             |                                            
    |             |--> determine start and end page offset     
    |             |                                            
    |             +--> call ->map_pages(), e.g.,               
    |                  +-------------------+                   
    |                  | filemap_map_pages |                   
    |                  +-------------------+                   
    |    +------------+                                        
    +--> | __do_fault | ensure 2nd-level table exists, call ->fault()
         +------------+                                        
                         
```
  
```
+------------+
| __do_fault | : ensure 2nd-level table exists, call ->fault()
+--|---------+
   |
   |--> ensure 2nd-level table exists
   |
   +--> call ->fault(), e.g.,
        +---------------+
        | filemap_fault | read ahead (async or sync)
        +---------------+
```
  
```c
const struct vm_operations_struct generic_file_vm_ops = {
    .fault      = filemap_fault,
    .map_pages  = filemap_map_pages,
    .page_mkwrite   = filemap_page_mkwrite,
};
```
                     
```
+---------------+                                                                            
| filemap_fault | : read ahead (async or sync)
+---|-----------+                                                                            
    |    +---------------+                                                                   
    |--> | find_get_page | find page in mapping                                              
    |    +---------------+                                                                   
    |                                                                                        
    |--> if found                                                                            
    |                                                                                        
    |        +-------------------------+                                                     
    |------> | do_async_mmap_readahead | asynchronously read ahead if the page has such label
    |        +-------------------------+                                                     
    |                                                                                        
    |--> else                                                                                
    |                                                                                        
    |        +------------------------+                                                      
    +------> | do_sync_mmap_readahead | synchronously read ahead anyway                      
             +------------------------+                                                      
```   
  
```
+--------------+                                                     
| do_cow_fault | : alloc a page, set up page table, copy data        
+---|----------+ (what's the difference from do_?)
    |    +------------------+                                        
    |--> | anon_vma_prepare | ensure vma has an anon_vma             
    |    +------------------+                                        
    |    +----------------+                                          
    |--> | alloc_page_vma | alloc a page                             
    |    +----------------+                                          
    |    +------------+                                              
    |--> | __do_fault | ensure 2nd-level table exists, call ->fault()
    |    +------------+                                              
    |    +--------------------+                                      
    |--> | copy_user_highpage | copy page data                       
    |    +--------------------+                                      
    |    +--------------+                                            
    +--> | finish_fault | set up page table                          
         +--------------+                                            
```

```
+-----------------+                                                              
| do_shared_fault | : ensure 2nd-level table exists, call ->fault(), dirty the page
+----|------------+                                                              
     |    +------------+                                                         
     |--> | __do_fault | ensure 2nd-level table exists, call ->fault()           
     |    +------------+                                                         
     |                                                                           
     |--> if ->page_mkwrite() exists                                             
     |                                                                           
     |------> (skip, not our case)                                               
     |                                                                           
     |    +-------------------------+                                            
     +--> | fault_dirty_shared_page | dirty the page                             
          +-------------------------+                                            
```

```
+------------+                                                      
| do_wp_page | : handle the fault of copy-on-write                    
+--|---------+ (what's the difference from do_cow_fault?)
   |    +----------------+                                          
   |--> | vm_normal_page | get struct page of given pte              
   |    +----------------+                                          
   |    +--------------+                                            
   +--> | wp_page_copy | : alloc a page, copy data, add to lru, set pte                          
        +--------------+                                            
```
  
```
+--------------+                                                                         
| wp_page_copy | : alloc a page, copy data, add to lru, set pte                          
+---|----------+                                                                         
    |    +----------------+                                                              
    |--> | alloc_page_vma | alloc a page                                                 
    |    +----------------+                                                              
    |    +---------------+                                                               
    |--> | cow_user_page | copy data from old page                                       
    |    +---------------+                                                               
    |    +---------------------+                                                         
    |--> | pte_offset_map_lock | kmap                                                    
    |    +---------------------+                                                         
    |    +--------+                                                                      
    |--> | mk_pte | prepare pte entry                                                    
    |    +--------+                                                                      
    |    +------------------------+                                                      
    |--> | page_add_new_anon_rmap | label 'swap backed' on page, and save rmap info to it
    |    +------------------------+                                                      
    |    +---------------------------------------+                                       
    |--> | lru_cache_add_inactive_or_unevictable | add to lru cache                      
    |    +---------------------------------------+                                       
    |    +-------------------+                                                           
    |--> | set_pte_at_notify | set pte                                                   
    |    +-------------------+                                                           
    |    +------------------+                                                            
    |--> | page_remove_rmap | page->_mapcount--                                          
    |    +------------------+                                                            
    |    +------------------+                                                            
    +--> | pte_unmap_unlock | kunmap                                                     
         +------------------+                                                            
```
  
```
+--------------+                                                                   
| expand_stack | : expand stack                                                    
+---|----------+                                                                   
    |    +------------------+                                                      
    +--> | expand_downwards | : expand stack (downward version)                    
         +----|-------------+                                                      
              |                                                                    
              |--> ensure the stack guard gap is still valid                       
              |                                                                    
              |    +------------------+                                            
              |--> | anon_vma_prepare | ensure vma has an anon_vma                 
              |    +------------------+                                            
              |                                                                    
              |--> if it's not yet expanded (in case somewhere else did it already)
              |                                                                    
              +------> update vma 'start' and 'pgoff'                              
```
  
```
+-------------------+                                                                        
| __do_kernel_fault | : try to fix exception, if fail, dump info and exit task               
+----|--------------+                                                                        
     |    +-----------------+                                                                
     |--> | fixup_exception | if there's a fixup, let the faulted task continue from there   
     |    +-----------------+                                                                
     |                                                                                       
     |--> return if there's fixup                                                            
     |                                                                                       
     |    +----------+                                                                       
     |--> | show_pte | dump info                                                             
     |    +----------+                                                                       
     |    +-----+                                                                            
     |--> | die | dump info                                                                  
     |    +-----+                                                                            
     |    +---------+                                                                        
     +--> | do_exit | transfer child tasks to reaper, release resources, and yield scheduling
          +---------+                                                                        
```
  
```
+-----------------+                                                                                                     
| fixup_exception | : if there's a fixup, let the faulted task continue from there                                      
+----|------------+                                                                                                     
     |    +-------------------------+                                                                                   
     |--> | search_exception_tables | :                                                                                 
     |    +------|------------------+                                                                                   
     |           |    +-------------------------------+                                                                 
     |           |--> | search_kernel_exception_table | search in exception table, but nothing is compiled in this range
     |           |    +-------------------------------+                                                                 
     |           |                                                                                                      
     |           |--> if not found                                                                                      
     |           |                                                                                                      
     |           |        +-------------------------------+                                                             
     |           |------> | search_kernel_exception_table | search in exception table (probably nothing as well)        
     |           |        +-------------------------------+                                                             
     |           |                                                                                                      
     |           |--> if not found                                                                                      
     |           |                                                                                                      
     |           |        +---------------------+                                                                       
     |           +------> | search_bpf_extables | do nothing bc of disabled config                                      
     |                    +---------------------+                                                                       
     |                                                                                                                  
     |--> if found                                                                                                      
     |                                                                                                                  
     +------> regs->pc = found addr (so it starts from there after returning to svc or usr mode)                        
```
  
</details>
  
## <a name="task-startup"></a> Task Startup
  
Before the main function runs, there are several important steps that are performed when executing a program like a 'Hello World' program from a shell. 
Here is a chronological order of the essential mapping-related steps:

- `fork`: The shell clones itself to create a new process.
  1. Duplicate mm: The memory management structure is duplicated.
  2. Duplicate page tables and vma list: The page tables and virtual memory areas are duplicated from the parent shell.

- `execve`: The new process replaces itself with the target binary (the 'Hello World' program) and prepares for its execution.
  1. Prepare new `mm` and page table: A new memory management structure and page table are created for the new process.
  2. Replace the old `mm` and page table: The old memory management structure and page table are replaced with the new ones.
  3. Prepare stack: The stack for the new process is set up.
  4. Load the main program into memory: The executable code of the main program is loaded into memory.
  5. Prepare heap: The heap region is prepared for dynamic memory allocation.
  6. Load the linker into memory: The linker (e.g., `ld.so` or `ld-linux.so`) is loaded into memory. It is responsible for loading and linking shared libraries.
  7. Prepare special mappings: Special mappings such as `sigpage`, `vvar`, and `vdso` are set up.
  8. Copy arguments and environment variables to the stack: The command-line arguments and environment variables are copied to the stack for access by the program.
  9. Pass control to the linker: Control is transferred to the linker, which performs additional initialization and sets up the program's execution environment.

These steps ensure that the necessary memory mappings and structures are established before the main function of the program starts executing.
After the necessary preparations, the linker (e.g., `ld.so`) loads the libraries specified by the main program and passes control to it. 
The linker's role is vital, and its path is determined during program compilation.

- Linker:
  - Loads libraries.
  - Extends the heap if needed.
  - Passes control to the main program.
- Main:
  - Executes the program's instructions, such as printing "Hello, Za Warudo!".
  
<p align="center"><img src="images/task-memory/task-startup.png" /></p>
  
<details><summary> More Details </summary>
  
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

```
00010000-00011000 r-xp  /home/root/a.out    | 2. [kernel] map 1st segment of a.out


00020000-00021000 r--p  /home/root/a.out    | 3. [kernel] map 2nd segment of a.out     | 14. [linker] mprotect 'r'
00021000-00022000 rw-p  /home/root/a.out    |


000ac000-000cd000 rw-p  [heap]              | 16. [linker] extend heap size from 0

76db8000-76f11000 r-xp  /lib/libc.so.6      | 8. [linker] map file (libc, rx)
                                            |
76f11000-76f20000 ---p  /lib/libc.so.6      |      | 9. [linker] mprotect 'none'
                                            |
76f20000-76f22000 r--p  /lib/libc.so.6      |      | 10. [linker] map file (libc, rw)  | 13. [linker] mprotect 'r'
76f22000-76f24000 rw-p  /lib/libc.so.6      |      |
                                            |
76f24000-76f2d000 rw-p                      |      | 11. [linker] map anon (rw)

76f30000-76f56000 r-xp  /lib/ld-linux.so.3  | 4. [kernel] map 1st segment of linker

76f64000-76f66000 rw-p                      | 7. [linker] map anon (rw)                | 12. [linker] set tls, tid, ...

76f66000-76f67000 r--p  /lib/ld-linux.so.3  | 5. [kernel] map 2nd segment of linker    | 15. [linker] mprotect 'r'
76f67000-76f68000 rw-p  /lib/ld-linux.so.3  |

7e99e000-7e9bf000 rw-p  [stack]             | 1. [kernel] prepare stack

7eb92000-7eb93000 r-xp  [sigpage]           | 6. [kernel] prepare sigpage, vvar, and vdso
7eb93000-7eb94000 r--p  [vvar]              |
7eb94000-7eb95000 r-xp  [vdso]              |

ffff0000-ffff1000 r-xp  [vectors]           | 0. [kernel] it's already there during boot time
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

```
+-----------------+                                                                                      
| load_elf_binary | : replace mm, map exec and interp, copy ang/env to stack, set regs->pc to interp           
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
     |--> | begin_new_exec | ensure no other threads, replace mm, clone structures, set task name, reset sig handlers
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
     |--> | ARCH_SETUP_ADDITIONAL_PAGES | 
     |    +-------+---------------------+  
     |            |    +-----------------------------+
     |            +--> | arch_setup_additional_pages | install three vmas for 'sigpage', 'vvar', and 'vdso'
     |                 +-----------------------------+
     |    +-------------------+                                                                          
     |--> | create_elf_tables | copy elf info, arg, env to user space stack                              
     |    +-------------------+                                                                          
     |    +--------------+                                                                               
     +--> | START_THREAD |  set pc and sp of 'regs'                                                      
          +--------------+                                                                               
```

```
+----------------+
| begin_new_exec | : ensure no other threads, replace mm, clone structures, set task name, reset sig handlers
+---|------------+
    |    +-----------+
    +--> | de_thread | ensure no other threads in group, and arg task assumes the leader
    |    +-----------+
    |    +---------------+
    |--> | unshare_files | ensure the fdtable isn't shared
    |    +---------------+
    |        |    +------------+
    |        +--> | unshare_fd | clone fdtable from current task, and assign to current task?
    |             +------------+
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
+-----------+                                                                               
| de_thread | : ensure no other threads in group, and arg task assumes the leader             
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
| unshare_fd | : clone fdtable from current task, and assign to current task?                    
+--|---------+                                                                                 
   |                                                                                           
   |--> if flash has specified 'clone'                                                         
   |                                                                                           
   |        +--------+                                                                         
   +------> | dup_fd | : clone fdtable from current task (both fd arrays point to the same files)
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
+-----------------+                                                            
| load_elf_interp | : map interpreter segments                                   
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
  
```
+-----------------------------+                                                     
| arch_setup_additional_pages | : install three vmas for 'sigpage', 'vvar', and 'vdso'
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
        +--> | arm_install_vdso | :                                                   
             +----|-------------+                                                   
                  |    +--------------+                                             
                  |--> | install_vvar | install vma for 'vvar'                      
                  |    +--------------+                                             
                  |    +--------------------------+                                 
                  +--> | _install_special_mapping | install vma for 'vdso'          
                       +--------------------------+                                 
```
   
</details>
  
## <a name="others"></a> Others

### Reverse Mapping

The process of page unmapping utilizes the constructed structures. It may be appropriate to relocate this topic to the 'Page Cache' chapter.
  
<details><summary> More Details </summary>
  
```c
struct page {
    union {
        atomic_t _mapcount;     // indicates how many page tables reference to this page
                                // -1: unused, 0: one owner, n: additional users
        unsigned int page_type;
        unsigned int active;
        int units;
    };
}
```
  
```
+------------------------+                                                        
| page_add_new_anon_rmap | : label 'swap backed' on page, and save rmap info to it
+-----|------------------+                                                        
      |                                                                           
      |--> label 'swap backed' on page                                            
      |                                                                           
      |    +----------------------+                                               
      +--> | __page_set_anon_rmap | save rmap info in page                        
           +----------------------+                                               
```
  
```
+----------------------+                         
| __page_set_anon_rmap | : save rmap info in page
+-----|----------------+                         
      |                                          
      |--> if page isn't exclusive to this vma   
      |                                          
      |------> choose the oldest anon_vma instead
      |                                          
      |--> page->mapping = anon_vma              
      |                                          
      +--> page->index = offset (unit: page)     
```
  
```
+--------------------+                                                  
| page_add_anon_rmap | : page->_mapcount++                              
+----|---------------+                                                  
     |    +-----------------------+                                     
     +--> | do_page_add_anon_rmap | : page->_mapcount++                 
          +-----|-----------------+                                     
                |                                                       
                |--> page->_mapcount++                                  
                |                                                       
                |--> if it's the first owner                            
                |                                                       
                |        +----------------------+                       
                |------> | __page_set_anon_rmap | save rmap info in page
                |        +----------------------+                       
                |                                                       
                |--> else                                               
                |                                                       
                |        +------------------------+                     
                +------> | __page_check_anon_rmap | only sanity checks  
                         +------------------------+                     
```
  
```
+--------------------+                  
| page_add_file_rmap | page->_mapcount++
+--------------------+                  
```
  
```
+-----------------+                                                 
| page_referenced | : determine how actively the page is used
+----|------------+                                                 
     |                                                              
     |--> set up rmap_walk_control                                  
     |                                                              
     |    +-----------+                                             
     +--> | rmap_walk | determine how actively the page is used
          +-----------+                                             
```
  
```
+---------------------+                                                           
| page_referenced_one | : check if the page is actively referenced in the vma     
+-----|---------------+                                                           
      |    +----------------------+                                               
      +--> | page_vma_mapped_walk | : return true if the page is mapped to the vma
           +-----|----------------+                                               
                 |                                                                
                 |--> get mm, addr, and page from arg                             
                 |                                                                
                 |--> look up target pte by mm and addr                           
                 |                                                                
                 |    +-----------+                                               
                 +--> | check_pte | check if pfn of pte == pfn of page            
                      +-----------+                                               
```
  
```
+-----------+                                                                               
| rmap_walk | : for each vma related to the give page, apply the specified action
+--|--------+   e.g., clear pte in each mapping
   |                                                                                        
   |--> if page is anonymous mapping                                                        
   |                                                                                        
   |        +----------------+                                                              
   |------> | rmap_walk_anon | for each avc of the given page, clear the pte in each mapping
   |        +----------------+                                                              
   |                                                                                        
   |--> else (file mapping)                                                                 
   |                                                                                        
   |        +----------------+                                                              
   +------> | rmap_walk_file | for each vma refer to this mapping, walk vma and clear pte   
            +----------------+                                                              
```

```
+----------------+                                                                
| rmap_walk_anon | ： for each avc of the given page, apply the callback,
+---|------------+   e.g., clear the pte in each mapping         
    |    +---------------+                                                        
    |--> | page_anon_vma | get anon_vma of the given page                         
    |    +---------------+                                                        
    |                                                                             
    |--> for each avc within the range                                            
    |                                                                             
    |------> call ->rmap_one(), e.g.,                                             
    |        +------------------+                                                 
    |        | try_to_unmap_one | walk vma and clear pte                          
    |        +------------------+                                                 
    |                                                                             
    +------> if ->done() exists, call it, e.g.,                                   
             +-----------------+                                                  
             | page_not_mapped | check if page isn't mapped anymore               
             +-----------------+                                                  
```

```
+----------------+                                                             
| rmap_walk_file | ： for each vma refer to this mapping, walk vma and apply the callback,
+---|------------+   e.g., clear pte
    |                                                                          
    |--> get mapping from the given page                                       
    |                                                                          
    |--> for each vma refer to this mapping                                    
    |                                                                          
    |------> call ->rmap_one(), e.g.,                                          
    |        +------------------+                                              
    |------> | try_to_unmap_one | walk vma and clear pte                       
    |        +------------------+                                              
    |                                                                          
    |------> if ->done() exists, call it, e.g.,                                
    |        +-----------------+                                               
    +------> | page_not_mapped | check if page isn't mapped anymore            
             +-----------------+                                               
```
  
```
+------------------+                                     
| page_remove_rmap | : page->_mapcount--                 
+----|-------------+                                     
     |                                                   
     |--> if it's not a anonymous age                    
     |                                                   
     |        +-----------------------+                  
     |------> | page_remove_file_rmap | page->_mapcount--
     |        +-----------------------+                  
     |                                                   
     |        return                                     
     |                                                   
     +--> page->_mapcount--                              
```
  
```
+---------------+                                                               
| anon_vma_fork | : ensure new vma has anon_vma and avc (either old or new)     
+---|-----------+                                                               
    |                                                                           
    |--> return if parent vma has no anon_vma                                   
    |                                                                           
    |    +----------------+                                                     
    |--> | anon_vma_clone | for each avc in src vma: prepare new avc for dst vma
    |    +----------------+                                                     
    |                                                                           
    |--> return if existing anon_vma is reused                                  
    |                                                                           
    |--> alloc anon_vma nad avc                                                 
    |                                                                           
    +--> relate them and dst vma                                                
```
  
```
+----------------+                                                            
| anon_vma_clone | : for each avc in src vma: prepare new avc for dst vma     
+---|------------+                                                            
    |                                                                         
    |--> for each avc in src vma (in reverse direction)                       
    |                                                                         
    |        +----------------------+                                         
    |------> | anon_vma_chain_alloc | alloc avc                               
    |        +----------------------+                                         
    |        +---------------------+                                          
    |------> | anon_vma_chain_link | add avc to dst vma list and anon_vma tree
    |        +---------------------+                                          
    |                                                                         
    +------> try to have dst vma reuse anon_vma                               
```
  
```
+------------------+                                                                                     
| anon_vma_prepare | : ensure vma has an anon_vma                                                        
+----|-------------+                                                                                     
     |                                                                                                   
     |--> return if vma has anon_vma                                                                     
     |                                                                                                   
     |    +--------------------+                                                                         
     +--> | __anon_vma_prepare | : prepare avc, link to vma list and anon_vma tree                       
          +----|---------------+                                                                         
               |    +----------------------+                                                             
               |--> | anon_vma_chain_alloc | alloc an avc                                                
               |    +----------------------+                                                             
               |    +-------------------------+                                                          
               |--> | find_mergeable_anon_vma | check if vma can share anon_vma with its prev or next vma
               |    +-------------------------+                                                          
               |                                                                                         
               |--> if not found                                                                         
               |                                                                                         
               |        +----------------+                                                               
               |------> | anon_vma_alloc |                                                               
               |        +----------------+                                                               
               |                                                                                         
               |--> if vma doesn't have an anon_vma yet                                                  
               |                                                                                         
               |        +---------------------+                                                          
               +------> | anon_vma_chain_link | add avc to vma list and anon_vma tree                    
                        +---------------------+                                                          
```
  
</details>
  
### Kmap
  
In our study case, the highmem zone is not populated with any pages, although in many scenarios, it is intended for high memory utilization.

<details><summary> More Details </summary>

```
+------+                                                             
| kmap | : ensure the page has a mapped virtual address                
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
| kmap_high | : map a highmem page                                         
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
| kunmap | :
+-|------+                                                               
  |                                                                      
  |--> if arg page isn't from high mem, return                           
  |                                                                      
  |    +-------------+                                                   
  +--> | kunmap_high | :
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

- W. Mauerer, Professional Linux Kernel Architecture
- [World Class, DIO ZA WARUDO compilation (JoJo's Bizarre Adventure:Stardust Crusaders)](https://www.youtube.com/watch?v=DefXS17jZwE&ab_channel=WorldClass)
