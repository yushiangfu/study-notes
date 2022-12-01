> Study case: Linux version 5.15.0 on OpenBMC

## Index

- [Introduction](#introduction)
- [Driver](#driver)
- [Pin Control](#pin-control)
- [System Startup](#system-startup)
- [Cheat Sheet](#cheat-sheet)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

Generic purpose input-output (GPIO) can work as output to control target device or as input to receive signals from outside. 
In modern design, these pins are usually multi-functional. One pin can be an SDA in SMBus protocol or a TX in UART with the proper configuration. 
Pin control (pinctrl) is the mechanism implemented in the kernel to properly configure those pins when we expect them to perform a specific function. 

## <a name="driver"></a> Driver

The 'GPIO Lib' framework in kernel space bridges the userspace requests and the real GPIO chip driver provided by the vendor, e.g., Aspeed.

```
                                                +--
                                                |   static const struct file_operations gpio_fileops = {
                                                |       .release = gpio_chrdev_release,
                                                |       .open = gpio_chrdev_open,
  user             +------+ e.g. /dev/gpiochip0 |       .poll = lineinfo_watch_poll,
  -----------------| cdev |------------------   |       .read = lineinfo_watch_read,
  kernel           +------+                     |       .owner = THIS_MODULE,
                       |                        |       .llseek = no_llseek,
                       v                        |       .unlocked_ioctl = gpio_ioctl,
             |--  +---------+                   |   };
    generic  |    | gpiolib |                   +--
             +--  +---------+
                       |
                       v
             |--  +--------+
   specific  |    | driver |
e.g. aspeed  +--  +--------+                                                                              
```

Nothing worths mentioning from the function **gpiolib_dev_init** in our study case. 
Just note that once **gpiolib_initialized** becomes true, the subsequent GPIO chip driver registration will set up the cdev file operations during initialization.

After kernel adds GPIO device based on device tree and initializes the GPIO driver, probe function triggers because the property **compatible** matches.

- Device descriptor in DTS

```
gpio@1e780000 {                  
    #gpio-cells = <0x02>;                  
    gpio-controller;                  
    compatible = "aspeed,ast2500-gpio";     <========== match
    reg = <0x1e780000 0x200>;                  
    interrupts = <0x14>;                  
    gpio-ranges = <0x0d 0x00 0x00 0xe8>;    <========== interpret as <pinctrl_phandle gpio_offset pin_offset pin#>
    clocks = <0x02 0x1a>;                  
    interrupt-controller;                  
    #interrupt-cells = <0x02>;                  
    gpio-line-names = [...
                       63 66 61 6d 2d 72 65 73 65 74 00 \               <========== GPIO_A0, "cfam-reset"
                       ... \
                       66 73 69 2d 6d 75 78 00 \                        <========== GPIO_A6, "fsi-mux"
                       ... \
                       66 73 69 2d 65 6e 61 62 6c 65 00 \               <========== GPIO_D0, "fsi-enable"
                       ... \
                       6e 69 63 5f 66 75 6e 63 5f 6d 6f 64 65 30 00 \   <========== GPIO_D3, "nic_func_mode0"
                       6e 69 63 5f 66 75 6e 63 5f 6d 6f 64 65 31 00 \   <========== GPIO_D4, "nic_func_mode1"
                       ... \
                       ... \
                       70 6f 77 65 72 2d 62 75 74 74 6f 6e 00 \         <========== GPIO_I3, "power-button"
                       ... \
                       63 68 65 63 6b 73 74 6f 70 00 \                  <========== GPIO_J2, "checkstop"
                       ... \
                       6c 65 64 2d 66 61 75 6c 74 00 \                  <========== GPIO_N2, "led-fault"
                       ... \
                       6c 65 64 2d 69 64 65 6e 74 69 66 79 00 \         <========== GPIO_N4, "led-identify"
                       ... \
                       69 64 2d 62 75 74 74 6f 6e 00 \                  <========== GPIO_Q7, "id-button"
                       ... \
                       66 73 69 2d 74 72 61 6e 73 00 \                  <========== GPIO_R2, "fsi-trans"
                       ... \
                       6c 65 64 2d 70 6f 77 65 72 00 \                  <========== GPIO_R5, "led-power"
                       ... \
                       73 65 71 5f 63 6f 6e 74 00 \                     <========== GPIO_S7, "seq_cont"
                       ... \
                       66 73 69 2d 63 6c 6f 63 6b 00 \                  <========== GPIO_AA0, "fsi-clock"
                       ... \
                       66 73 69 2d 64 61 74 61 00 \                     <========== GPIO_AA2, "fsi-data"
                       ...];
    phandle = <0x29>; 
```

- Table in driver

```
static const struct of_device_id aspeed_gpio_of_table[] = {
    { .compatible = "aspeed,ast2400-gpio", .data = &ast2400_config, },
    { .compatible = "aspeed,ast2500-gpio", .data = &ast2500_config, }, <========== match
    { .compatible = "aspeed,ast2600-gpio", .data = &ast2600_config, },
    {}
};
```

The probe function allocates vendor-specific structure, installs regular GPIO operations, and sets up the IRQ chip since it's also an interrupt controller. 
Then it calls **devm_gpiochip_add_data** to connect the generic layer (GPIO Lib) to the specific driver (Aspeed).

At the moment, we have the complete GPIO framework, and userspace utilities can send requests through the char device.

```
root@romulus:~# gpiodetect
gpiochip0 [1e780000.gpio] (232 lines)

root@romulus:~# gpioinfo 0 | head
gpiochip0 - 232 lines:
        line   0:      unnamed       unused   input  active-high 
        line   1: "cfam-reset"       unused   input  active-high 
        line   2:      unnamed       unused   input  active-high 
        line   3:      unnamed       unused   input  active-high 
        line   4:      unnamed       kernel   input  active-high [used]
        line   5:      unnamed       kernel   input  active-high [used]
        line   6:    "fsi-mux"       unused  output  active-high 
        line   7:      unnamed       unused   input  active-high 
        line   8:      unnamed       unused   input  active-high 

root@romulus:~# gpioget 0 2     # get value from gpio chip '0' line '2'
0

root@romulus:~# gpioset 0 2=1   # set value to gpio chip '0' line '2' to '1'

root@romulus:~# gpioget 0 2     # get value from gpio chip '0' line '2'
1

root@romulus:~# gpiomon 0 2     # monitor value change on gpio chip '0' line '2'
```

Here's the example of a flow chart showing how the code switches from generic layer (GPIO Lib) to chip vendor (Aspeed) implementation.

```
                                            |                                   
                                   gpio lib | aspeed driver                     
                                            |                                   
+----------------------+                    |                                   
| gpiod_request_commit |                    |                                   
+-----|----------------+                    |                                   
      |                                     |                                   
      |--> gc->request() -------------------|--- e.g., aspeed_gpio_request      
      |                                     |                                   
      |    +---------------------+          |                                   
      +--> | gpiod_get_direction |          |                                   
           +-----|---------------+          |                                   
                 |                          |                                   
                 +--> gc->get_direction() --|--- e.g., aspeed_gpio_get_direction
                                            |                                   
```

## <a name="pin-control"></a> Pin Control

Some GPIO pins are multi-functional; they can have one or two extra functionality through proper settings. 
For kernel to ensure these functions work as expected, it introduces the pin control (pinctrl) mechanism to set the required registers beforehand automatically. 
Let say I2C channels one and two are born to be I2C channels, and pinctrl is unnecessary before they work. 
However, I2C channel three is different, and proper settings are essential.

```
i2c-bus@c0 {
    #address-cells = <0x01>;
    #size-cells = <0x00>;
    #interrupt-cells = <0x01>;
    reg = <0xc0 0x40>;
    compatible = "aspeed,ast2500-i2c-bus";
    clocks = <0x02 0x1a>;
    resets = <0x02 0x07>;
    bus-frequency = <0x186a0>;
    interrupts = <0x02>;
    interrupt-parent = <0x1c>;
    pinctrl-names = "default";
    pinctrl-0 = <0x1d>; <========== refer to phandle 0x1d
    status = "okay";
};
```

And descriptor with phandle 0x1d is:

```
i2c3_default {      
    function = "I2C3";
    groups = "I2C3";
    phandle = <0x1d>;
};
```

From file **drivers/pinctrl/aspeed/pinctrl-aspeed-g5.c** we have the below definition:

```                                                                                                
 #define I2C3_DESC   SIG_DESC_SET(SCU90, 16) <========== setting to enable multi-function of the below pins
                                                                                                           
                                                                                                           
                          ball       setting                                                               
                            |           |                                                                  
 #define A11 128            v           v                                                                  
 SIG_EXPR_LIST_DECL_SINGLE(A11, SCL3, I2C3, I2C3_DESC);                                                    
 PIN_DECL_1(A11, GPIOQ0, SCL3);   ^              ^                                                         
                    ^      ^      |              |                                                         
                    |      |   signal         setting                                                      
                  gpio     |                                                                               
               (low prio)  |                                                                               
                           |                                                                               
                         func1                                                                             
                      (high prio)                                                                          
                                                                                                           
 #define A10 129                                                                                           
 SIG_EXPR_LIST_DECL_SINGLE(A10, SDA3, I2C3, I2C3_DESC);                                                    
 PIN_DECL_1(A10, GPIOQ1, SDA3);                                                                            
                                                                                                           
 FUNC_GROUP_DECL(I2C3, A11, A10) <========== declare A11, A10 as group I2C3                                
```

In the above declaration, ball **A11** can only be function 1 (SCL3, high priority) or other (GPIO, low priority). 
In some cases, it equips two extra functions instead of one. 
The I2C channel nine is an excellent example of this illustration.

```
 #define I2C9_DESC   SIG_DESC_SET(SCU90, 22) <========== setting to enable multi-function of the below pins                 
                                                                                                                            
                               signal                                                                                       
                                  |             settings to enable this function                                            
                          ball    |  function   +--------+                                                                  
                            |     |     |       |        |                                                                  
 #define C14 4              v     v     v       v        v                                                                  
 SIG_EXPR_LIST_DECL_SINGLE(C14, SCL9, I2C9, I2C9_DESC, COND1);                                                              
 SIG_EXPR_LIST_DECL_SINGLE(C14, TIMER5, TIMER5, SIG_DESC_SET(SCU80, 4), COND1);                                             
                            ^     ^     ^                  ^              ^                                                 
                            |     |     |                  |              |                                                 
                          ball    |  function              +--------------+                                                 
                                  |                        settings to enable this function                                 
                               signal                                                                                       
                                                                                                                            
                         func1                                                                                              
                      (high prio)                                                                                           
                           |                                                                                                
                           v                                                                                                
 PIN_DECL_2(C14, GPIOA4, SCL9, TIMER5);                                                                                     
                    ^             ^                                                                                         
                    |             |                                                                                         
                  other         func2                                                                                       
              (lowest prio)   (low prio)                                                                                    
                                                                                                                            
 FUNC_GROUP_DECL(TIMER5, C14);    <========== declare that group 'TIMER5' contains ball 'C14'                               
                                                                                                                            
 #define A13 5                                                                                                              
 SIG_EXPR_LIST_DECL_SINGLE(A13, SDA9, I2C9, I2C9_DESC, COND1);                                                              
 SIG_EXPR_LIST_DECL_SINGLE(A13, TIMER6, TIMER6, SIG_DESC_SET(SCU80, 5), COND1);                                             
 PIN_DECL_2(A13, GPIOA5, SDA9, TIMER6);                                                                                     
                                                                                                                            
 FUNC_GROUP_DECL(TIMER6, A13);    <========== declare that group 'TIMER6' contains ball 'A13'                               
                                                                                                                            
 FUNC_GROUP_DECL(I2C9, C14, A13); <========== declare that group 'I2C9' contains ball 'C14' and 'A13'                       
```

It's ok not to expand the above macro into scary expressions and complicated definitions. 
Knowing these relations between pins and functions is enough, and let's turn to the pinctrl initialization during kernel boot up. 
Pinctrl also has its descriptor in the device tree and triggers the probing after matching the driver.

```
Driver:

static const struct of_device_id aspeed_g5_pinctrl_of_match[] = {
    { .compatible = "aspeed,ast2500-pinctrl", }, <========== match
    { .compatible = "aspeed,g5-pinctrl", },
    { },
};
```

```
DTS:

pinctrl@80 {
    compatible = "aspeed,ast2500-pinctrl"; <========== match
    reg = <0x80 0x18 0xa0 0x10>;
    aspeed,external-nodes = <0x08 0x09>;
    phandle = <0x0d>;
```

It's better to register pinctrl device first, otherwise devices that need pinctrl will have to defer to a later probe.

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

<details>
  <summary> Other code tracing </summary>

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
              ball          function1                         
                |               |                             
                |               |                             
 #define B14 0  v               v                             
 SSSF_PIN_DECL(B14, GPIOA0, MAC1LINK, SIG_DESC_SET(SCU80, 0));
                       ^                         ^            
                       |                         |            
                       |                         |            
                     GPIO                     setting         
                                                for           
                                             function1                             
```

```
                                                          setting          
                                                            for            
                               function1                 function1         
                                   |                         |             
                                   |                         |             
                                   v                         v             
 SIG_EXPR_LIST_DECL_SINGLE(D13, SPI1CS1, SPI1CS1, SIG_DESC_SET(SCU80, 15));
 SIG_EXPR_LIST_DECL_SINGLE(D13, TIMER3, TIMER3, SIG_DESC_SET(SCU80, 2));   
                                   ^                         ^             
                                   |                         |             
                                   |                         |             
                               function2                  setting          
                                                            for            
                       (high prio)                       function2         
           ball         function1                                          
             |              |                                              
             |              |                                              
             v              v                                              
 PIN_DECL_2(D13, GPIOA2, SPI1CS1, TIMER3);                                 
                    ^                ^                                     
                    |                |                                     
                    |                |                                     
                  GPIO           function2                                 
              (lowest prio)      (low prio)                                
```

```
// MAC1LINK
static const struct aspeed_sig_desc sig_descs_MAC1LINK_MAC1LINK[] = {
    { ASPEED_IP_SCU, SCU80, BIT_MASK(0), 1, 0 }
};
static const struct aspeed_sig_expr sig_exprs_MAC1LINK_MAC1LINK = {
    .signal = "MAC1LINK",
    .function = "MAC1LINK",
    .ndescs = ARRAY_SIZE(sig_descs_MAC1LINK_MAC1LINK),
    .descs = &sig_descs_MAC1LINK_MAC1LINK[0],
};
static const struct aspeed_sig_expr *sig_exprs_MAC1LINK_MAC1LINK[] = {
    &sig_exprs_MAC1LINK_MAC1LINK, NULL
};
static const struct aspeed_sig_expr *
sig_exprs_0_MAC1LINK[ARRAY_SIZE(sig_exprs_MAC1LINK_MAC1LINK)]
__attribute__((alias(istringify(sig_exprs_MAC1LINK_MAC1LINK))));

// GPIOA0
static const struct aspeed_sig_desc sig_descs_GPIOA0_GPIOA0[] = {
};
static const struct aspeed_sig_expr sig_exprs_GPIOA0_GPIOA0 = {
    .signal = "GPIOA0",
    .function = "GPIOA0",
    .ndescs = ARRAY_SIZE(sig_descs_GPIOA0_GPIOA0),
    .descs = &sig_descs_GPIOA0_GPIOA0[0],
};
static const struct aspeed_sig_expr *sig_exprs_GPIOA0_GPIOA0[] = {
    &sig_exprs_GPIOA0_GPIOA0, NULL
};
static const struct aspeed_sig_expr *
sig_exprs_0_GPIOA0[ARRAY_SIZE(sig_exprs_GPIOA0_GPIOA0)]
__attribute__((alias(istringify(sig_exprs_GPIOA0_GPIOA0))));

// Both
static const struct aspeed_sig_expr **pin_exprs_0[] = {
    sig_exprs_0_MAC1LINK, sig_exprs_0_GPIOA0, NULL
};
static const struct aspeed_pin_desc pin_0 = {
    "0", &pin_exprs_0[0]
};
static const int group_pins_MAC1LINK[] = { 0 };
static const char *func_groups_MAC1LINK[] = { "MAC1LINK" };
```
  
</details>

When drivers and devices are busy matching each other, the pinctrl mechanism goes before the probing function to ensure the device is ready. 
As we mentioned earlier that I2C channel three needs pin control on **I2C3** for its default state. 
The below **create_pinctrl** looks into the device tree for the pinctrl properties, and later **pinctrl_select_state** operates on corresponding settings.

```
+-------------------+                                                                  
| pinctrl_bind_pins |                                                                  
+----|--------------+                                                                  
     |                                                                                 
     |--> allocate dev_pin_info for device                                             
     |                                                                                 
     |    +------------------+                                                         
     |--> | devm_pinctrl_get |                                                         
     |    +----|-------------+                                                         
     |         |                                                                       
     |         |--> allocate pinctrl                                                   
     |         |                                                                       
     |         |    +-------------+                                                    
     |         |--> | pinctrl_get |                                                    
     |         |    +---|---------+                                                    
     |         |        |    +--------------+                                          
     |         |        |--> | find_pinctrl | get pinctrl if it exists already         
     |         |        |    +--------------+                                          
     |         |        |    +----------------+                                        
     |         |        +--> | create_pinctrl | set up pinctrl, add map to pinctrl_maps
     |         |             +----------------+                                        
     |         |    +------------+                                                     
     |         +--> | devres_add | add pinctrl to device as resource                   
     |              +------------+                                                     
     |                                                                                 
     |--> get 'default' state                                                          
     |                                                                                 
     |--> get 'init' state (it's ok not to specify this in dts)                        
     |                                                                                 
     |    +----------------------+                                                     
     +--> | pinctrl_select_state | ensure pinctrl is in the specified state            
          +----------------------+                                                     
```

```
                +----------------+                  +----------------------+
                |can be 'default'|                  |nri1_default {        |
                |       'init'   |                  |    function = "NRI1";|
                |       'sleep'  |                  |    groups = "NRI1";  |
                |       'idle'   |                  |    phandle = <0x17>; |
                +----------------+                  |};                    |
                                                    +----------------------+
                       state                                                
 serial@1e783000 {       |                          config                  
     ...                 v                             |                    
     pinctrl-names = "default";                        v                    
     pinctrl-0 = <0x10 0x11 0x12 0x13 0x14 0x15 0x16 0x17>;                 
 };                                               ^                         
                                                  |                         
                                               config                       
                                                                            
                                      +-----------------------+             
                                      |ndcd1_default {        |             
                                      |    function = "NDCD1";|             
                                      |    groups = "NDCD1";  |             
                                      |    phandle = <0x16>;  |             
                                      |};                     |             
                                      +-----------------------+             
```

<details>
  <summary> Other code tracing </summary>

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
    |--> return if the device needs no pinctrl (most cases)                                                   
    |                                                                                                         
    |--> for each dev-name-matched map in pinctrl_maps                                                        
    |                                                                                                         
    |        +-------------+                                                                                  
    |        | add_setting | ensure 'state' exists and set up 'setting' based on 'map', and add to 'state'    
    |        +-------------+                                                                                  
    |                                                                                                         
    +--> add pinctrl to pinctrl_list     
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
+-------------+                                                                                                               
| add_setting | ensure 'state' exists and set up 'setting' based on 'map', and add to 'state'                                 
+---|---------+                                                                                                               
    |    +------------+                                                                                                       
    |--> | find_state | find the state in pinctrl based on the map name                                                       
    |    +------------+                                                                                                       
    |                                                                                                                         
    |--> if not found                                                                                                         
    |                                                                                                                         
    |         +--------------+                                                                                                
    |         | create_state | create 'state' and add to pinctrl                                                              
    |         +--------------+                                                                                                
    |                                                                                                                         
    |--> allocate and set up 'setting'                                                                                        
    |                                                                                                                         
    |--> if map type is MUX_GROUP                                                                                             
    |                                                                                                                         
    |        +-----------------------+                                                                                        
    |        | pinmux_map_to_setting | save indexes of function and group name to 'setting'                                   
    |        +-----------------------+                                                                                        
    |                                                                                                                         
    |--> else if map type is CONFIGS_PIN or CONFIGS_GROUP                                                                     
    |                                                                                                                         
    |        +------------------------+                                                                                       
    |        | pinconf_map_to_setting | save pin number of matched name to 'setting', copy config info from 'map' to 'setting'
    |        +------------------------+                                                                                       
    |                                                                                                                         
    +--> add 'setting' to 'state'                                                                                             
```
  
```
+------------------------+                                                        
| pinconf_map_to_setting |                                                        
+-----|------------------+                                                        
      |                                                                           
      |--> if type is PIN                                                         
      |                                                                           
      |        +-------------------+                                              
      |        | pin_get_from_name |                                              
      |        +-------------------+                                              
      |                                                                           
      |        save pin number in 'setting'                                       
      |                                                                           
      |--> else if type is GROUP                                                  
      |                                                                           
      |        +----------------------------+                                     
      |        | pinctrl_get_group_selector | get pin number of matched group name
      |        +----------------------------+                                     
      |                                                                           
      |        save pin number in 'setting'                                       
      |                                                                           
      +--> copy config info from 'map' to 'setting'                               
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
  
</details>

## <a name="system-startup"></a> System Startup

```
gpiolib_dev_init: register bus & driver, reserve dev#
gpiolib_sysfs_init: register 'gpio' class, for each entry@gpio_devices: create device and register to sysfs
gpiolib_debugfs_init: prepare '/sys/kernel/debug/gpio' with fops=gpiolib_fops
aspeed_gpio_probe: set up aspeed_gpio, install ops, prepare gpio line descriptors, register gpio chip
aspeed_sgpio_driver_init: (skip, no matched device)
gpio_clk_driver_init: (skip, no matched device)
gpio_keys_polled_driver_init: (skip, no matched device)
i2c_mux_gpio_driver_init: (skip, no matched device)
w1_gpio_driver_init: (skip, no matched device)
gpio_led_driver_init:
fsi_master_gpio_driver_init:
gpio_keys_init:
```

```
+------------------+                                                                                 
| gpiolib_dev_init | : register bus & driver, reserve dev#
+----|-------------+                                                                                 
     |    +--------------+                                                                           
     |--> | bus_register | register 'gpio_bus_type'                                                  
     |    +--------------+                                                                           
     |    +-----------------+                                                                        
     |--> | driver_register | register 'gpio_stub_drv' (not important in our case)                   
     |    +-----------------+                                                                        
     |    +---------------------+                                                                    
     |--> | alloc_chrdev_region | reserve a range (256) of minor numbers for gpio character devices  
     |    +---------------------+                                                                    
     |                                                                                               
     |--> gpiolib_initialized = true                                                                 
     |                                                                                               
     |    +---------------------+                                                                    
     +--> | gpiochip_setup_devs | register each gpiochip dev in 'gpio_devices' (empty in in our case)
          +---------------------+                                                                    
```

```                                                                                        
drivers/gpio/gpiolib-sysfs.c                                                                                   
+--------------------+                                                                                          
| gpiolib_sysfs_init | : register 'gpio' class, for each entry@gpio_devices: create device and register to sysfs
+-|------------------+                                                                                          
  |    +----------------+                                                                                       
  |--> | class_register | register 'gpio' class                                                                 
  |    +----------------+                                                                                       
  |                                                                                                             
  +--> for each entry on 'gpio_devices'                                                                         
       |                                                                                                        
       |    +-------------------------+                                                                         
       +--> | gpiochip_sysfs_register | create device and register to sysfs                                     
            +-------------------------+                                                                         
```

```
drivers/gpio/gpiolib.c                                                           
+----------------------+                                                          
| gpiolib_debugfs_init | : prepare '/sys/kernel/debug/gpio' with fops=gpiolib_fops
+----------------------+                                                          
```


```
drivers/gpio/gpio-aspeed.c
+-------------------+                                                                                                    
| aspeed_gpio_probe | : set up aspeed_gpio, install ops, prepare gpio line descriptors, register gpio chip
+----|--------------+                                                                                                    
     |                                                                                                                   
     |--> set up 'aspeed_gpio' and install gpio chip operations (in, out, get, set, ...)                                 
     |                                                                                                                   
     |--> set up 'irq_chip' of 'aspeed_gpio' if device tree specifies the irq#                                           
     |                                                                                                                   
     |    +------------------------+                                                                                     
     +--> | devm_gpiochip_add_data | prepare desc for each gpio line, set up irq desc, register gpio chip 
          +------------------------+ (connect the generic layer (gpio lib) to specific driver (aspeed))
```

```
include/linux/gpio/driver.h
+------------------------+
| devm_gpiochip_add_data | : prepare desc for each gpio line, set up irq desc, register gpio chip
+-----|------------------+
      |    +---------------------------------+
      +--> | devm_gpiochip_add_data_with_key | : prepare desc for each gpio line, set up irq desc, register gpio chip
           +--------|------------------------+
                    |    +----------------------------+
                    +--> | gpiochip_add_data_with_key | prepare desc for each gpio line, set up irq desc, register gpio chip
                    |    +----------------------------+
                    |    +--------------------------+
                    +--> | devm_add_action_or_reset | add action 'devm_gpio_chip_release' as device resource
                         +--------------------------+
```

```
drivers/gpio/gpiolib.c
+----------------------------+
| gpiochip_add_data_with_key | : prepare desc for each gpio line, set up irq desc, register gpio chip
+------|---------------------+
       |
       |--> set up 'gpio device'
       |
       |--> allocate 'descriptor' for each GPIO line
       |
       |    +---------------------+
       |--> | gpiodev_add_to_list | add 'gpio device' to the 'gpio_devices' list
       |    +---------------------+
       |    +-----------------+
       |--> | of_gpiochip_add | add pin range and set hog (default gpio behavior)
       |    +-----------------+
       |    +----------------------+
       |--> | gpiochip_add_irqchip | set up target 'irq desc' based on the 'gpio chip'
       |    +----------------------+
       |
       +--> if gpiolib_initialized is set (yes, in gpio lib init)
            |
            |    +--------------------+
            +--> | gpiochip_setup_dev | register cdev
                 +--------------------+
```

## <a name="cheat-sheet"></a> Cheat Sheet

```
cd /sys/class/gpio
echo 123 > export
cd gpio123
cat direction # check current direction
echo out > direction
cat value
```

```
cat /sys/kernel/debug/gpio
```

## <a name="reference"></a> Reference

- [Linux kernel GPIO user space interface](https://embeddedbits.org/new-linux-kernel-gpio-user-space-interface/)
