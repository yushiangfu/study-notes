## Index

- [Introduction](#introduction)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

```
+--------------+                                                  +----------------------+
| start_kernel |                                                  | kernel_init_freeable |
+---|----------+                                                  +-----|----------------+
    |    +----------------------+                                       |    +----------------+
    +--> | workqueue_init_early |                                       +--> | workqueue_init |
         +----------------------+                                            +---|------------+
               |                                                                 |
               |--> create kmem cache for pool workqueue (pwq)                   |--> for each cpu
               |                                                                 |
               |--> for each cpu                                                 |------> for each worker pool (2)
               |                                                                 |
               |------> for each worker pool (n = 2)                             |            +---------------+
               |                                                                 |----------> | create_worker |
               |----------> initialize                                           |            +---------------+
               |                                                                 |
               |    +-----------------+                                          |--> for each unbound pool
               |--> | alloc_workqueue | 'system_wq'                              |
               |    +-----------------+                                          |        +---------------+
               |    +-----------------+                                          |------> | create_worker |
               |--> | alloc_workqueue | 'system_highpri_wq'                      |        +---------------+
               |    +-----------------+                                          |    +------------------+
               |    +-----------------+                                          +--> | wq_watchdog_init |
               |--> | alloc_workqueue | 'system_long_wq'                              +------------------+
               |    +-----------------+
               |    +-----------------+
               |--> | alloc_workqueue | 'system_unbound_wq'
               |    +-----------------+
               |    +-----------------+
               |--> | alloc_workqueue | 'system_freezable_wq'
               |    +-----------------+
               |    +-----------------+
               |--> | alloc_workqueue | 'system_power_efficient_wq'
               |    +-----------------+
               |    +-----------------+
               +--> | alloc_workqueue | 'system_freezable_power_efficient_wq'
                    +-----------------+                                 
```

```
+-----------------+                                                         
| alloc_workqueue |                                                         
+----|------------+                                                         
     |                                                                      
     |--> allocate and set up 'wq'                                          
     |                                                                      
     |    +---------------------+                                           
     +--> | alloc_and_link_pwqs | prepare 'pwq' of each cpu and link to 'wq'
          +---------------------+                                           
```

```
+---------------+                                                   
| create_worker |                                                   
+---|-----------+                                                   
    |    +-----------+                                              
    |--> | ida_alloc | determine worker id                          
    |    +-----------+                                              
    |    +--------------+                                           
    |--> | alloc_worker | alocate 'worker', and it's not a kthread  
    |    +--------------+                                           
    |    +------------------------+                                 
    |--> | kthread_create_on_node | create 'worker' kthread         
    |    +------------------------+                                 
    |    +-----------------------+                                  
    |--> | worker_attach_to_pool | attach the 'worker' to arg 'pool'
    |    +-----------------------+                                  
    |    +-----------------+                                        
    +--> | wake_up_process | wake up the worker                     
         +-----------------+                                        
```

```
      +---------------+                                                           
      | worker_thread |                                                           
      +---|-----------+                                                           
          |    +---------------+                                                  
          |--> | set_pf_worker | set WORKER flag on worker task                   
          |    +---------------+                                                  
 woke_up: |                                                                       
          |--> while we haven't finished all the work                             
          |                                                                       
          |------> get the first work from list                                   
          |                                                                       
          |        +------------------+                                           
          |------> | process_one_work | remove work from list and execute ->func()
          |        +------------------+                                           
          |                                                                       
          |--> enter idle state                                                   
          |                                                                       
          |    +----------+                                                       
          |--> | schedule | yield the cpu and sleep                               
          |    +----------+                                                       
          |                                                                       
          +--> go to 'woke_up'                                                    
```

```
+------------------+                                                                                                                 
| mod_delayed_work | : steal a work, and either add to a pool or to a timer                                                          
+----|-------------+                                                                                                                 
     |    +---------------------+                                                                                                    
     +--> | mod_delayed_work_on |                                                                                                    
          +-----|---------------+                                                                                                    
                |    +---------------------+                                                                                         
                |--> | try_to_grab_pending | unlink the work and label it 'pending'                                                  
                |    +---------------------+                                                                                         
                |    +----------------------+                                                                                        
                +--> | __queue_delayed_work | : queue work or set a timer for dwork                                                  
                     +-----|----------------+                                                                                        
                           |                                                                                                         
                           |--> if no delay                                                                                          
                           |                                                                                                         
                           |        +--------------+                                                                                 
                           |------> | __queue_work | : queue work to the appropriate pool                                            
                           |        +---|----------+                                                                                 
                           |            |                                                                                            
                           |            |--> determine pwq                                                                           
                           |            |                                                                                            
                           |            |--> determine which list to insert: active (likely) or inactive (pool has reached max usage)
                           |            |                                                                                            
                           |            |    +-------------+                                                                         
                           |            +--> | insert_work | queue work to that specified list                                       
                           |                 +-------------+                                                                         
                           |                                                                                                         
                           |------> return                                                                                           
                           |                                                                                                         
                           |    +-----------+                                                                                        
                           +--> | add_timer | add dwork timer                                                                        
                                +-----------+                                                                                        
```

```
+---------------------+                                                                 
| try_to_grab_pending | : unlink the work and label it 'pending'                        
+-----|---------------+                                                                 
      |                                                                                 
      |--> if it's a dwork                                                              
      |                                                                                 
      |        +-----------+                                                            
      |------> | del_timer | deactivate the dwork timer                                 
      |        +-----------+                                                            
      |                                                                                 
      |------> return 1 if the timer is active                                          
      |    (reaching here means it's not a dwork or an inactive dwork)                  
      |    +---------------+                                                            
      |--> | get_work_pool | get the worker pool give the work                          
      |    +---------------+                                                            
      |                                                                                 
      |--> if the work is inactive                                                      
      |                                                                                 
      |        +----------------------------+                                           
      |------> | pwq_activate_inactive_work | move work to pool and clear 'inactive' bit
      |        +----------------------------+                                           
      |                                                                                 
      |--> remove work from list                                                        
      |                                                                                 
      |    +--------------------------------+                                           
      +--> | set_work_pool_and_keep_pending | label the work 'pending'                  
           +--------------------------------+                                           
```

## <a name="reference"></a> Reference

(TBD)




