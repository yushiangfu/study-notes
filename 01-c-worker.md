## Index

- [Introduction](#introduction)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

```
+----------------------+                                            
| workqueue_init_early |                                            
+-----|----------------+                                            
      |                                                             
      |--> create kmem cache for pool workqueue (pwq)               
      |                                                             
      |--> for each cpu                                             
      |                                                             
      |------> for each worker pool (n = 2)                         
      |                                                             
      |----------> initialize                                       
      |                                                             
      |    +-----------------+                                      
      +--> | alloc_workqueue | 'system_wq'                          
           +-----------------+                                      
           +-----------------+                                      
           | alloc_workqueue | 'system_highpri_wq'                  
           +-----------------+                                      
           +-----------------+                                      
           | alloc_workqueue | 'system_long_wq'                     
           +-----------------+                                      
           +-----------------+                                      
           | alloc_workqueue | 'system_unbound_wq'                  
           +-----------------+                                      
           +-----------------+                                      
           | alloc_workqueue | 'system_freezable_wq'                
           +-----------------+                                      
           +-----------------+                                      
           | alloc_workqueue | 'system_power_efficient_wq'          
           +-----------------+                                      
           +-----------------+                                      
           | alloc_workqueue | 'system_freezable_power_efficient_wq'
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

## <a name="reference"></a> Reference

(TBD)




