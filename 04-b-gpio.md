> Study case: Linux version 5.15.0 on OpenBMC

## Index

- [Introduction](#introduction)
- [Boot Flow](#boot-flow)
- [To-Do List](#to-do-list)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

Generic purpose input-output (GPIO) can work as output to control target device or as input to receive signals from outside. 
In modern design, these pins are usually multi-functional. One pin can be an SDA in SMBus protocol or a TX in UART with the proper configuration. 
Pin control (pinctrl) is the mechanism implemented in the kernel to properly configure those pins when we expect them to perform a specific function. 

## <a name="boot-flow"></a> Boot Flow

The pinctrl driver and the device in DTS/DTB fit each other because their property 'compatible' hold the same value (**aspeed,ast2500-pinctrl**). 
Then the match triggers probe function **aspeed_g5_pinctrl_probe** for further structure allocation and set up.

- Driver

```
static const struct of_device_id aspeed_g5_pinctrl_of_match[] = {
    { .compatible = "aspeed,ast2500-pinctrl", },
    /*
     * The aspeed,g5-pinctrl compatible has been removed the from the
     * bindings, but keep the match in case of old devicetrees.
     */
    { .compatible = "aspeed,g5-pinctrl", },
    { },
};

static struct platform_driver aspeed_g5_pinctrl_driver = {
    .probe = aspeed_g5_pinctrl_probe,
    .driver = {
        .name = "aspeed-g5-pinctrl",
        .of_match_table = aspeed_g5_pinctrl_of_match,
    },
};
```

- Device

```
                pinctrl: pinctrl@80 {
                    compatible = "aspeed,ast2500-pinctrl";
                    reg = <0x80 0x18>, <0xa0 0x10>;
                    aspeed,external-nodes = <&gfx>, <&lhc>;
                };
```

- Code flow

```
+-------------------------+                                                                  
| aspeed_g5_pinctrl_probe |                                                                  
+------|------------------+                                                                  
       |                                                                                     
       |--> reassign the number of each pin in aspeed_g5_pins                                
       |                                                                                     
       |    +----------------------+                                                         
       +--> | aspeed_pinctrl_probe |                                                         
            +-----|----------------+                                                         
                  |    +-----------------------+                                             
                  |--> | syscon_node_to_regmap | get scu register base address               
                  |    +-----------------------+                                             
                  |    +------------------+                                                  
                  +--> | pinctrl_register | set up pin device and register all the pins to it
                       +------------------+                                                  
```

```
+------------------+                                                    
| pinctrl_register |                                                    
+----|-------------+                                                    
     |    +-------------------------+                                   
     +--> | pinctrl_init_controller |                                   
          +------|------------------+                                   
                 |                                                      
                 |--> allocate and set up pinctrl dev                   
                 |                                                      
                 |    +-----------------------+                         
                 +--> | pinctrl_register_pins |                         
                      +-----|-----------------+                         
                            |                                           
                            +--> for each pin                           
                                                                        
                                     +--------------------------+       
                                     | pinctrl_register_one_pin |       
                                     +------|-------------------+       
                                            |                           
                                            |--> allocate pin descriptor
                                            |                           
                                            +--> add to pinctrl dev     
```

```
+------------------------------------+                                                                     
| pinconf_generic_dt_node_to_map_all | save mux/configs info of one DT node to map                         
+--------|---------------------------+                                                                     
         |    +--------------------------------+                                                           
         +--> | pinconf_generic_dt_node_to_map |                                                           
              +-------|------------------------+                                                           
                      |    +-----------------------------------+                                           
                      |--> | pinconf_generic_dt_subnode_to_map |                                           
                      |    +--------|--------------------------+                                           
                      |             |                                                                      
                      |             |--> get property 'function'                                           
                      |             |                                                                      
                      |             |    +---------------------------------+                               
                      |             |--> | pinconf_generic_parse_dt_config |                               
                      |             |    +---------------------------------+                               
                      |             |                                                                      
                      |             +--> for each string in property (one string in our case)              
                      |                                                                                    
                      |                      if property 'function' exists                                 
                      |                                                                                    
                      |                          +---------------------------+                             
                      |                          | pinctrl_utils_add_map_mux | save mux info in map        
                      |                          +---------------------------+                             
                      |                                                                                    
                      |                      if pinconf node has config(s)                                 
                      |                                                                                    
                      |                          +-------------------------------+                         
                      |                          | pinctrl_utils_add_map_configs | save configs info in map
                      |                          +-------------------------------+                         
                      |                                                                                    
                      +-->  apply the same logic to the child nodes (no child in our case)                 
```

```
+----------------------+                                                                                    
| dt_to_map_one_config | save info to map, add to pinctrl and pinctrl_maps (global)                         
+-----|----------------+                                                                                    
      |                                                                                                     
      |--> get pinctrl dev                                                                                  
      |                                                                                                     
      |--> call ->dt_node_to_map()                                                                          
      |          +------------------------------------+                                                     
      |    e.g., | pinconf_generic_dt_node_to_map_all | save mux/configs info of one DT node to map         
      |          +------------------------------------+                                                     
      |                                                                                                     
      |    +-------------------------+                                                                      
      +--> | dt_remember_or_free_map |                                                                      
           +------|------------------+                                                                      
                  |                                                                                         
                  |--> set up dt_map and add to pinctrl                                                     
                  |                                                                                         
                  |    +----------------------------+                                                       
                  +--> |  pinctrl_register_mappings | set up maps_node and register to pinctrl_maps (global)
                       +----------------------------+                                                       
```

```
+----------------+                                                                                            
| create_pinctrl |                                                                                            
+---|------------+                                                                                            
    |                                                                                                         
    |--> allocate pinctrl                                                                                     
    |                                                                                                         
    |    +-------------------+                                                                                
    |--> | pinctrl_dt_to_map |                                                                                
    |    +----|--------------+                                                                                
    |         |                                                                                               
    |         +--> for each state ('default', 'init', 'sleep', 'idle')                                        
    |                                                                                                         
    |                  get property from DT                                                                   
    |                                                                                                         
    |                  return if not found                                                                    
    |                                                                                                         
    |                  for each config                                                                        
    |                                                                                                         
    |                      +-------------------------+                                                        
    |                      | of_find_node_by_phandle | find target node by config (handle)                    
    |                      +-------------------------+                                                        
    |                                                                                                         
    |                      +----------------------+                                                           
    |                      | dt_to_map_one_config | save info to map, add to pinctrl and pinctrl_maps (global)
    |                      +----------------------+                                                           
    |                                                                                                         
    +--> return if the device needs no pinctrl                                                                
```

```
+-----------------------+                                                                                                             
| pinmux_enable_setting |                                                                                                             
+-----|-----------------+                                                                                                             
      |                                                                                                                               
      |--> get pinctrl ops from pinctrl dev                                                                                           
      |                                                                                                                               
      |--> get pinmux ops from pinctrl dev                                                                                            
      |                                                                                                                               
      |--> if pinctrl_ops->get_group_pins is prepared                                                                                 
      |                                                                                                                               
      |        all pinctrl_ops->get_group_pins()                                                                                      
      |              +-------------------------------+                                                                                
      |        e.g., | aspeed_pinctrl_get_group_pins | get a group of pins                                                            
      |              +-------------------------------+                                                                                
      |                                                                                                                               
      |--> for each pin in group                                                                                                      
      |                                                                                                                               
      |        +-------------+                                                                                                        
      |        | pin_request |                                                                                                        
      |        +---|---------+                                                                                                        
      |            |                                                                                                                  
      |            +--> get pin desc from pinctrl dev                                                                                 
      |                                                                                                                               
      |                 update owner of the pin desc                                                                                  
      |                                                                                                                               
      |                 call pinmux_ops->gpio_request_enable()                                                                        
      |                       +----------------------------+                                                                          
      |                 e.g., | aspeed_gpio_request_enable | disable all other high priority functions, so it's now a regular gpio pin
      |                       +----------------------------+                                                                          
      |                                                                                                                               
      |    call pinmux_ops->set_mux()                                                                                                 
      +-->       +-----------------------+                                                                                            
           e.g., | aspeed_pinmux_set_mux | disable higher priority functions, and enable mux                                          
                 +-----------------------+                                                                                            
```

```
+-----------------------+                                                                              
| pinconf_apply_setting |                                                                              
+-----|-----------------+                                                                              
      |                                                                                                
      |--> if setting type is PIN                                                                      
      |                                                                                                
      |        call ops->pin_config_set()                                                              
      |              +-----------------------+                                                         
      |        e.g., | aspeed_pin_config_set | for each config, update register based on config and map
      |              +-----------------------+                                                         
      |                                                                                                
      +--> if setting type is GROUP                                                                    
                                                                                                       
               call ops->pin_config_group_set()                                                        
                     +-----------------------------+                                                   
               e.g., | aspeed_pin_config_group_set | get a group of pins and config them               
                     +-----------------------------+                                                   
```

```
+----------------------+                                                                            
| pinctrl_select_state |                                                                            
+-----|----------------+                                                                            
      |                                                                                             
      |--> return if it's already the specified state                                               
      |                                                                                             
      |    +----------------------+                                                                 
      +--> | pinctrl_commit_state |                                                                 
           +-----|----------------+                                                                 
                 |                                                                                  
                 |--> if pinctrl has old state                                                      
                 |                                                                                  
                 |        +------------------------+                                                
                 |        | pinmux_disable_setting |                                                
                 |        +------------------------+                                                
                 |                                                                                  
                 |--> for each setting on new state                                                 
                 |                                                                                  
                 |        +-----------------------+                                                 
                 |        | pinmux_enable_setting | request pin(s) and set mux to enable the setting
                 |        +-----------------------+                                                 
                 |                                                                                  
                 |--> for each setting on new state                                                 
                 |                                                                                  
                 |        +-----------------------+                                                 
                 |        | pinconf_apply_setting | configure pin(s)                                
                 |        +-----------------------+                                                 
                 |                                                                                  
                 +--> update pinctrl state                                                          
```

## <a name="to-do-list"></a> To-Do List

- Introduce GPIO and multi-function with their priority

## <a name="reference"></a> Reference

(None)


