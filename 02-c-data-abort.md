## Index

- [Introduction](#introduction)
- [Reference](#reference)


## <a name="introduction"></a> Introduction

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
                  +----------+                                                                           
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

## <a name="reference"></a> Reference

(TBD)



