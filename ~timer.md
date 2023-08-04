> Study case: ASPEED Linux version 5.15.41

## Index

- [Introduction](#introduction)
- [System Startup](#system-startup)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

```
kernel/time/posix-timers.c                                                      
+----------------------------+                                                   
| sys_clock_nanosleep_time32 | : set up hrtimer sleeper, sleep till condition met
+-|--------------------------+                                                   
  |    +-------------------+                                                     
  |--> | clockid_to_kclock | given clock id, get k_clock                         
  |    +-------------------+                                                     
  |                                                                              
  +--> call ->nsleep(), e.g.,                                                    
       +---------------+                                                         
       | common_nsleep | set up hrtimer sleeper, sleep till condition met        
       +---------------+                                                         
```

```
kernel/time/posix-timers.c                                                  
+---------------+                                                            
| common_nsleep | : set up hrtimer sleeper, sleep till condition met         
+-|-------------+                                                            
  |    +---------------------+                                               
  |--> | timespec64_to_ktime | convert timespec to ktime                     
  |    +---------------------+                                               
  |    +-------------------+                                                 
  +--> | hrtimer_nanosleep | set up hrtimer sleeper, sleep till condition met
       +-------------------+                                                 
```

```
kernel/time/hrtimer.c                                                       
+-------------------+                                                        
| hrtimer_nanosleep | : set up hrtimer sleeper, sleep till condition met     
+-|-----------------+                                                        
  |    +-------------------------------+                                     
  |--> | hrtimer_init_sleeper_on_stack | set up hrtimer sleeper for 'current'
  |    +-------------------------------+                                     
  |    +------------------------------+                                      
  |--> | hrtimer_set_expires_range_ns | set timer expires                    
  |    +------------------------------+                                      
  |    +--------------+                                                      
  |--> | do_nanosleep | task sleeps till condition met                       
  |    +--------------+                                                      
  |    +--------------------------+                                          
  +--> | destroy_hrtimer_on_stack | do nothing bc of disabled config         
       +--------------------------+                                          
```

```
kernel/time/hrtimer.c                                                  
+-------------------------------+                                       
| hrtimer_init_sleeper_on_stack | : set up hrtimer sleeper for 'current'
+----------------------+--------+                                       
| hrtimer_init_sleeper | : set up hrtimer sleeper for 'current'         
+------------------------+                                              
| __hrtimer_init_sleeper | : set up hrtimer sleeper for 'current'       
+-|----------------------+                                              
  |    +----------------+                                               
  |--> | __hrtimer_init | save base in hrtimer                          
  |    +----------------+                                               
  |                                                                     
  |--> install function                                                 
  |    +----------------+                                               
  |    | hrtimer_wakeup | wake up task, return no-restart               
  |    +----------------+                                               
  |                                                                     
  +--> save 'current' in sleeper                                        
```

```
kernel/time/hrtimer.c                   
+----------------+                       
| __hrtimer_init | : save base in hrtimer
+-|--------------+                       
  |                                      
  |--> get percpu hrtimer bases          
  |                                      
  +--> determine base and save in hrtimer
```

```
kernel/time/hrtimer.c                              
+----------------+                                  
| hrtimer_wakeup | : wake up task, return no-restart
+-|--------------+                                  
  |                                                 
  |--> if task                                      
  |    |                                            
  |    |    +-----------------+                     
  |    +--> | wake_up_process |                     
  |         +-----------------+                     
  |                                                 
  +--> return no-restart                            
```

```
kernel/time/hrtimer.c                                              
+--------------+                                                    
| do_nanosleep | : task sleeps till condition met                   
+|-------------+                                                    
 |                                                                  
 |--> while saved task isn't cleared && no signal pending           
 |    |                                                             
 |    |    +-------------------------------+                        
 |    |--> | hrtimer_sleeper_start_expires | start a hrtimer sleeper
 |    |    +-------------------------------+                        
 |    |    +--------------------+                                   
 |    |--> | freezable_schedule | schedule                          
 |    |    +--------------------+                                   
 |    |    +----------------+                                       
 |    +--> | hrtimer_cancel | cancel a hrtimer                      
 |         +----------------+                                       
 |                                                                  
 +--> if saved task info is cleared, return (normal path)           
```





```
kernel/time/tick-common.c                                                                                    
+----------------------+                                                                                      
| tick_handle_periodic | : update jiffies/wall_time, run expired hrtimers/timers/posix_timers, update rq clock
+-|--------------------+                                                                                      
  |    +---------------+                                                                                      
  |--> | tick_periodic | update jiffies/wall_time, run expired hrtimers/timers/posix_timers, update rq clock  
  |    +---------------+                                                                                      
  |                                                                                                           
  |--> if event handler is overwritten (not tick_handle_periodic anymore), return                             
  |                                                                                                           
  +--> if state isn't oneshot, return                                                                         
       |                                                                                                      
       |    +---------------------------+                                                                     
       +--> | clockevents_program_event | program next event                                                  
            +---------------------------+                                                                     
```

```
kernel/time/tick-common.c                                                                             
+---------------+                                                                                      
| tick_periodic | : update jiffies/wall_time, run expired hrtimers/timers/posix_timers, update rq clock
+-|-------------+                                                                                      
  |                                                                                                    
  |--> if current cpu is owner of the action                                                           
  |    |                                                                                               
  |    |--> update 'tick_next_period' (next event)                                                     
  |    |                                                                                               
  |    |    +----------+                                                                               
  |    |--> | do_timer | update 'jiffies_64' and global loads                                          
  |    |    +----------+                                                                               
  |    |    +------------------+                                                                       
  |    +--> | update_wall_time | update wall time                                                      
  |         +------------------+                                                                       
  |    +----------------------+                                                                        
  |--> | update_process_times | run hrtimers/timers, update rq clock, run posix timers                 
  |    +----------------------+                                                                        
  |    +--------------+                                                                                
  +--> | profile_tick | do nothing bc of disabled config                                               
       +--------------+                                                                                
```

```
kernel/time/timer.c                                                                         
+----------------------+                                                                     
| update_process_times | : run hrtimers/timers, update rq clock, run posix timers            
+-|--------------------+                                                                     
  |    +------------------+                                                                  
  |--> | run_local_timers | run expired hrtimers and timers                                  
  |    +------------------+                                                                  
  |                                                                                          
  |--> if it's in irq                                                                        
  |    |                                                                                     
  |    |    +---------------+                                                                
  |    +--> | irq_work_tick | (skip)                                                         
  |         +---------------+                                                                
  |                                                                                          
  |    +----------------+                                                                    
  |--> | scheduler_tick | update rq clock, call ->task_tick(), balance runqueues if necessary
  |    +----------------+                                                                    
  |                                                                                          
  +--> if CONFIG_POSIX_TIMERS                                                                
       |                                                                                     
       |    +----------------------+                                                         
       +--> | run_posix_cpu_timers | collect expired timers from task/group, trigger them    
            +----------------------+                                                         
```

```
kernel/time/timer.c                                                                       
+------------------+                                                                       
| run_local_timers | : switch to hrtimer mechanism, or run expired hrtimers                
+-|----------------+                                                                       
  |                                                                                        
  |--> get percpu timer base (std)                                                         
  |                                                                                        
  |    +--------------------+                                                              
  |--> | hrtimer_run_queues | switch to hrtimer mechanism, or run expired hrtimers         
  |    +--------------------+                                                              
  |                                                                                        
  +--> if 'jiffies' exceeds expire time                                                    
       |                                                                                   
       |    +---------------+                                                              
       +--> | raise_softirq | timer                                                        
            +---------------+ +-------------------+                                        
                              | run_timer_softirq | run each expired timer of timer base(s)
                              +-------------------+                                        
```

```
kernel/time/hrtimer.c                                                                       
+--------------------+                                                                       
| hrtimer_run_queues | : switch to hrtimer mechanism, or run expired hrtimers                
+-|------------------+                                                                       
  |                                                                                          
  |--> get percpu hrtimer base                                                               
  |                                                                                          
  |--> if hrtimer mechanism isn't active, return                                             
  |                                                                                          
  |--> if we are to to switch to hrtimer mechanism                                           
  |    |                                                                                     
  |    |    +------------------------+                                                       
  |    |--> | hrtimer_switch_to_hres | overwrite handler for hrtimer, switch to oneshot mode,
  |    |    +------------------------+ lable 'hres_active', setup sched hrtimer              
  |    |                                                                                     
  |    +--> return                                                                           
  |                                                                                          
  |--> if at lease one timer expires                                                         
  |    |                                                                                     
  |    |    +----------------------++---------------------+                                  
  |    +--> | raise_softirq_irqoff || hrtimer_run_softirq |                                  
  |         +----------------------++---------------------+                                  
  |                                 run expired hrtimers of percpu hrtimer bases             
  |                                                                                          
  |    +----------------------+                                                              
  +--> | __hrtimer_run_queues | for each base, run expired hrtimers                          
       +----------------------+                                                              
```

```
kernel/time/hrtimer.c                                                                                                      
+------------------------+                                                                                                  
| hrtimer_switch_to_hres | : overwrite handler for hrtimer, switch to oneshot mode, lable 'hres_active', setup sched hrtimer
+-|----------------------+                                                                                                  
  |    +-------------------+                                                                                                
  |--> | tick_init_highres | overwrite evtdev handler for hrtimer, switch to state/mode to oneshot                          
  |    +-------------------+                                                                                                
  |                                                                                                                         
  |--> label hres_active in base                                                                                            
  |                                                                                                                         
  |    +------------------------+                                                                                           
  |--> | tick_setup_sched_timer | setup a sched hrtimer                                                                     
  |    +------------------------+                                                                                           
  |    +----------------------+                                                                                             
  +--> | retrigger_next_event | retrigger the interrupt to get things going?                                                
       +----------------------+                                                                                             
```

```
kernel/time/tick-oneshot.c                                                                       
+-------------------+                                                                             
| tick_init_highres | : overwrite evtdev handler for hrtimer, switch to state/mode to oneshot     
+------------------------+                                                                        
| tick_switch_to_oneshot | : overwrite evtdev handler for hrtimer, switch to state/mode to oneshot
+-|----------------------+                                                                        
  |                                                                                               
  |--> if evtdev doesn't support one shot, return error                                           
  |                                                                                               
  |--> tick_dev mode = oneshot                                                                    
  |                                                                                               
  |--> install arg handler, e.g.,                                                                 
  |    +-------------------+                                                                      
  |    | hrtimer_interrupt | for each base: run expired hrtimers, program next event              
  |    +-------------------+                                                                      
  |                                                                                               
  |    +--------------------------+                                                               
  |--> | clockevents_switch_state | oneshot                                                       
  |    +--------------------------+                                                               
  |    +----------------------------------+                                                       
  +--> | tick_broadcast_switch_to_oneshot | (skip)                                                
       +----------------------------------+                                                       
```

```
kernel/time/hrtimer.c                                                                       
+-------------------+                                                                        
| hrtimer_interrupt | : for each base: run expired hrtimers, program next event              
+-|-----------------+                                                                        
  |                                                                                          
  |--> get percpu hrtimer base                                                               
  |                                                                                          
  |    +---------------------+                                                               
  |--> | hrtimer_update_base | update boot/real/tai offset of arg base                       
  |    +---------------------+                                                               
  |                                                                                          
  |--> if softriq expires                                                                    
  |    |                                                                                     
  |    |    +----------------------+                                                         
  |    +--> | raise_softirq_irqoff | raise the corresponding flag (hrtimer) in softirq bitmap
  |         +----------------------+                                                         
  |    +----------------------+                                                              
  |--> | __hrtimer_run_queues | for each base, run expired hrtimers                          
  |    +----------------------+                                                              
  |    +---------------------------+                                                         
  |--> | hrtimer_update_next_event | calculate next expire time                              
  |    +---------------------------+                                                         
  |    +--------------------+                                                                
  +--> | tick_program_event | given next expire time, program evtdev event                   
       +--------------------+                                                                
```

```
kernel/time/tick-sched.c                                                                                                     
+------------------------+                                                                                                    
| tick_setup_sched_timer | : setup a sched hrtimer                                                                            
+-|----------------------+                                                                                                    
  |    +--------------+                                                                                                       
  |--> | hrtimer_init | init hrtimer                                                                                          
  |    +--------------+                                                                                                       
  |                                                                                                                           
  |--> install function                                                                                                       
  |    +------------------+                                                                                                   
  |    | tick_sched_timer | update jiffies/wall_time, run hrtimers/timers/posix_timers, update rq clock, forward hrtimer      
  |    +------------------+                                                                                                   
  |                                                                                                                           
  |    +---------------------+                                                                                                
  |--> | hrtimer_set_expires | set next expiry                                                                                
  |    +---------------------+                                                                                                
  |                                                                                                                           
  |--> if 'sched_skew_tick' is set (probably not our case), blabla                                                            
  |                                                                                                                           
  |    +-----------------+                                                                                                    
  |--> | hrtimer_forward | forward hrtimer                                                                                    
  |    +-----------------+                                                                                                    
  |    +-----------------------+                                                                                              
  |--> | hrtimer_start_expires | dequeue hrtimer, set next expiry, switch base and enqueue, re-program next event is necessary
  |    +-----------------------+                                                                                              
  |    +--------------------+                                                                                                 
  +--> | tick_nohz_activate | (skip)                                                                                          
       +--------------------+                                                                                                 
```

```
kernel/time/tick-sched.c                                                                                          
+------------------+                                                                                               
| tick_sched_timer | : update jiffies/wall_time, run hrtimers/timers/posix_timers, update rq clock, forward hrtimer
+-|----------------+                                                                                               
  |    +---------------------+                                                                                     
  |--> | tick_sched_do_timer | update jiffies/global_load/wall_time                                                
  |    +---------------------+                                                                                     
  |                                                                                                                
  |--> if we're in irq context                                                                                     
  |    |                                                                                                           
  |    |    +-------------------+                                                                                  
  |    +--> | tick_sched_handle | run hrtimers/timers, update rq clock, run posix timers                           
  |         +-------------------+                                                                                  
  |    +-----------------+                                                                                         
  +--> | hrtimer_forward | updatre next expiry of hrtimer                                                          
       +-----------------+                                                                                         
```

```
kernel/time/tick-sched.c                                                    
+---------------------+                                                      
| tick_sched_do_timer | : update jiffies/global_load/wall_time               
+-|-------------------+                                                      
  |                                                                          
  +--> if current cpu is the one                                             
       |                                                                     
       |    +--------------------------+                                     
       +--> | tick_do_update_jiffies64 | update jiffies/global_load/wall_time
            +--------------------------+                                     
```

```
kernel/time/tick-sched.c                                          
+--------------------------+                                       
| tick_do_update_jiffies64 | : update jiffies/global_load/wall_time
+-|------------------------+                                       
  |                                                                
  |--> get 'tick_next_period'                                      
  |                                                                
  |--> if 'now' doesn't reach next expire, return                  
  |                                                                
  |--> update 'jiffies_64'                                         
  |                                                                
  |--> update 'tick_next_period'                                   
  |                                                                
  |    +------------------+                                        
  |--> | calc_global_load |                                        
  |    +------------------+                                        
  |    +------------------+                                        
  +--> | update_wall_time |                                        
       +------------------+                                        
```

```
kernel/time/tick-sched.c                                                             
+-------------------+                                                                 
| tick_sched_handle | : run hrtimers/timers, update rq clock, run posix timers        
+-|-----------------+                                                                 
  |    +----------------------+                                                       
  |--> | update_process_times | run hrtimers/timers, update rq clock, run posix timers
  |    +----------------------+                                                       
  |    +--------------+                                                               
  +--> | profile_tick | do nothing bc of disabled config                              
       +--------------+                                                               
```

```
kernel/time/tick-sched.c                                                                                                        
+-----------------------+                                                                                                        
| hrtimer_start_expires | : dequeue hrtimer, set next expiry, switch base and enqueue, re-program next event is necessary        
+------------------------+                                                                                                       
| hrtimer_start_range_ns | : dequeue hrtimer, set next expiry, switch base and enqueue, re-program next event is necessary       
+-|----------------------+                                                                                                       
  |    +--------------------------+                                                                                              
  |--> | __hrtimer_start_range_ns | dequeue hrtimer, set next expiry, switch base and enqueue, re-program next event is necessary
  |    +--------------------------+                                                                                              
  |                                                                                                                              
  +--> if need reprogram                                                                                                         
       |                                                                                                                         
       |    +-------------------+                                                                                                
       +--> | hrtimer_reprogram | re-program next event                                                                          
            +-------------------+                                                                                                
```

```
kernel/time/hrtimer.c                                                                                                      
+--------------------------+                                                                                                
| __hrtimer_start_range_ns | : dequeue hrtimer, set next expiry, switch base and enqueue, re-program next event is necessary
+-|------------------------+                                                                                                
  |    +----------------+                                                                                                   
  |--> | remove_hrtimer | remove hrtimer, might reprogram next event accordingly                                            
  |    +----------------+                                                                                                   
  |    +------------------------------+                                                                                     
  |--> | hrtimer_set_expires_range_ns | set next expiry                                                                     
  |    +------------------------------+                                                                                     
  |                                                                                                                         
  |--> if no force_local                                                                                                    
  |    |                                                                                                                    
  |    |    +---------------------+                                                                                         
  |    +--> | switch_hrtimer_base | save target base in hrtimer                                                             
  |         +---------------------+                                                                                         
  |    +-----------------+                                                                                                  
  |--> | enqueue_hrtimer | add hrtimer to target base                                                                       
  |    +-----------------+                                                                                                  
  |                                                                                                                         
  |--> if no force_local, return                                                                                            
  |                                                                                                                         
  |    +-------------------------+                                                                                          
  +--> | hrtimer_force_reprogram | program next event                                                                       
       +-------------------------+                                                                                          
```

```
kernel/time/posix-cpu-timers.c                                                       
+----------------------+                                                              
| run_posix_cpu_timers | : collect expired timers from task/group, trigger them       
+-|--------------------+                                                              
  |    +----------------------+                                                       
  |--> | fastpath_timer_check | check if there's expired timer                        
  |    +----------------------+                                                       
  |                                                                                   
  |--> if no, return                                                                  
  |                                                                                   
  |    +------------------------+                                                     
  +--> | __run_posix_cpu_timers | collect expired timers from task/group, trigger them
       +------------------------+                                                     
```

```
kernel/time/posix-cpu-timers.c                                                       
+------------------------+                                                            
| __run_posix_cpu_timers | : collect expired timers from task/group, trigger them     
+-------------------------+                                                           
| handle_posix_cpu_timers | : collect expired timers from task/group, trigger them    
+-|-----------------------+                                                           
  |    +---------------------+                                                        
  |--> | check_thread_timers | move expired timers of given task to arg list          
  |    +---------------------+                                                        
  |    +----------------------+                                                       
  |--> | check_process_timers | move expired timers of given task's signal to arg list
  |    +----------------------+                                                       
  |                                                                                   
  +--> for each expired timer in list                                                 
       |                                                                              
       |--> remove it from list                                                       
       |                                                                              
       |    +----------------+                                                        
       +--> | cpu_timer_fire | determine pid type and send signal to target pid       
            +----------------+                                                        
```

```
kernel/time/posix-cpu-timers.c
+---------------------+                                                        
| check_thread_timers | : move expired timers of given task to arg list        
+-|-------------------+                                                        
  |                                                                            
  +--> get cpu timers from task                                                
  |                                                                            
  |    +---------------------+                                                 
  |--> | task_sample_cputime | take utime/stime from task and save in samples  
  |    +---------------------+                                                 
  |    +-------------------------+                                             
  +--> | collect_posix_cputimers | move expired timers of all bases to arg list
       +-------------------------+                                             
```

```
kernel/time/posix-cpu-timers.c                                           
+-------------------------+                                               
| collect_posix_cputimers | : move expired timers of all bases to arg list
+-|-----------------------+                                               
  |                                                                       
  +--> for each clock base                                                
       |                                                                  
       |    +--------------------+                                        
       +--> | collect_timerqueue | move expired timers to arg list        
            +--------------------+                                        
```

```
kernel/time/posix-cpu-timers.c                                
+--------------------+                                         
| collect_timerqueue | : move expired timers to arg list       
+-|------------------+                                         
  |                                                            
  +--> while time queue has next entry                         
       |                                                       
       |--> if the timer hans't expired, return                
       |                                                       
       |    +-------------------+                              
       |--> | cpu_timer_dequeue | remove timer from where it is
       |    +-------------------+                              
       |    +---------------+                                  
       +--> | list_add_tail | add to arg list                  
            +---------------+                                  
```

```
kernel/time/posix-cpu-timers.c                                                  
+----------------------+                                                         
| check_process_timers | : move expired timers of given task's signal to arg list
+-|--------------------+                                                         
  |                                                                              
  |--> get timer base(s) from signal                                             
  |                                                                              
  |    +-------------------------+                                               
  +--> | collect_posix_cputimers | move expired timers of all bases to arg list  
       +-------------------------+                                               
```

```
kernel/time/posix-cpu-timers.c                                                   
+----------------+                                                                
| cpu_timer_fire | : determine pid type and send signal to target pid             
+-|--------------+                                                                
  |                                                                               
  |--> if it's a oneshot timer                                                    
  |    |                                                                          
  |    |    +-------------------+                                                 
  |    |--> | posix_timer_event | determine pid type and send signal to target pid
  |    |    +-------------------+                                                 
  |    |                                                                          
  |    +--> clear expiry                                                          
  |                                                                               
  +--> else                                                                       
       |                                                                          
       |    +-------------------+                                                 
       +--> | posix_timer_event | determine pid type and send signal to target pid
            +-------------------+                                                 
```

## <a name="system-startup"></a> System Startup

```
start_kernel
|
|--> time_init
|    -
|    +--> timer_probe
|         -
|         +--> aspeed_timer_init
|              fttmr010_common_init
|              |
|              |--> clocksource_mmio_init
|              |    -
|              |    +--> clocksource_register_hz     [    0.000000] clocksource: FTTMR010-TIMER2: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 77222644334 ns
|              |
|              |--> sched_clock_register             [    0.000124] sched_clock: 32 bits at 24MHz, resolution 40ns, wraps every 86767015915ns
|              |
|              +--> register_current_timer_delay     [    0.001753] Switching to timer-based delay loop, resolution 40ns
|
+--> calibrate_delay                                 [    0.005060] Calibrating delay loop (skipped), value calculated using timer frequency.. 49.50 BogoMIPS (lpj=247500)


kernel_init
-
+--> kernel_init_freeable
     -
     +--> do_basic_setup
          |
          |--> init_jiffies_clocksource
          |    -
          |    +--> __clocksource_register           [    0.081094] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 19112604462750000 ns
          |
          |--> clocksource_done_booting
          |    -
          |    +--> clocksource_select               [    0.212355] clocksource: Switched to clocksource FTTMR010-TIMER2
          |
          +--> aspeed_i2c_bus_driver_init
               -
               +--> aspeed_i2c_probe_bus
                    -
                    +--> i2c_new_client_device
                         -
                         +--> rv8803_probe           [    1.722342] rtc-rv8803 11-0032: registered as rtc0
                                                     [    1.723329] rtc-rv8803 11-0032: setting system clock to 2023-07-05T03:02:00 UTC (1688526120)
```

```
init_timers
hrtimers_init
timekeeping_init
time_init
dummy_timer_starting_cpu: register dummy evtdev
init_timer_list_procfs: create '/proc/timer_list' with 'timer_list_sops' installed
alarmtimer_init: (skip, it matches no device)
init_posix_timers: create kmem cache for 'k_itimer'
timeriomem_rng_driver_init: (skip, random# related)
timer_led_trigger_init: (skip)
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

```
kernel/time/timer_list.c                                                              
+------------------------+                                                             
| init_timer_list_procfs | ： create '/proc/timer_list' with 'timer_list_sops' installed
+-|----------------------+                                                             
  |                                                                                    
  +--> create '/proc/timer_list' with 'timer_list_sops' installed                      
                                                                                       
                                                                                       
       static const struct seq_operations timer_list_sops = {                          
           .start = timer_list_start,                                                  
           .next = timer_list_next,                                                    
           .stop = timer_list_stop,                                                    
           .show = timer_list_show,                                                    
       };                                                                              
```

## <a name="reference"></a> Reference
