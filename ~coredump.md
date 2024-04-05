```
fs/coredump.c                                                                        
+-------------+                                                                       
| do_coredump | : given core_pattern, setup and call usermode helper, call ->core_dump
+-|-----------+                                                                       
  |    +---------------+                                                              
  |--> | coredump_wait | kill other thtreads, wait for them to be inactive (?)        
  |    +---------------+                                                              
  |    +-----------------+                                                            
  |--> | format_corename | alloc argv, fill data to cn and argv                       
  |    +-----------------+                                                            
  |                                                                                   
  |--> if ispipe (our case)                                                           
  |    |                                                                              
  |    |--> alloc helper argv                                                         
  |    |                                                                              
  |    |--> setup helper argv from previous cn and argv                               
  |    |                                                                              
  |    |    +---------------------------+                                             
  |    |--> | call_usermodehelper_setup |                                             
  |    |    +---------------------------+                                             
  |    |    +--------------------------+                                              
  |    +--> | call_usermodehelper_exec | (e.g., systemd-coredump)                     
  |         +--------------------------+                                              
  |                                                                                   
  |--> else                                                                           
  |    -                                                                              
  |    +--> (skip)                                                                    
  |                                                                                   
  |    +---------------+                                                              
  |--> | unshare_files |                                                              
  |    +---------------+                                                              
  |    +-------------------+                                                          
  |--> | dump_vma_snapshot | traverse vma list and save info in cprm                  
  |    +-------------------+                                                          
  |                                                                                   
  |--> call ->core_dump, e.g.,                                                        
  |    +---------------+                                                              
  |    | elf_core_dump |                                                              
  |    +---------------+                                                              
  |                                                                                   
  +--> if is pipe                                                                     
       |                                                                              
       |    +-----------------------+                                                 
       +--> | wait_for_dump_helpers | wait for pipe reader                            
            +-----------------------+                                                 
```

```
fs/coredump.c                                                                                          
+---------------+                                                                                       
| coredump_wait | : kill other thtreads, wait for them to be inactive (?)                               
+-|-------------+                                                                                       
  |                                                                                                     
  |--> setup core_state                                                                                 
  |                                                                                                     
  |    +-------------+                                                                                  
  |--> | zap_threads | setup task signal fields, kill other threads, clear 'sigpending', set 'dump core'
  |    +-------------+                                                                                  
  |                                                                                                     
  +--> if at least one thread got killed                                                                
       |                                                                                                
       |    +---------------------------+                                                               
       |--> | wait_for_completion_state | wait for what?                                                
       |    +---------------------------+                                                               
       |                                                                                                
       +--> wait for other threads to be inactive                                                       
```

```
fs/coredump.c                                                                                                                
+-------------+                                                                                                               
| zap_threads | : setup task signal fields, kill other threads, clear 'sigpending', set 'dump core'                           
+-|-----------+                                                                                                               
  |                                                                                                                           
  +--> if it's not group exit                                                                                                 
       |                                                                                                                      
       |                                                                                                                      
       |--> save core_state in signal                                                                                         
       |                                                                                                                      
       |    +-------------+                                                                                                   
       |--> | zap_process | setup current task signal fields, send 'kill' to other thread of signal, return # of killed thread
       |    +-------------+                                                                                                   
       |    +-----------------------+                                                                                         
       |--> | clear_tsk_thread_flag | clear flag 'sigpending'                                                                 
       |    +-----------------------+                                                                                         
       |                                                                                                                      
       +--> set flag 'dump core'                                                                                              
```

```
fs/coredump.c                                                                                                      
+-------------+                                                                                                     
| zap_process | : setup current task signal fields, send 'kill' to other thread of signal, return # of killed thread
+-|-----------+                                                                                                     
  |                                                                                                                 
  |--> setup task signal fields                                                                                     
  |                                                                                                                 
  +--> for each thread of the signal                                                                                
       |                                                                                                            
       |    +---------------------------+                                                                           
       +--> | task_clear_jobctl_pending | clear job ctrl pending bits                                               
       |    +---------------------------+                                                                           
       |                                                                                                            
       +--> if thread isn't current task                                                                            
            |                                                                                                       
            |    +-----------+                                                                                      
            |--> | sigaddset | add 'kill' signal                                                                    
            |    +-----------+                                                                                      
            |    +----------------+                                                                                 
            +--> | signal_wake_up |  wake up thread to face the kill                                                
                 +----------------+                                                                                 
```

```
fs/coredump.c                                                                                             
+-----------------+                                                                                        
| format_corename | : alloc argv, fill data to argv and cn                                                 
+-|---------------+                                                                                        
  |                                                                                                        
  |--> given core_pattern, check if it's pipe (true in our case)                                           
  |                                                                                                        
  |--> alloc argv                                                                                          
  |                                                                                                        
  +--> fill data to argv and cn                                                                            
                                                                                                           
                                                                                                           
                                                                                                           
                                                                                                           
                  |/usr/lib/systemd/systemd-coredump %P %u %g %s %t %c %h                                  
                                                                                                           
                  |+--------------------------------  |  |  |  |  |  |  -                                  
                  ||                                  |  |  |  |  |  |  +-> hostname                       
                  ||                                  |  |  |  |  |  +----> cpu the task ran on            
                  ||                                  |  |  |  |  +-------> time of coredump               
                  ||                                  |  |  |  +----------> signal that caused the coredump
                  ||                                  |  |  +-------------> gid                            
                  ||                                  |  +----------------> uid                            
                  ||                                  +-------------------> global pid                     
                  |+------------------------------------------------------> core name                      
                  +-------------------------------------------------------> pipe                           
```
