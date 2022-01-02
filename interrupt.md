> Study case: Linux version 5.15.0 on OpenBMC

## Index

- [Introduction](#introduction)
- [Boot Flow](#boot-flow)
- [ISR Registration](#isr-registeration)
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

## <a name="isr-registration"></a> ISR Registration





## <a name="reference"></a> Reference




