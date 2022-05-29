## Index

- [Introduction](#introduction)
- [Reference](#reference)


## <a name="introduction"></a> Introduction

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



