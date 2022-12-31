> Study case: Linux version 5.15.0 on OpenBMC

## Index

- [Introduction](#introduction)
- [Framework](#framework)
- [System Startup](#system-startup)
- [Cheat Sheet](#cheat-sheet)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

(TBD)

## <a name="framework"></a> Framework

```
static const struct file_operations watchdog_fops = {
    .owner      = THIS_MODULE,
    .write      = watchdog_write,
    .unlocked_ioctl = watchdog_ioctl,
    .compat_ioctl   = compat_ptr_ioctl,
    .open       = watchdog_open,
    .release    = watchdog_release,
};
```

```
drivers/watchdog/watchdog_dev.c                                         
+----------------+                                                       
| watchdog_write | ： feed watchdog, start or cancel timer accordingly    
+-|--------------+                                                       
  |                                                                      
  |--> check if there's magic char 'V' in user buffer                    
  |                                                                      
  |    +---------------+                                                 
  +--> | watchdog_ping | feed watchdog, start or cancel timer accordingly
       +---------------+                                                 
```

```
drivers/watchdog/watchdog_dev.c                                        
+----------------+                                                      
| watchdog_ioctl | : watchdog ioctl                                     
+-|--------------+                                                      
  |                                                                     
  |--> switch cmd                                                       
  +--> case get_support                                                 
  |    -                                                                
  |    +--> copy wdd info to user                                       
  |                                                                     
  +--> case get_status                                                  
  |    -                                                                
  |    +--> prepare status and copy to user                             
  |                                                                     
  +--> case get_boot_status                                             
  |    -                                                                
  |    +--> copy boot status to user                                    
  |                                                                     
  +--> case set_options                                                 
  |    -                                                                
  |    +--> get option from user, disable or enable watchdog accordingly
  |                                                                     
  +--> case keep_alive                                                  
  |    -                                                                
  |    +--> ping watchdog                                               
  |                                                                     
  +--> case set_timeout                                                 
  |    -                                                                
  |    +--> set watchdog timeout, ping it                               
  |                                                                     
  +--> case get_timeout                                                 
  |    -                                                                
  |    +--> copy timeout to user                                        
  |                                                                     
  +--> case get_timeleft                                                
  |    -                                                                
  |    +--> copy timeleft (not supported in our case)                   
  |                                                                     
  +--> case set_pretimeout                                              
  |    -                                                                
  |    +--> set pretimeout (probably not supported in our case)         
  |                                                                     
  +--> case get_pretimeout                                              
       -                                                                
       +--> get pretiemout                                              
```

```
drivers/watchdog/watchdog_dev.c                                           
+---------------+                                                          
| watchdog_open | : ensure watchdog is active, save wd_data in file private
+-|-------------+                                                          
  |                                                                        
  |--> get watchdog_data (wd_data) from inode                              
  |                                                                        
  |--> if it's opened already, return 'busy' error                         
  |                                                                        
  |    +----------------+                                                  
  |--> | watchdog_start | ensure watchdog is active                        
  |    +----------------+                                                  
  |                                                                        
  |--> save wd_data in file_private                                        
  |                                                                        
  |    +-------------+                                                     
  +--> | stream_open | ensure file isn't seekable                          
       +-------------+                                                     
```

```
drivers/watchdog/watchdog_dev.c                                                
+----------------+                                                              
| watchdog_start | : ensure watchdog is active                                  
+-|--------------+                                                              
  |                                                                             
  |--> if it's already active, return                                           
  |                                                                             
  |--> label 'keep_alive' on watchdog_data (wd_data)                            
  |                                                                             
  |--> if watchdog hw is running && ->ping() exists                             
  |    |                                                                        
  |    |    +-----------------+                                                 
  |    |--> | __watchdog_ping | feed watchdog, start or cancel timer accordingly
  |    |    +-----------------+                                                 
  |    |                                                                        
  |    +--> label 'active' on watchdog_dev (wdd)                                
  |                                                                             
  +--> else                                                                     
       -                                                                        
       +--> label 'active' on watchdog_dev (wdd)                                
       |                                                                        
       |--> call ->start(), e.g.,                                               
       |    +------------------+                                                
       |    | aspeed_wdt_start | write hw reg to start watchdog                 
       |    +------------------+                                                
       |                                                                        
       |    +------------------------+                                          
       +--> | watchdog_update_worker | start or cancel timer accordingly        
            +------------------------+                                          
```

## <a name="system-startup"></a> System Startup

```
aspeed_wdt_init: register 'aspeed_watchdog_driver' to 'platform' bus
  aspeed_wdt_probe: map registers, install ops, register watchdog_dev
watchdog_init: create 'watchdogd', register class, alloc dev#, register each watchdog_dev on list
```

```
drivers/watchdog/aspeed_wdt.c
+------------------+
| aspeed_wdt_probe | : map registers, install ops, register watchdog_dev
+-|----------------+
  |
  |--> alloc aspeed_wdt
  |
  |--> ioremap registers
  |
  |--> install 'aspeed_wdt_ops'
  |
  |    +-----------------------+
  |--> | watchdog_init_timeout | init timeout field
  |    +-----------------------+
  |
  |--> set wdt_ctrl based on properties
  |
  |--> if wdt is running (by checking hw reg)
  |    |
  |    |    +------------------+
  |    +--> | aspeed_wdt_start | ensure we're using the 1mhz clock source
  |         +------------------+
  |
  |--> if it's a 2500 or 2600 wdt
  |    -
  |    +--> set hw reg 'reset_width' based on dt properties
  |
  |    +-------------------------------+
  +--> | devm_watchdog_register_device | add watchdog_dev to list
       +-------------------------------+
```

```
drivers/watchdog/aspeed_wdt.c
+-------------------------------+
| devm_watchdog_register_device | : add watchdog_dev to list
+-|-----------------------------+
  |
  |--> alloc watchdog_device ptr as device resource
  |
  |    +--------------------------+
  |--> | watchdog_register_device | add watchdog_dev to list
  |    +--------------------------+
  |
  +--> have pointer point to the wdd
```

```                                                                                      
drivers/watchdog/watchdog_core.c                                                             
+--------------------------+                                                                  
| watchdog_register_device | : add watchdog_dev to list                                       
+-|------------------------+                                                                  
  |                                                                                           
  |--> if wtd_deferred_reg_done is set (not our case)                                         
  |    |                                                                                      
  |    |    +----------------------------+                                                    
  |    +--> | __watchdog_register_device | (skip)                                             
  |         +----------------------------+                                                    
  |                                                                                           
  +--> else                                                                                   
       |                                                                                      
       |    +------------------------------------+                                            
       +--> | watchdog_deferred_registration_add | add watchdog_dev to 'wtd_deferred_reg_list'
            +------------------------------------+                                            
```

```
drivers/watchdog/watchdog_core.c
+---------------+
| watchdog_init | : create 'watchdogd', register class, alloc dev#, register each watchdog_dev on list
+-|-------------+
  |    +-------------------+
  |--> | watchdog_dev_init | create kthread 'watchdogd', set to rt sched class, register 'watchdog_class', alloc dev#
  |    +-------------------+
  |    +--------------------------------+
  +--> | watchdog_deferred_registration | for each watchdog_dev on list: register it (work/timer/cdev/dev)
       +--------------------------------+
```

```
drivers/watchdog/watchdog_dev.c                                                                                
+-------------------+                                                                                           
| watchdog_dev_init | : create kthread 'watchdogd', set to rt sched class, register 'watchdog_class', alloc dev#
+-|-----------------+                                                                                           
  |                      +-----------------------+                                                              
  |-> watchdog_kworker = | kthread_create_worker |                                                              
  |                      +-----------------------+                                                              
  |                      create kthread (kthread_worker_fn), prepare worker, wake up kthread                    
  |                                                                                                             
  |   +----------------+                                                                                        
  |-> | sched_set_fifo | change policy to sched_fifo (rt class)                                                 
  |   +----------------+                                                                                        
  |   +----------------+                                                                                        
  |-> | class_register | register 'watchdog_class'                                                              
  |   +----------------+                                                                                        
  |   +---------------------+                                                                                   
  +-> | alloc_chrdev_region | alloc dev# (major = whatever, count = 32)                                         
      +---------------------+                                                                                   
```

```
drivers/watchdog/watchdog_core.c                                                                                    
+--------------------------------+                                                                                   
| watchdog_deferred_registration | : for each watchdog_dev on list: register it (work/timer/cdev/dev)                
+-|------------------------------+                                                                                   
  |                                                                                                                  
  |--> wtd_deferred_reg_done = true                                                                                  
  |                                                                                                                  
  +--> while there's still watchdog_dev on wtd_deferred_reg_list                                                     
       |                                                                                                             
       |--> remove watchdog_dev from list                                                                            
       |                                                                                                             
       |    +----------------------------+                                                                           
       +--> | __watchdog_register_device | get id, prepare watchdog_data, register cdev/dev, register restart handler
            +----------------------------+                                                                           
```

```
drivers/watchdog/watchdog_core.c                                                                          
+----------------------------+                                                                             
| __watchdog_register_device | : get id, prepare watchdog_data, register cdev/dev, register restart handler
+-|--------------------------+                                                                             
  |                                                                                                        
  |--> get a unique id and save in watchdog_dev (wdd)                                                      
  |                                                                                                        
  |    +-----------------------+                                                                           
  |--> | watchdog_dev_register | prepare watchdog_data (dev, work, timer), register cdev/dev               
  |    +-----------------------+                                                                           
  |                                                                                                        
  +--> if ->restart() exists                                                                               
       |                                                                                                   
       |    +--------------------------+ +---------------------------+                                     
       +--> | register_restart_handler | | watchdog_restart_notifier |                                     
            +--------------------------+ +---------------------------+                                     
                                         call ->restart()                                                  
```

```
drivers/watchdog/watchdog_dev.c                                                             
+-----------------------+                                                                    
| watchdog_dev_register | : prepare watchdog_data (dev, work, timer), register cdev/dev      
+-|---------------------+                                                                    
  |    +------------------------+                                                            
  |--> | watchdog_cdev_register | prepare watchdog_data (dev, work, timer), register cdev/dev
  |    +------------------------+                                                            
  |    +------------------------------+                                                      
  +--> | watchdog_register_pretimeout | simply return 0 bc of disabled config                
       +------------------------------+                                                      
```

```
drivers/watchdog/watchdog_dev.c                                                            
+------------------------+                                                                  
| watchdog_cdev_register | : prepare watchdog_data (dev, work, timer), register cdev/dev    
+-|----------------------+                                                                  
  |                                                                                         
  |--> alloc watchdog_data (wdd), which includes 'device' struct                            
  |                                                                                         
  |--> set up device                                                                        
  |                                                                                         
  |    +-------------------+ +--------------------+                                         
  |--> | kthread_init_work | | watchdog_ping_work | feed watchdog if needed                 
  |    +-------------------+ +--------------------+                                         
  |                                +------------------------+                               
  |--> init timer and install func | watchdog_timer_expired | queue work to watchdog_kworker
  |                                +------------------------+                               
  |--> if wdd id == 0                                                                       
  |    |                                                                                    
  |    |    +---------------+                                                               
  |    +--> | misc_register | determine dev#, add arg 'misc' to list (misc_list)            
  |         +---------------+                                                               
  |    +-----------+ +---------------+                                                      
  |--> | cdev_init | | watchdog_fops |                                                      
  |    +-----------+ +---------------+                                                      
  |    +-----------------+                                                                  
  |--> | cdev_device_add | add cdev to kobj_map and register inner dev                      
  |    +-----------------+                                                                  
  |                                                                                         
  +--> if watchdog hw is running                                                            
       -                                                                                    
       +--> if handle_boot_enabled is set (our case)                                        
            |                                                                               
            |    +---------------+                                                          
            +--> | hrtimer_start |                                                          
                 +---------------+                                                          
```

```
drivers/watchdog/watchdog_dev.c                                                
+--------------------+                                                          
| watchdog_ping_work | : feed watchdog if needed                                
+-|------------------+                                                          
  |                                                                             
  |--> get watchdog_data from work                                              
  |                                                                             
  |    +-----------------------------+                                          
  |--> | watchdog_worker_should_ping | check if we should ping the watchdog     
  |    +-----------------------------+                                          
  |                                                                             
  +--> if should                                                                
       |                                                                        
       |    +-----------------+                                                 
       +--> | __watchdog_ping | feed watchdog, start or cancel timer accordingly
            +-----------------+                                                 
```

```
drivers/watchdog/watchdog_dev.c                                      
+-----------------------------+                                       
| watchdog_worker_should_ping | : check if we should ping the watchdog
+-|---------------------------+                                       
  |                                                                   
  |--> if watchdog_dev is active, return true                         
  |                                                                   
  +--> if hw is still running && haven't passed deadline, return true 
```

```
drivers/watchdog/watchdog_dev.c                                             
+-----------------+                                                          
| __watchdog_ping | : feed watchdog, start or cancel timer accordingly       
+-|---------------+                                                          
  |                                                                          
  |--> ->last_xxx = now                                                      
  |                                                                          
  |--> if ->ping() exists                                                    
  |    -                                                                     
  |    +--> call ->ping(), e.g.,                                             
  |         +-----------------+                                              
  |         | aspeed_wdt_ping | write magic to hw reg 'restart'              
  |         +-----------------+                                              
  |                                                                          
  |--> else                                                                  
  |    -                                                                     
  |    +--> call ->start()                                                   
  |                                                                          
  |    +-----------------------------------+                                 
  |--> | watchdog_hrtimer_pretimeout_start | do nothing bc of disabled config
  |    +-----------------------------------+                                 
  |    +------------------------+                                            
  +--> | watchdog_update_worker | start or cancel timer accordingly          
       +------------------------+                                            
```

```
drivers/watchdog/watchdog_dev.c                                   
+------------------------+                                         
| watchdog_update_worker | : start or cancel timer accordingly     
+-|----------------------+                                         
  |    +----------------------+                                    
  |--> | watchdog_need_worker | check if we need the worker        
  |    +----------------------+                                    
  |                                                                
  |--> if need                                                     
  |    |                                                           
  |    |    +-------------------------+                            
  |    |--> | watchdog_next_keepalive | calculate the time interval
  |    |    +-------------------------+                            
  |    |    +---------------+                                      
  |    +--> | hrtimer_start |                                      
  |         +---------------+                                      
  |                                                                
  +--> else                                                        
       |                                                           
       |    +----------------+                                     
       +--> | hrtimer_cancel |                                     
            +----------------+                                     
```

## <a name="cheat-sheet"></a> Cheat Sheet


## <a name="reference"></a> Reference

