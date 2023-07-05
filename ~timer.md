```
init_timers
hrtimers_init
timekeeping_init
time_init
```


```
kernel/time/timer.c                                                                 
+-------------+                                                                      
| init_timers | : init percpu timer bases, register softirq action                   
+-|-----------+                                                                      
  |    +-----------------+                                                           
  |--> | init_timer_cpus | init percpu timer_bases                                   
  |    +-----------------+                                                           
  |    +---------------------------+                                                 
  |--> | posix_cputimers_init_work | do nothing bc of disabled config                
  |    +---------------------------+                                                 
  |    +--------------+                                                              
  +--> | open_softirq | register softirq action                                      
       +--------------+ +-------------------+                                        
                        | run_timer_softirq | run each expired timer of timer base(s)
                        +-------------------+                                        
```

```
kernel/time/timer.c                                           
+-------------------+                                          
| run_timer_softirq | : run each expired timer of timer base(s)
+-|-----------------+                                          
  |                                                            
  |--> get percpu timer base (std)                             
  |                                                            
  |    +--------------+                                        
  |--> | __run_timers | run each expired timer                 
  |    +--------------+                                        
  |                                                            
  +--> if deferrable timer base exists                         
       |                                                       
       |    +--------------+                                   
       +--> | __run_timers | run each expired timer            
            +--------------+                                   
```

```
kernel/time/timer.c                                                                 
+--------------+                                                                     
| __run_timers | : run each expired timer                                            
+-|------------+                                                                     
  |                                                                                  
  |--> if nothing expires yet, return                                                
  |                                                                                  
  +--> while there's at least one expired timer                                      
      |                                                                              
      |    +------------------------+                                                
      |--> | collect_expired_timers | collect expired timers from arg base           
      |    +------------------------+                                                
      |                                                                              
      |--> base->clk++                                                               
      |                                                                              
      |    +------------------------+                                                
      |--> | __next_timer_interrupt | look into timer base to get next expiration    
      |    +------------------------+                                                
      |                                                                              
      +--> while level--                                                             
           |                                                                         
           |    +---------------+                                                    
           +--> | expire_timers | given list, detach each timer and call its function
                +---------------+                                                    
```

```
kernel/time/timer.c                                                           
+------------------------+                                                     
| collect_expired_timers | : collect expired timers from arg base              
+-|----------------------+                                                     
  |                                                                            
  +--> for each level_depth                                                    
       |                                                                       
       |--> determine idx                                                      
       |                                                                       
       |--> if it's set in bitmap (clear it)                                   
       |    |                                                                  
       |    |--> get vector from base (base contains all vectors in all wheels)
       |    |                                                                  
       |    |--> add vector to head list                                       
       |    |                                                                  
       |    +--> heads++, level++                                              
       |                                                                       
       |--> if no need to check next level, break                              
       |                                                                       
       +--> adjust level to fit next granularity                               
```

```
kernel/time/timer.c                                                    
+------------------------+                                              
| __next_timer_interrupt | : look into timer base to get next expiration
+-|----------------------+                                              
  |                                                                     
  +--> for each level depth                                             
       |                                                                
       |    +---------------------+                                     
       |--> | next_pending_bucket | get pos of next pending bucket      
       |    +---------------------+                                     
       |                                                                
       |--> if it expires early than next, update next with it          
       |                                                                
       |--> if no timer can expire earlier, break                       
       |                                                                
       +--> adjust clk for next round                                   
```

```
kernel/time/timer.c                                                   
+---------------+                                                      
| expire_timers | : given list, detach each timer and call its function
+-|-------------+                                                      
  |                                                                    
  +--> while list isn't empty                                          
       |                                                               
       |--> get first timer from list                                  
       |                                                               
       |    +--------------+                                           
       |--> | detach_timer |                                           
       |    +--------------+                                           
       |    +---------------+                                          
       +--> | call_timer_fn |                                          
            +---------------+                                          
```

```
kernel/time/hrtimer.c                                                                      
+---------------+                                                                           
| hrtimers_init | : init hrtimer bases, register softirq action                             
+-|-------------+                                                                           
  |    +----------------------+                                                             
  |--> | hrtimers_prepare_cpu | init hrtimer bases for the current cpu                      
  |    +----------------------+                                                             
  |    +--------------+                                                                     
  +--> | open_softirq | register softirq action                                             
       +--------------+ +---------------------+                                             
                        | hrtimer_run_softirq | run expired hrtimers of percpu hrtimer bases
                        +---------------------+                                             
```

```
kernel/time/hrtimer.c                                                                    
+---------------------+                                                                   
| hrtimer_run_softirq | : run expired hrtimers of percpu hrtimer bases                    
+-|-------------------+                                                                   
  |                                                                                       
  |--> get percpu hrtimer base                                                            
  |                                                                                       
  |    +---------------------+                                                            
  |--> | hrtimer_update_base | given hrtimer abse, update its realtime/boottime/tai offset
  |    +---------------------+                                                            
  |    +----------------------+                                                           
  +--> | __hrtimer_run_queues | for each base, run expired hrtimers                       
       +----------------------+                                                           
```

```
kernel/time/hrtimer.c                                                                 
+----------------------+                                                               
| __hrtimer_run_queues | : for each base, run expired hrtimers                         
+-|--------------------+                                                               
  |                                                                                    
  +--> for each base in hrtimer_bases                                                  
       -                                                                               
       +--> while we can still get active hrtimer from base                            
            |                                                                          
            |--> if it's not expired, break                                            
            |                                                                          
            |    +---------------+                                                     
            +--> | __run_hrtimer | detach hrtimer, call function, attach back if needed
                 +---------------+                                                     
```

```
kernel/time/hrtimer.c                                                                         
+---------------+                                                                              
| __run_hrtimer | : detach hrtimer, call function, attach back if needed                       
+-|-------------+                                                                              
  |    +------------------+                                                                    
  |--> | __remove_hrtimer | detach hrtimer from queue, if queue becomes empty, clear bit@bitmap
  |    +------------------+                                                                    
  |                                                                                            
  |--> call timer's function                                                                   
  |                                                                                            
  +--> if need restart                                                                         
       |                                                                                       
       |    +-----------------+                                                                
       +--> | enqueue_hrtimer | add hrtimer back to base->active                               
            +-----------------+                                                                
```

```
kernel/time/timekeeping.c                                             
+------------------+                                                   
| timekeeping_init |                                                   
+-|----------------+                                                   
  |    +--------------------------------------+                        
  |--> | read_persistent_wall_and_boot_offset |                        
  |    +--------------------------------------+                        
  |    +----------+                                                    
  |--> | ntp_init | (skip)                                             
  |    +----------+                                                    
  |    +---------------------------+                                   
  |--> | clocksource_default_clock | get 'clocksource_jiffies'         
  |    +---------------------------+                                   
  |                                                                    
  |--> if it has ->enable(), call it (not our case)                    
  |                                                                    
  |    +--------------------+                                          
  |--> | tk_setup_internals | set up internals to use clocksource clock
  |    +--------------------+                                          
  |                                                                    
  |--> set walltime to 'tk_core'                                       
  |                                                                    
  |    +--------------------+                                          
  +--> | timekeeping_update |                                          
       +--------------------+                                          
```

```
arch/arm/kernel/time.c                                                                                  
+-----------+                                                                                            
| time_init | : init clock, probe timer, setup broadcast                                                 
+-|---------+                                                                                            
  |                                                                                                      
  |--> if machine desc has ->init_time()                                                                 
  |    -                                                                                                 
  |    +--> (not our case)                                                                               
  |                                                                                                      
  +--> else                                                                                              
       |                                                                                                 
       |    +-------------+                                                                              
       |--> | of_clk_init | for each clk provider, call its callback and remove from list                
       |    +-------------+                                                                              
       |    +-------------+                                                                              
       |--> | timer_probe | probe timer (register clocksource/schedclock_read()/timer_isr/clockevent_dev)
       |    +-------------+                                                                              
       |    +------------------------------+                                                             
       +--> | tick_setup_hrtimer_broadcast | (skip)                                                      
            +------------------------------+                                                             
```

```
drivers/clk/clk.c                                                                                 
+-------------+                                                                                    
| of_clk_init | : for each clk provider, call its callback and remove from list                    
+-|-----------+                                                                                    
  |                                                                                                
  |--> if arg 'matches' isn't provided                                                             
  |    -                                                                                           
  |    +--> use '__clk_of_table' by default                                                        
  |                                                                                                
  |--> for each dt node of entry in table                                                          
  |    |                                                                                           
  |    |--> alloc 'parent' for the match                                                           
  |    |    (the matched one is probably 'aspeed,ast2500-scu')                                     
  |    |                                                                                           
  |    +--> add 'parent' to 'clk_provider_list'                                                    
  |                                                                                                
  +--> while 'clk_provider_list' isn't empty                                                       
       -                                                                                           
       +--> for each provider in list                                                              
            |                                                                                      
            |--> call ->clk_init_cb(), e.g.,                                                       
            |    +----------------+                                                                
            |    | aspeed_cc_init | read 'strap' reg, register fixed factors, register clk_provider
            |    +----------------+                                                                
            |                                                                                      
            |    +---------------------+                                                           
            |--> | of_clk_set_defaults | (seems do nothing)                                        
            |    +---------------------+                                                           
            |                                                                                      
            +--> remove clk_provider from list and free it                                         
```

```
drivers/clk/clk-aspeed.c                                                                                
+----------------+                                                                                       
| aspeed_cc_init | : read 'strap' reg, register fixed factors, register clk_provider                     
+-|--------------+                                                                                       
  |    +----------+                                                                                      
  |--> | of_iomap | map scu base                                                                         
  |    +----------+                                                                                      
  |                                                                                                      
  |--> alloc 'aspeed_clk_data'                                                                           
  |                                                                                                      
  |    +-----------------------+                                                                         
  |--> | syscon_node_to_regmap | get syscon regmap (what's the difference from scu base？）                
  |    +-----------------------+                                                                         
  |    +-------------+                                                                                   
  |--> | regmap_read | read 'aspeed_strap'                                                               
  |    +-------------+                                                                                   
  |                                                                                                      
  |--> if the node is compatible with 'aspeed,ast2500-scu'                                               
  |    |                                                                                                 
  |    |    +-------------------+                                                                        
  |    +--> | aspeed_ast2500_cc | read 'strap' register, register fixed factors for 'clkin', 'ahb', 'apb'
  |         +-------------------+                                                                        
  |    +------------------------+                                                                        
  +--> | of_clk_add_hw_provider | alloc 'clk_provider' and register to 'of_clk_providers'                
       +------------------------+                                                                        
```

```
drivers/clocksource/timer-probe.c                                                                                           
+-------------+                                                                                                              
| timer_probe | : probe timer (register clocksource/schedclock_read()/timer_isr/clockevent_dev)                              
+-|-----------+                                                                                                              
  |                                                                                                                          
  +--> for each matched dt_node of entry in '__timer_of_table'                                                               
       |                                                                                                                     
       |--> (the matched node is probably compatible with 'aspeed,ast2400-timer')                                            
       |                                                                                                                     
       +--> call init_func_ret(), e.g.,                                                                                      
            +-------------------+                                                                                            
            | aspeed_timer_init | register clocksource, register read() for sched, request timer irq, register clockevent_dev
            +-------------------+                                                                                            
```

```
drivers/clocksource/timer-probe.c                                                                                    
+-------------------+                                                                                                 
| aspeed_timer_init | : register clocksource, register read() for sched, request timer irq, register clockevent_dev   
+----------------------+                                                                                              
| fttmr010_common_init | : register clocksource, register read() for sched, request timer irq, register clockevent_dev
+-|--------------------+                                                                                              
  |                                                                                                                   
  |--> alloc and setup 'fttmr010'                                                                                     
  |                                                                                                                   
  |    +----------+                                                                                                   
  |--> | of_iomap |                                                                                                   
  |    +----------+                                                                                                   
  |    +----------------------+                                                                                       
  |--> | irq_of_parse_and_map | get timer irq#                                                                        
  |    +----------------------+                                                                                       
  |                                                                                                                   
  +--> write hw reg to get timer start and disable its interrupt                                                      
  |                                                                                                                   
  |    +-----------------------+                                                                                      
  |--> | clocksource_mmio_init | prepare clocksource and register it, update 'curr_clocksource' is condition met      
  |    +-----------------------+                                                                                      
  |    +----------------------+                                                                                       
  |--> | sched_clock_register | register read() function for sched clock                                              
  |    +----------------------+                                                                                       
  |                                                                                                                   
  |--> write hw reg to set up clockevent timer (interrupt-driven)                                                     
  |                                                                                                                   
  |    +-------------+                                                                                                
  |--> | request_irq | register isr for timer interrupt                                                               
  |    +-------------+ +--------------------------+                                                                   
  |                    | fttmr010_timer_interrupt | call clockevent_dev->event_handler()                              
  |                    +--------------------------+                                                                   
  |                                                                                                                   
  |--> set up clockevent_dev (install ops)                                                                            
  |                                                                                                                   
  |    +---------------------------------+                                                                            
  |--> | clockevents_config_and_register | register evtdev, install to tick_dev if preferred                          
  |    +---------------------------------+                                                                            
  |    +------------------------------+                                                                               
  +--> | register_current_timer_delay | install delay timer                                                           
       +------------------------------+                                                                               
```

```
drivers/clocksource/timer-fttmr010.c                                                                      
+-----------------------+                                                                                  
| clocksource_mmio_init | : prepare clocksource and register it, update 'curr_clocksource' is condition met
+-|---------------------+                                                                                  
  |                                                                                                        
  |--> alloc and setup 'cs' (clocksource)                                                                  
  |                                                                                                        
  |    +-------------------------+                                                                         
  +--> | clocksource_register_hz | register clocksource, switch 'curr_clocksource' to it if it's better    
       +-------------------------+                                                                         
```

```
drivers/clocksource/mmio.c                                                                            
+-------------------------+                                                                            
| clocksource_register_hz | : register clocksource, switch 'curr_clocksource' to it if it's better     
+------------------------------+                                                                       
| __clocksource_register_scale | : register clocksource, switch 'curr_clocksource' to it if it's better
+-|----------------------------+                                                                       
  |    +-----------------------+                                                                       
  |--> | clocksource_arch_init | (do nothing on arm)                                                   
  |    +-----------------------+                                                                       
  |    +---------------------------------+                                                             
  |--> | __clocksource_update_freq_scale | update mult and shift of cs                                 
  |    +---------------------------------+                                                             
  |    +---------------------+                                                                         
  |--> | clocksource_enqueue | add to 'clocksource_list' ('rating' ascending list)                     
  |    +---------------------+                                                                         
  |    +------------------------------+                                                                
  |--> | clocksource_enqueue_watchdog | (do nothing bc of disabled config)                             
  |    +------------------------------+                                                                
  |    +--------------------+                                                                          
  |--> | clocksource_select | select the best clocksource and switch to it                             
  |    +--------------------+                                                                          
  |    +-----------------------------+                                                                 
  |--> | clocksource_select_watchdog | (do nothing bc of disabled config)                              
  |    +-----------------------------+                                                                 
  |    +------------------------------+                                                                
  +--> | __clocksource_suspend_select | determine 'suspend_clocksource'                                
       +------------------------------+                                                                
```

```
kernel/time/clocksource.c                                                
+---------------------------------+                                       
| __clocksource_update_freq_scale | : update mult and shift of cs         
+-|-------------------------------+                                       
  |                                                                       
  |--> if arg freq is provided                                            
  |    |                                                                  
  |    |    +------------------------+                                    
  |    +--> | clocks_calc_mult_shift | update mult and shift of cs        
  |         +------------------------+                                    
  |                                                                       
  +--> print "%s: mask: 0x%llx max_cycles: 0x%llx, max_idle_ns: %lld ns\n"
```

```
kernel/time/clocksource.c                                             
+--------------------+                                                 
| clocksource_select | : select the best clocksource and switch to it  
+----------------------+                                               
| __clocksource_select | : select the best clocksource and switch to it
+-|--------------------+                                               
  |    +-----------------------+                                       
  |--> | clocksource_find_best | get the best clocksource              
  |    +-----------------------+                                       
  |                                                                    
  |--> if no 'override_name' (our case)                                
  |    -                                                               
  |    +--> go to 'found'                                              
  |                                                                    
  |found:                                                              
  |                                                                    
  +--> if 'curr_clocksource' isn't the best clocksource yet            
       |                                                               
       |--> print "Switched to clocksource %s\n"                       
       |                                                               
       +--> 'curr_clocksource' = best                                  
```

```
kernel/time/sched_clock.c                                                             
+----------------------+                                                               
| sched_clock_register | : register read() function for sched clock                    
+-|--------------------+                                                               
  |                                                                                    
  |--> set up 'cd' (clock data)                                                        
  |                                                                                    
  |--> if sched_clock_timer is ready                                                   
  |    |                                                                               
  |    |    +---------------+                                                          
  |    +--> | hrtimer_start | sched_clock_timer                                        
  |         +---------------+                                                          
  |                                                                                    
  +--> print "sched_clock: %u bits at %lu%cHz, resolution %lluns, wraps every %lluns\n"
```

```
drivers/clocksource/timer-fttmr010.c                                                   
+---------------------------------+                                                     
| clockevents_config_and_register | : register evtdev, install to tick_dev if preferred 
+-|-------------------------------+                                                     
  |    +--------------------+                                                           
  |--> | clockevents_config | colnfig min_delta_ns/max_delta_ns of clockevent_dev       
  |    +--------------------+                                                           
  |    +-----------------------------+                                                  
  +--> | clockevents_register_device | register evtdev, install to tick_dev if preferred
       +-----------------------------+                                                  
```

```
kernel/time/clockevents.c                                                                           
+-----------------------------+                                                                      
| clockevents_register_device | : register evtdev, install to tick_dev if preferred                  
+-|---------------------------+                                                                      
  |                                                                                                  
  |--> register clockevent_dev to 'clockevent_devices'                                               
  |                                                                                                  
  |    +-----------------------+                                                                     
  |--> | tick_check_new_device | if arg evtdev is preferred, install it to tick_dev                  
  |    +-----------------------+                                                                     
  |    +-----------------------------+                                                               
  +--> | clockevents_notify_released | try to install those released evtdev to tick_dev (should fail)
       +-----------------------------+                                                               
```

```
kernel/time/tick-common.c                                                            
+-----------------------+                                                             
| tick_check_new_device | : if arg evtdev is preferred, install it to tick_dev        
+-|---------------------+                                                             
  |                                                                                   
  |--> get percpu tick_dev, and further get current clockevent_dev from it            
  |                                                                                   
  |    +------------------------+                                                     
  |--> | tick_check_replacement | check if new clockevent_dev is preferred            
  |    +------------------------+                                                     
  |                                                                                   
  |--> if not, go to 'out'                                                            
  |                                                                                   
  |    +-----------------------------+                                                
  |--> | clockevents_exchange_device | remove old one and shutdown new one            
  |    +-----------------------------+                                                
  |    +-------------------+                                                          
  +--> | tick_setup_device | install arg evtdev to tick_dev, ensure next event happens
  |    +-------------------+                                                          
  |out                                                                                
  |    +-------------------------------+                                              
  +--> | tick_install_broadcast_device | (skip)                                       
       +-------------------------------+                                              
```

```
kernel/time/tick-common.c                                                                                        
+-------------------+                                                                                             
| tick_setup_device | : install arg evtdev to tick_dev, ensure next event happens                                 
+-|-----------------+                                                                                             
  |                                                                                                               
  |--> if tick_dev has no clockevent_dev yet                                                                      
  |    |                                                                                                          
  |    |--> if no cpu responsible for do_timer                                                                    
  |    |    |                                                                                                     
  |    |    |--> assign that job to current cpu                                                                   
  |    |    |                                                                                                     
  |    |    +--> tick_next_period = now                                                                           
  |    |                                                                                                          
  |    +--> td->mode = periodic                                                                                   
  |                                                                                                               
  |--> else                                                                                                       
  |    |                                                                                                          
  |    |--> get handler from td->evtdev                                                                           
  |    |                                                                                                          
  |    +--> reset td->evtdev's handler                                                                            
  |                                                                                                               
  |--> td->evtdev = arg dev                                                                                       
  |                                                                                                               
  |--> if td->mode == periodic                                                                                    
  |    |                                                                                                          
  |    |    +---------------------+                                                                               
  |    +--> | tick_setup_periodic | do nothing if it's periodic, otherwise (oneshot), program timer for next event
  |         +---------------------+                                                                               
  |                                                                                                               
  +--> else                                                                                                       
       |                                                                                                          
       |    +--------------------+                                                                                
       +--> | tick_setup_oneshot | install handler, program next event                                            
            +--------------------+                                                                                
```

```
kernel/time/tick-common.c                                                                              
+---------------------+                                                                                 
| tick_setup_periodic | : do nothing if it's periodic, otherwise (oneshot), program timer for next event
+-|-------------------+                                                                                 
  |    +---------------------------+                                                                    
  |--> | tick_set_periodic_handler | install evtdev handler                                             
  |    +---------------------------+ +----------------------+                                           
  |                                  | tick_handle_periodic |                                           
  |                                  +----------------------+                                           
  |                                                                                                     
  |--> if it's a dummy dev, return                                                                      
  |                                                                                                     
  |--> if state == periodic                                                                             
  |    |                                                                                                
  |    |    +--------------------------+                                                                
  |    +--> | clockevents_switch_state | periodic                                                       
  |         +--------------------------+                                                                
  |                                                                                                     
  +--> else                                                                                             
       |                                                                                                
       |    +--------------------------+                                                                
       |--> | clockevents_switch_state | oneshot                                                        
       |    +--------------------------+                                                                
       |    +---------------------------+                                                               
       +--> | clockevents_program_event | program timer for next event                                  
            +---------------------------+                                                               
```

```
arch/arm/lib/delay.c                                                  
+------------------------------+                                       
| register_current_timer_delay | : install delay timer                 
+-|----------------------------+                                       
  |                                                                    
  |--> install delay timer                                             
  |                                                                    
  +--> print "Switching to timer-based delay loop, resolution %lluns\n"
```
