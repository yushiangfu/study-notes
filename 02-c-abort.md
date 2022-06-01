## Index

- [Introduction](#introduction)
- [Fault](#fault)
- [Reference](#reference)


## <a name="introduction"></a> Introduction

Dereferencing the NULL pointer or accessing other invalid addresses are the most common reason causing the tasks to crash. 
The path to crash is as below steps:

1. Accessing an invalid address
2. CPU exception (abort) happens
3. Kernel finds out it's a genuine fault and sends a signal to the task for termination.

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
 Prefetch     |                  | (refer to FSR)
  Abort  -----+                  |                                                      
              +-  from kernel  --|                             1. copy on write         
                                 |                             2. valid fault: anonymous
                                 +-------- handle page fault   3. valid fault: file     
                                 |                             4. write protect?        
              +-  from user    --|                             5. genuine fault            
  Data        |                  |                                                      
  Abort  -----+                  | (refer to IFSR)                                                    
              +-  from kernel  --+                                                      
```

Based on the index fetched from FSR or IFSR, the corresponding function, mainly **do_page_fault**, will be called to handle the fault correctly.

```
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

<details>
  <summary> Code trace </summary>

```
 +-------------+                                                                       
 | vector_pabt |                                                                       
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
| __pabt_usr | get ifsr and handle fault accordingly, return to user mode                  
+--|---------+                                                                             
   |    +-----------+                                                                      
   |--> | usr_entry | store r0 ~ r12, sp_usr, lr_usr, old pc to sp_svc                     
   |    +-----------+                                                                      
   |    +-------------+                                                                    
   |--> | pabt_helper |                                                                    
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
| v6_pabort | get ifsr and handle the fault accordingly      
+--|--------+                                                
   |                                                         
   |--> get ifsr from coprocessor                            
   |                                                         
   |    +------------------+                                 
   +--> | do_PrefetchAbort |                                 
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
| __pabt_svc | get ifsr and handle fault accordingly                                      
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
 | vector_dabt |                                                                      
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
| __dabt_usr | get fsr and handle fault accordingly, return to user mode                   
+--|---------+                                                                             
   |    +-----------+                                                                      
   |--> | usr_entry | store r0 ~ r12, sp_usr, lr_usr, old pc to sp_svc                     
   |    +-----------+                                                                      
   |    +-------------+                                                                    
   |--> | dabt_helper |                                                                    
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
| v6_early_abort | get fsr and handle the fault accordingly
+---|------------+                                         
    |                                                      
    |--> get 'fsr' and 'far' from coprocessor              
    |                                                      
    |    +--------------+                                  
    +--> | do_DataAbort |                                  
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
| __dabt_svc | get fsr and handle fault accordingly                                
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

## <a name="fault"></a> Fault

### Copy on Write

When a task forks a child, the kernel duplicates **mm**-related structures, including page tables, so the child task has its own. 
However, the family still shares the page frames mostly since **fork** is usually followed by **execve**, making the page copy seem redundant. 
If either parent or child tries to write data into any of these pages, the action will be trapped, and a duplicated page will be prepared accordingly.

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

### Fault on Anonymous Area

When an anonymous area is accessed while its page entry isn't ready yet, the fault happens and is trapped by the processor. 
The kernel first attempts to look up if that address is a 'valid' fault or not by checking whether it lies in any existent VMA. 
If that VMA is of an anonymous type, such as heap memory, then the kernel allocates a page frame and fills the corresponding page table entry. 
Everything is ready, and the flow goes back to the logic following the initial access.

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

<details>
  <summary> Code trace </summary>

```
+---------------+                                                                                                                              
| do_page_fault |                                                                                                                              
+---|-----------+                                                                                                                              
    |    +-----------------+                                                                                                                   
    |--> | __do_page_fault |                                                                                                                   
    |    +----|------------+                                                                                                                   
    |         |    +----------+                                                                                                                
    |         |--> | find_vma | try to find target vma based on faulted addr                                                                   
    |         |    +----------+                                                                                                                
    |         |                                                                                                                                
    |         |--> check if it's a valid fault (covered by vma, or lies in stack)                                                              
    |         |                                                                                                                                
    |         |--> return error if not                                                                                                         
    |         |                                                                                                                                
    |         |    +-----------------+                                                                                                         
    |         +--> | handle_mm_fault |                                                                                                         
    |              +----|------------+                                                                                                         
    |                   |    +---------------------+                                                                                           
    |                   |--> | __set_current_state | set state = running                                                                       
    |                   |    +---------------------+                                                                                           
    |                   |    +-------------------+                                                                                             
    |                   +--> | __handle_mm_fault |                                                                                             
    |                        +----|--------------+                                                                                             
    |                             |    +------------+                                                                                          
    |                             |--> | pgd_offset | get target pgd based on addr                                                             
    |                             |    +------------+                                                                                          
    |                             |    +-----------+                                                                                           
    |                             |--> | p4d_alloc | return arg pgd                                                                            
    |                             |    +-----------+                                                                                           
    |                             |    +-----------+                                                                                           
    |                             |--> | pud_alloc | return arg p4d                                                                            
    |                             |    +-----------+                                                                                           
    |                             |    +-----------+                                                                                           
    |                             |--> | pmd_alloc |  return arg pud                                                                           
    |                             |    +-----------+                                                                                           
    |                             |    +------------------+                                                                                    
    |                             +--> | handle_pte_fault | ensure 2nd-level table exists, and either call ->fault() or simply update pte entry
    |                                  +------------------+                                                                                    
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
    +------> | __do_kernel_fault | die                                                                                                         
             +-------------------+                                                                                                             
```

```
 +------------------+                                                                                     
 | handle_pte_fault | ensure 2nd-level table exists, and either call ->fault() or simply update pte entry 
 +----|-------------+                                                                                     
      |                                                                                                   
      |--> if no pet yet                                                                                  
      |                                                                                                   
      |------> if vma is anon type                                                                        
      |                                                                                                   
      |            +-------------------+                                                                  
      |----------> | do_anonymous_page | ensure 2nd-level table exists, prepare pte value and update entry
      |            +-------------------+                                                                  
      |                                                                                                   
      |------> else                                                                                       
      |                                                                                                   
      |            +----------+                                                                           
      +----------> | do_fault | ensure 2nd-level table exists, call ->fault()                             
      |            +----------+                                                                           
      |                                                                                                   
      |--> if flag has specified 'write'                                                                  
      |                                                                                                   
      |        +------------+                                                                             
      +------> | do_wp_page | handle the fault of write-protect?
               +------------+                                                                             
```

```
+-------------------+                                                                  
| do_anonymous_page | ensure 2nd-level table exists, prepare pte value and update entry
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
     |--> | page_add_new_anon_rmap | (skip for now)                                    
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
| do_fault | ensure 2nd-level table exists, call ->fault()                                     
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
| do_read_fault | ensure 2nd-level table exists, call ->fault()
+---|-----------+                                              
    |                                                          
    |--> if vm ops has ->map_pages()                           
    |                                                          
    |        +-----------------+                               
    |------> | do_fault_around |                               
    |        +----|------------+                               
    |             |                                            
    |             |--> determine start and end page offset     
    |             |                                            
    |             +--> call ->map_pages(), e.g.,               
    |                  +-------------------+                   
    |                  | filemap_map_pages |                   
    |                  +-------------------+                   
    |    +------------+                                        
    +--> | __do_fault |                                        
         +--|---------+                                        
            |                                                  
            |--> ensure 2nd-level table exists                 
            |                                                  
            +--> call ->fault(), e.g.,                         
                 +---------------+                             
                 | filemap_fault |                             
                 +---------------+                             
```

```
+-----------------+                                                              
| do_shared_fault | ensure 2nd-level table exists, call ->fault(), dirty the page
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
| do_wp_page | handle the fault of copy-on-write                    
+--|---------+                                                      
   |    +----------------+                                          
   |--> | vm_normal_page | get struct page of give pte              
   |    +----------------+                                          
   |    +--------------+                                            
   +--> | wp_page_copy |                                            
        +---|----------+                                            
            |                                                       
            |--> allocate a page                                    
            |                                                       
            |    +---------------+                                  
            |--> | cow_user_page | copy data from src to dst page   
            |    +---------------+                                  
            |                                                       
            |    +------------------------+                         
            +--> | page_add_new_anon_rmap | set up anon rmap of page
                 +------------------------+                         
```
  
</details>

## <a name="reference"></a> Reference

(TBD)



