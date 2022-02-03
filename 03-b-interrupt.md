> Study case: Linux version 5.15.0 on OpenBMC

## Index

- [Introduction](#introduction)
- [Boot Flow](#boot-flow)
- [IRQ Registration](#irq-registration)
- [IRQ Execution](#irq-execution)
- [Software IRQ](#software-irq)
- [To-Do List](#to-do-list)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

Hardware components send out interrupts to inform all kinds of events, e.g., timer tick, data arrival on a network card, users input a character. 
Unlike instruction SWI, these device events are asynchronous, and current tasks don't trigger them directly. 
Once the CPU receives an interrupt, it sequentially switches to IRQ and SVC mode and then calls the interrupt service routine (ISR) to handle the event properly. 
This chapter will introduce this mechanism with an example and talk about software interruptions.

## <a name="boot-flow"></a> Boot Flow

In our study case, the config enables CONFIG_SPARSE_IRQ, and the kernel benefits from less memory usage at the cost of longer traverse time. 
It's suitable for systems with fewer interrupts since a traversal is negligible in that case. 
First, the kernel prepares 16 irq descriptors and inserts them into a tree structure for later traverse when an interrupt happens.

- Figure

```
+--------------+                                                               
| start_kernel |                                                               
+---|----------+                                                               
    |    +----------------+                                                    
    +--> | early_irq_init |                                                    
         +---|------------+                                                    
             |                                                                 
             |--> print "NR_IRQS: 16, nr_irqs: 16, preallocated irqs: 16"      
             |                                                                 
             +--> prepare 16 irq_desc and add to irq_desc_tree for later search
```

Then the data specified in DTS instructs the kernel to execute the below three interrupt initialization by checking the property _compatible_.

- DTS (arch/arm/boot/dts/aspeed-g5.dtsi)

```
        vic: interrupt-controller@1e6c0080 {
            compatible = "aspeed,ast2400-vic"; ==========> avic_of_init
            interrupt-controller;
            #interrupt-cells = <1>; 
            valid-sources = <0xfefff7ff 0x0807ffff>;
            reg = <0x1e6c0080 0x80>;
        };

...

                scu_ic: interrupt-controller@18 {
                    #interrupt-cells = <1>;
                    compatible = "aspeed,ast2500-scu-ic"; ==========> aspeed_scu_ic_of_init
                    reg = <0x18 0x4>;
                    interrupts = <21>;
                    interrupt-controller;
                };
   
...
   
&i2c {
    i2c_ic: interrupt-controller@0 {
        #interrupt-cells = <1>;
        compatible = "aspeed,ast2500-i2c-ic"; ==========> aspeed_i2c_ic_of_init
        reg = <0x0 0x40>;
        interrupts = <12>;
        interrupt-controller;
    };
```

- Code flow

```
+--------------+                                            
| start_kernel |                                            
+---|----------+                                            
    |    +----------+                                       
    +--> | init_IRQ |                                       
         +--|-------+                                       
            |    +--------------+                           
            +--> | irqchip_init |                           
                 +---|----------+                           
                     |    +-------------+                   
                     +--> | of_irq_init |                   
                          +---|---------+                   
                              |    +--------------+         
                              |--> | avic_of_init |         
                              |    +--------------+         
                              |    +-----------------------+
                              |--> | aspeed_scu_ic_of_init |
                              |    +-----------------------+
                              |    +-----------------------+
                              +--> | aspeed_i2c_ic_of_init |
                                   +-----------------------+
```

```
+--------------+                                                
| avic_of_init |                                                
+---|----------+                                                
    |    +----------+                                           
    |--> | of_iomap | read register base from DTS/DTB           
    |    +----------+                                           
    |    +-------------+                                        
    |--> | vic_init_hw | set up registers to mask all interrupts
    |    +-------------+                                        
    |    +----------------+                                     
    |--> | set_handle_irq | handle_arch_irq = avic_handle_irq   
    |    +----------------+                                     
    |    +-----------------------+                              
    +--> | irq_domain_add_simple | didn't look into it yet      
         +-----------------------+                              
```

```
+-----------------------+ 
| aspeed_scu_ic_of_init | 
+-----|-----------------+ 
      |   
      |--> set up scu interrupt controller (scu_ic) 
      |   
      |    +------------------------------+ 
      +--> | aspeed_scu_ic_of_init_common | 
           +-------|----------------------+ 
                   |    +-----------------------+ 
                   |--> | syscon_node_to_regmap | ensure system controller exists and return its register map 
                   |    +-----------------------+ 
                   |    +-----------------------+ 
                   |--> | irq_domain_add_linear | register domain 
                   |    +-----------------------+ 
                   |    +----------------------------------+ 
                   +--> | irq_set_chained_handler_and_data | handler = aspeed_scu_ic_irq_handler 
                        +----------------------------------+ 
```

```
+-----------------------+                                                          
| aspeed_i2c_ic_of_init |                                                          
+-----|-----------------+                                                          
      |                                                                            
      |--> set up i2c interrupt controller (i2c_ic)                                
      |                                                                            
      |    +----------+                                                            
      |--> | of_iomap | read register base from DTS/DTB                            
      |    +----------+                                                            
      |    +----------------------+                                                
      +--> | irq_of_parse_and_map | get parent irq                                 
      |    +----------------------+                                                
      |    +-----------------------+                                               
      |--> | irq_domain_add_linear | register domain                               
      |    +-----------------------+                                               
      |    +----------------------------------+                                    
      |--> | irq_set_chained_handler_and_data | handler = aspeed_i2c_ic_irq_handler
      |    +----------------------------------+                                    
      |                                                                            
      +--> print "i2c controller registered, irq 17"
```

## <a name="irq-registration"></a> IRQ Registration

If a device emits an interrupt for an event, its driver will prepare the ISR and register to that interrupt number. 
In most cases, a handler is enough to process the event, but some drivers will instead have the kthread ready for handling. 
The primary registration function is *__setup_irq*, responsible for installing the _descriptor_'s _action_. 
If kthread isn't an option to the drivers, there are two typical wrappers for use:

- _request_irq_: register handler to the interrupt number, it then generates the action to contain handler, and install it to irq descriptor.
- _devm_request_irq_: similar to the above, except it belongs to devise resource and will be freed if kernel releases the device.



```
      +-------------+                                                                     
+-----| request_irq | register ISR to an interrupt                                        
|     +-------------+                                                                     
|                                                                                         
|     +------------------+                                                                
| +---| devm_request_irq | register ISR to an interrupt, and treat it as a device resource
| |   +------------------+                                                                
| |                                                                                       
| |   +--------------+                                                                    
+---->| __setup_irq  |                                                                    
      +---|----------+                                                                    
          |                                                                               
          |--> if thread function is provided                                             
          |                                                                               
          |        +------------------+                                                   
          |        | setup_irq_thread | create a kthread to help handle this interrupt    
          |        +------------------+                                                   
          |                                                                               
          |--> install irq action to irq descriptor                                       
          |                                                                               
          +--> if kthread is generated                                                    
                                                                                          
                   +-----------------+                                                    
                   | wake_up_process |                                                    
                   +-----------------+                                                    
```

We can inspect the information of all registered interrupts through the proc file. 
The hardware IRQ is the number from the interrupt controller perspective, while the regular interrupt number is the virtual one.
We traverse and fetch the target IRQ descriptor by the hardware IRQ.

```                                                                                        
 root@romulus:~# cat /proc/interrupts                                                             
            CPU0                                                                                  
  18:    3968106      AVIC       16 Edge      FTTMR010-TIMER1        fttmr010_timer_interrupt     
  20:       6383      AVIC        2 Edge      eth0                   ftgmac100_interrupt          
  21:          0      AVIC        5 Edge      aspeed_vhub            ast_vhub_irq                 
  22:          0      AVIC       25 Edge      aspeed gfx             aspeed_gfx_irq_handler       
  23:          0      AVIC        7 Edge      aspeed-video           irq_default_primary_handler + aspeed_video_irq
  32:          0      AVIC        9 Edge      ttyS0                  serial8250_interrupt         
  33:        922      AVIC       10 Edge      ttyS4                  serial8250_interrupt         
  34:          0      AVIC        8 Edge      ipmi-bt-host, ttyS5    bt_bmc_irq                   
  35:        214     dummy        1 Edge      1e78a080.i2c-bus       aspeed_i2c_bus_irq           
  36:        214     dummy        2 Edge      1e78a0c0.i2c-bus       aspeed_i2c_bus_irq           
  37:        214     dummy        3 Edge      1e78a100.i2c-bus       aspeed_i2c_bus_irq           
  38:        214     dummy        4 Edge      1e78a140.i2c-bus       aspeed_i2c_bus_irq           
  39:        214     dummy        5 Edge      1e78a180.i2c-bus       aspeed_i2c_bus_irq           
  40:        214     dummy        6 Edge      1e78a1c0.i2c-bus       aspeed_i2c_bus_irq           
  41:        214     dummy        7 Edge      1e78a300.i2c-bus       aspeed_i2c_bus_irq           
  42:        214     dummy        8 Edge      1e78a340.i2c-bus       aspeed_i2c_bus_irq           
  43:        214     dummy        9 Edge      1e78a380.i2c-bus       aspeed_i2c_bus_irq           
  44:        214     dummy       10 Edge      1e78a3c0.i2c-bus       aspeed_i2c_bus_irq           
  45:        652     dummy       11 Edge      1e78a400.i2c-bus       aspeed_i2c_bus_irq           
  46:        216     dummy       12 Edge      1e78a440.i2c-bus       aspeed_i2c_bus_irq           
  47:          0  1e780000.gpio  74 Edge      checkstop              gpio_keys_gpio_isr           
  48:          0  1e780000.gpio 135 Edge      id-button              gpio_keys_gpio_isr           
  49:          0  1e780000.gpio  67 Edge      gpiolib                gpio_sysfs_irq               
  50:          0  1e780000.gpio  73 Edge      gpiolib                gpio_sysfs_irq               
 Err:          0                                                                                  
 ----        ---  -------------  +- ----      ----------------       ------------------           
 irq#       stat      chip       |  trigger        action         ISR, just for reference         
                      name       |   type           name                                          
                                 v                                                                
                                hwirq                                                             
```

## <a name="irq-execution"></a> IRQ Execution

We've learned from the _Exception_ chapter that interruptions will cause the CPU to switch to IRQ mode and jump to the corresponding routine in the vector table. 
Whether it switches from _USR_ or _SVC_ mode, it ultimately leads to _handle_arch_irq_, which points to _avic_handle_irq_ during initialization.

```
+------------+                                                                             
| vector_irq |                                                                             
+------------+                                                                             
      -                                                                                    
      -                                                                                    
      v                                                                                    
+-----------------+                                                                        
| avic_handle_irq |                                                                        
+----|------------+                                                                        
     |                                                                                     
     +--> endless loop                                                                     
                                                                                           
              check register if there's any pending interrupt                              
                                                                                           
              exit loop if there's none                                                    
                                                                                           
              +-------------------+                                                        
              | handle_domain_irq |                                                        
              +----|--------------+                                                        
                   |                                                                       
                   |--> fetch target irq descriptor by hw irq                              
                   |                                                                       
                   |    +-----------------+                                                
                   +--> | handle_irq_desc |                                                
                        +----|------------+                                                
                             |    +-------------------------+                              
                             +--> | generic_handle_irq_desc |                              
                                  +------|------------------+                              
                                         |                                                 
                                         +--> call desc->handle_irq, e.g., handle_level_irq
                                              which is installed by domain->map()          
```

Commonly speaking, the action->handler plays the role of ISR as we know it. 
If a driver chooses a kthread over a handler to help process the event, **irq_default_primary_handler** will be the default handler for later kthread wake up.

```
+------------------+
| handle_level_irq |
+----|-------------+
     |    +------------------+
     +--> | handle_irq_event |
          +----|-------------+
               |    +-------------------------+
               +--> | handle_irq_event_percpu |
                    +------|------------------+
                           |    +---------------------------+
                           +--> | __handle_irq_event_percpu |
                                +------|--------------------+
                                       |
                                       |--> action->handler, e.g., serial8250_interrupt
                                       |
                                       +--> if there's kthread prepared during registration

                                                +-------------------+
                                                | __irq_wake_thread |
                                                +-------------------+                
```

## <a name="software-irq"></a> Software IRQ

Software IRQ is similar to its hardware version, except software triggers the interrupt and handles it appropriately. 
It's a fixed table shown below, and each index corresponds to an action or NULL. 
Subsystems like timer, network, and scheduler, register these actions in the boot-up flow, and then somewhere trigger them in runtime if necessary.

| Index            | Action                | Note                           |
| ---              | ---                   |---                             |
| HI_SOFTIRQ       | tasklet_hi_action     | (TBD)                          |
| TIMER_SOFTIRQ    | run_timer_softirq     | (TBD)                          |
| NET_TX_SOFTIRQ   | net_tx_action         | (TBD)                          |
| NET_RX_SOFTIRQ   | net_rx_action         | (TBD)                          |
| BLOCK_SOFTIRQ    | blk_done_softirq      | (TBD)                          |
| IRQ_POLL_SOFTIRQ | irq_poll_softirq      | config is disabled             |
| TASKLET_SOFTIRQ  | tasklet_action        | (TBD)                          |
| SCHED_SOFTIRQ    | run_rebalance_domains | help balance the load among rq |
| HRTIMER_SOFTIRQ  | hrtimer_run_softirq   | (TBD)                          |
| RCU_SOFTIRQ      | rcu_core_si           | (TBD)                          |

```
open_softirq(index, action): register the action for an index
raise_softirq(index): ensure the task 'ksoftirqd/#' is active to handle the request
```

Each processor has its **ksoftirqd**, and we can discern that by checking its tailing number, e.g., **ksoftirqd/1** runs only on the run queue of processor one. 
Function **spawn_ksoftirqd** is responsible for initializing the per-CPU task.

```
  +-------------+                                                                                                 
  | kernel_init |                                                                                                 
  +-------------+                                                                                                 
         -                                                                                                        
         -                                                                                                        
         v                                                                                                        
+-----------------+                                                                                               
| spawn_ksoftirqd | for each cpu, prepare a kthread running 'run_ksoftirqd'
+----|------------+                                                                                               
     |    +--------------------------------+                                                                      
     +--> | smpboot_register_percpu_thread | for each cpu, prepare a kthread running specified hotplug thread
          +--------------------------------+                                                                      
```

If there's no pending softirq, the ksoftirqd will sleep till somewhere invokes it again.

```
+---------------+                                                                      
| run_ksoftirqd |                                                                      
+---|-----------+                                                                      
    |                                                                                  
    +--> if somewhere raised the softirq (saved in percpu variable)                    
                                                                                       
             +--------------+                                                          
             | __do_softirq |                                                          
             +---|----------+                                                          
                 |                                                                     
                 +--> get pending softirq(s) from percpu variable                      
                                                                                       
                      reset that variable to 0 <-----------------------------+         
                                                                             |         
                      for each softirq (0 ~ 9)                               |         
                                                                             |         
                          call ->action() if the corresponding bit is raised |         
                                +---------------+                            |         
                          e.g., | net_rx_action |                            |         
                                +---------------+                            |         
                                                                             |         
                      if there's new pending softirq(s)                      |         
                                                                             |         
                          if it's appropriate to handle it now               |         
                                                                             |         
                              go to -----------------------------------------+         
                                                                                       
                          else                                                         
                                                                                       
                              +-----------------+                                      
                              | wakeup_softirqd | wake up the ksoftirqd to deal with it
                              +-----------------+                                      
```

## <a name="to-do-list"></a> To-Do List

- Why are there duplicated hardware IRQs?
- Why handle_level_irq is used to handle 'edge' type interrupt?
- Complete software IRQ table
- Introduce IRQ domain
- Introduce IPI

## <a name="reference"></a> Reference

(None)


