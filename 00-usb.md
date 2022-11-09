> The note is based on Linux version 5.15.0 in OpenBMC.

## Index

- [Introduction](#introduction)
- [Framework](#framework)
- [Virtual Hub](#virtual-hub)
- [System Startup](#system-startup)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

(TBD)

## <a name="framework"></a> Framework

```
struct usb_driver {
    const char *name;                                   // driver name (must be unique)
    int (*probe) (struct usb_interface *intf,
              const struct usb_device_id *id);          // check if (device, driver) match
    void (*disconnect) (struct usb_interface *intf);    // work with usb interface
    struct usbdrv_wrap drvwrap;                         // usb driver wrapper
};
```

```
struct usbdrv_wrap {
    struct device_driver driver;    // driver
    int for_devices;                // 0: interface driver, else: device driver
};
```

```
struct usb_device_id {
    __u16       match_flags;    // specifies which fields to compare
};
```

```
struct usb_device {
    int     devnum; // unique device number
    char        devpath[16];        // dev position in usb tree topology
    enum usb_device_state   state;  // e.g., attached, configured, ...
    enum usb_device_speed   speed;  // e.g., low, full, high, ...
    struct usb_device *parent;      // points to usb huuub
    struct usb_bus *bus;            // point to usb_bus
    struct device dev;              // generic device model
    struct usb_device_descriptor descriptor;    // more detailed dev data
    struct usb_host_config *config;             // list of possible configs
    struct usb_host_config *actconfig;          // poitns current working config
    char *product;      // hw info
    char *manufacturer; // hw info
    char *serial;       // hw info
    int maxchild;       // if self is a usb hub, this specifies how many ports it has
};
```

```
struct usb_bus {
    struct device *controller;  // points to controller
    struct device *sysdev;      /* as seen from firmware or bus */
    int busnum;             // unique bus number
    const char *bus_name;   // unique bus name
    u8 uses_pio_for_control;    /*
                     * Does the host controller use PIO
                     * for control transfers?
                     */
    u8 otg_port;            /* 0, or number of OTG/HNP port */
    unsigned is_b_host:1;       /* true during some HNP roleswitches */
    unsigned b_hnp_enable:1;    /* OTG: did A-Host enable HNP? */
    unsigned no_stop_on_short:1;    /*
                     * Quirk: some controllers don't stop
                     * the ep queue on a short transfer
                     * with the URB_SHORT_NOT_OK flag set.
                     */
    unsigned no_sg_constraint:1;    /* no sg constraint */
    unsigned sg_tablesize;      /* 0 or largest number of sg list entries */

    int devnum_next;        /* Next open device number in
                     * round-robin allocation */
    struct mutex devnum_next_mutex; /* devnum_next mutex */

    struct usb_devmap devmap;   // bitmap for dev# tracking
    struct usb_device *root_hub;    // points to root hub
    struct usb_bus *hs_companion;   /* Companion EHCI bus, if any */

    int bandwidth_allocated;    /* on this bus: how much of the time
                     * reserved for periodic (intr/iso)
                     * requests is used, on average?
                     * Units: microseconds/frame.
                     * Limits: Full/low speed reserve 90%,
                     * while high speed reserves 80%.
                     */
    int bandwidth_int_reqs;     /* number of Interrupt requests */
    int bandwidth_isoc_reqs;
```

```
+-------------------+                                         
| usb_ep_autoconfig | : get an available ep and set up        
+-|-----------------+                                         
  |    +----------------------+                               
  +--> | usb_ep_autoconfig_ss | get an available ep and set up
       +----------------------+                               
```

```
+----------------------+                                                                  
| usb_ep_autoconfig_ss | : get an available ep and set up                                 
+-|--------------------+                                                                  
  |                                                                                       
  |--> if gadget has ->match_ep                                                           
  |                                                                                       
  +------> call ->match_ep(), e.g.,                                                       
  |        +-----------------------+                                                      
  |        | ast_vhub_udc_match_ep | ensure we have a ep (find a matched one and alloc it)
  |        +-----------------------+                                                      
  |                                                                                       
  |------> go to 'found_ep' if found                                                      
  |                                                                                       
  |--> for each ep on list of gadget                                                      
  |                                                                                       
  |        +--------------------------+                                                   
  |------> | usb_gadget_ep_match_desc | check if endpoint and descriptor match each other 
  |        +--------------------------+                                                   
  |                                                                                       
  |------> go to 'found_ep' if found                                                      
  |found_ep                                                                               
  |--> determine addr and save in descriptor                                              
  |                                                                                       
  +--> set up ep                                                                          
```

```
+-----------------------+                                                                                          
| ast_vhub_udc_match_ep | : ensure we have a ep (find a matched one and alloc it)                                  
+-|---------------------+                                                                                          
  |                                                                                                                
  |--> for each ep on gadget                                                                                       
  |                                                                                                                
  |        +--------------------------+                                                                            
  |------> | usb_gadget_ep_match_desc | check if endpoint and descriptor match each other                          
  |        +--------------------------+                                                                            
  |                                                                                                                
  |------> return ep if match is found                                                                             
  |                                                                                                                
  |--> determine max speed based on type (control, isochronous, bulk, interrupt)                                   
  |                                                                                                                
  |--> traverse vhub to find an used endpoint                                                                      
  |                                                                                                                
  |    +--------------------+                                                                                      
  +--> | ast_vhub_alloc_epn | get an unused ep from vhub, set up (install ops, alloc buffer), add to list of gadget
       +--------------------+                                                                                      
```

```
+--------------------+                                                                                        
| ast_vhub_alloc_epn | : get an unused ep from vhub, set up (install ops, alloc buffer), add to list of gadget
+-|------------------+                                                                                        
  |                                                                                                           
  |--> get an unused endpoint from vhub                                                                       
  |                                                                                                           
  |--> set up endpoint                                                                                        
  |                                                                                                           
  |--> install ep ops 'ast_vhub_epn_ops'                                                                      
  |                                                                                                           
  |    +--------------------+                                                                                 
  |--> | dma_alloc_coherent |                                                                                 
  |    +--------------------+                                                                                 
  |    +---------------+                                                                                      
  +--> | list_add_tail | add ep to list of gadget                                                             
       +---------------+                                                                                      
```

```
+--------------------------+                                                    
| usb_gadget_ep_match_desc | : check if endpoint and descriptor match each other
+-|------------------------+                                                    
  |                                                                             
  |--> get type and max_packet of descriptor                                    
  |                                                                             
  |--> check direction, packet limit, and speed                                 
  |                                                                             
  +--> chec type (control, isochronous, bulk, interrupt)                        
```

```
+-----------------------+                                                                       
| composite_dev_prepare | : prepare req for composite, assign composite to gadget ep0 as private
+-|---------------------+                                                                       
  |    +----------------------+                                                                 
  |--> | usb_ep_alloc_request | alloc request                                                   
  |    +----------------------+                                                                 
  |                                                                                             
  |--> assign req to composite dev                                                              
  |                                                                                             
  |--> alloc buffer for req                                                                     
  |                                                                                             
  |    +--------------------+                                                                   
  |--> | device_create_file | create files under /sys/                                          
  |    +--------------------+                                                                   
  |                                                                                             
  |--> gadget ep0 private = composite                                                           
  |                                                                                             
  |    +-------------------------+                                                              
  +--> | usb_ep_autoconfig_reset | reset ep state                                               
       +-------------------------+                                                              
```

```
+----------------------+                   
| usb_ep_alloc_request | : alloc request   
+-|--------------------+                   
  |                                        
  +--> call ->alloc_request, e.g.,         
       +------------------------+          
       | ast_vhub_alloc_request | alloc req
       +------------------------+          
```

```
+---------------------+                                            
| usb_gstrings_attach | : duplicate strings and attach to composite
+|--------------------+                                            
 |    +---------------------+                                      
 |--> | copy_gadget_strings | alloc buffer and copy strings to it  
 |    +---------------------+                                      
 |    +---------------+                                            
 +--> | list_add_tail | append copied strings to list in composite 
      +---------------+                                            
```

```
+------------------------+                                                    
| gether_register_netdev | : set mac to net_dev and register it, clear carrier
+-|----------------------+                                                    
  |    +-----------------+                                                    
  |--> | eth_hw_addr_set | assign mac from dev to net_dev                     
  |    +-----------------+                                                    
  |    +-----------------+                                                    
  |--> | register_netdev | register net_dev                                   
  |    +-----------------+                                                    
  |                                                                           
  |--> print host and dev mac?                                                
  |                                                                           
  |    +-------------------+                                                  
  +--> | netif_carrier_off | clear carrier                                    
       +-------------------+                                                  
```

## <a name="virtual-hub"></a> Virtual Hub

```
+--------------+
| ast_vhub_irq | : handle vhub irq
+-|------------+
  |    +--------+
  |--> | writel | ack interrupt
  |    +--------+
  |
  |--> if there's interrupt for endpoint
  |
  |------> for each involved endpoint
  |
  |            +----------------------+
  |----------> | ast_vhub_epn_ack_irq | ack epn irq
  |            +----------------------+
  |
  |--> if there's interrupt for port
  |
  |------> for each port
  |
  |            +------------------+
  |----------> | ast_vhub_dev_irq | handle dev irq
  |            +------------------+
  |
  |--> if there's interrupt for top-level vhub ep0
  |
  |        +-------------------------+
  |------> | ast_vhub_ep0_handle_ack | given ep0 state, handle the 1st req on list accordingly (send or receive)
  |        +-------------------------+
  |         or
  |        +---------------------------+
  |------> | ast_vhub_ep0_handle_setup | handle req propery, pass to gadget driver if needed
  |        +---------------------------+
  |
  |--> if there's interrupt for top-level bus events
  |
  |        +---------------------+
  |------> | ast_vhub_hub_resume | for each port, call its driver->resume()
  |        +---------------------+
  |         or
  |        +----------------------+
  |------> | ast_vhub_hub_suspend | for each port, call its driver->suspend()
  |        +----------------------+
  |         or
  |        +--------------------+
  +------> | ast_vhub_hub_reset | for each port, call its driver->suspend(), and write hw reg to reset
           +--------------------+
```

```
+----------------------+
| ast_vhub_epn_ack_irq | : ack epn irq
+-|--------------------+
  |
  |--> if the ep is in desc mode
  |
  |        +------------------------------+
  |------> | ast_vhub_epn_handle_ack_desc | handle ack desc
  |        +------------------------------+
  |
  |--> else
  |
  |        +-------------------------+
  +------> | ast_vhub_epn_handle_ack | handle ack
           +-------------------------+              
```

```
+------------------------------+
| ast_vhub_epn_handle_ack_desc | : handle ack desc
+-|----------------------------+
  |
  |--> get 'last' from hardware register
  |
  |    +--------------------------+
  |--> | list_first_entry_or_null | get the first req from queue of ep
  |    +--------------------------+
  |
  |--> while ep last != 'last'
  |                                                                req
  |------> get desc from ep last                                 +------+
  |                                                              |  +------+
  |------> ep last++                                             |  | desc |
  |                                                              |  +------+
  |------> continue if it's not an active req                    |  | desc |
  |                                                              |  +------+
  |------> if current desc is the last specified in req          |  | desc |
  |                                                              +--+------+
  |            +---------------+
  |----------> | ast_vhub_done | finalize req
  |            +---------------+
  |            +--------------------------+
  |----------> | list_first_entry_or_null | get next req from list (in case somewhere adds it suddenly?)
  |            +--------------------------+
  |
  +----------> break
  |
  |--> if there's more work (req)
  |
  |        +------------------------+
  +------> | ast_vhub_epn_kick_desc | given req, set up descripters, kick hardware to process
           +------------------------+ 
```

```
+---------------+
| ast_vhub_done | : finalize req
+-|-------------+
  |    +---------------+
  |--> | list_del_init | remove req from list
  |    +---------------+
  |
  |--> if req has used dma
  |
  |        +---------------------------------+
  |------> | usb_gadget_unmap_request_by_dev |
  |        +---------------------------------+
  |
  |--> if the req isn't internal
  |
  |        +-----------------------------+
  +------> | usb_gadget_giveback_request | control led and call ->complete(), e.g.,
           +-----------------------------+ composite_setup_complete()
```

```
+------------------------+
| ast_vhub_epn_kick_desc | : given req, set up descripters, kick hardware to process
+-|----------------------+
  |
  |--> lable 'active' on req
  |
  |--> return if the last desc in req is handled
  |
  |--> while we can still create desc and req needs one
  |
  |------> get next free desc from ep
  |
  |------> ep next++
  |
  |------> calculate chunk size
  |
  |------> populate desc (save buf addr in desc)
  |
  |------> save chunk size in desc
  |
  |------> if it's the end of req or we don't have enough desc
  |
  |----------> set 'interrupt' bit in desc
  |
  |    +--------+
  +--> | writel | kick hardware to process descriptors
       +--------+
```

```
 +-------------------------+
 | ast_vhub_epn_handle_ack | : handle ack
 +-|-----------------------+
   |    +-------+
   |--> | readl | read ep status
   |    +-------+
   |    +--------------------------+
   |--> | list_first_entry_or_null | get the 1st request
   |    +--------------------------+
   |
   |--> return if no req
   |
   |--> go to 'next chunk' if req isn't active
   |
   |--> clear 'active' on req
   |
   |--> prepare data buffer if needed, and adjust size
   |
   |--> label 'last' on req if it's a short packet
   |
   |--> if no other desc needs handling
   |
   |        +---------------+
   |------> | ast_vhub_done | finalize req
   |        +---------------+
   |        +--------------------------+
   |------> | list_first_entry_or_null | pick up next req
   |        +--------------------------+
   |
   |------> return if nothing to do
   |next chunk
   |    +-------------------+
   +--> | ast_vhub_epn_kick | prepare buf, save addr to hw reg, label 'active' on req, and kick and hw
        +-------------------+
```

```
+-------------------+
| ast_vhub_epn_kick | : prepare buf, save addr to hw reg, label 'active' on req, and kick and hw
+-|-----------------+
  |
  |--> calculate chunk size
  |
  |--> prepare buffer and write addr to hw register
  |
  |--> label 'active' on req (packet on the fly)
  |
  |    +--------+
  +--> | writel | kick the hw
       +--------+
```

```
+------------------+
| ast_vhub_dev_irq | : handle dev irq
+-|----------------+
  |    +-------+
  |--> | readl | read isr status
  |    +-------+
  |    +--------+
  |--> | writel | ack interrupt
  |    +--------+
  |
  |--> if 'epo in ack stall'
  |
  |        +-------------------------+
  |------> | ast_vhub_ep0_handle_ack | given ep0 state, handle the 1st req on list accordingly (send or receive)
  |        +-------------------------+
  |
  |--> if 'epo out ack stall'
  |
  |        +-------------------------+
  |------> | ast_vhub_ep0_handle_ack | given ep0 state, handle the 1st req on list accordingly (send or receive)
  |        +-------------------------+
  |
  |--> if 'ep0 setup'
  |
  |        +---------------------------+
  +------> | ast_vhub_ep0_handle_setup | handle req propery, pass to gadget driver if needed
           +---------------------------+                
```

```
+-------------------------+
| ast_vhub_ep0_handle_ack | : given ep0 state, handle the 1st req on list accordingly (send or receive)
+-|-----------------------+
  |    +-------+
  |--> | readl | read ep0 status
  |    +-------+
  |    +--------------------------+
  |--> | list_first_entry_or_null | get the first req
  |    +--------------------------+
  |
  |--> switch state
  |
  |--> case token
  |------> if req (shouldn't happen)
  |            +---------------+
  |----------> | ast_vhub_nuke | finalize all req on ep
  |            +---------------+
  +----------> stall = true
  |
  |--> case data
  |------> if no req (should have)
  |----------> stall = true
  |----------> break
  |------> if direction is 'in'
  |            +----------------------+
  |----------> | ast_vhub_ep0_do_send | copy from req to ep buffer, and trigger send
  |            +----------------------+
  |------> else ('out')
  |            +-------------------------+
  |----------> | ast_vhub_ep0_do_receive | copy from ep buffer to req, trigger 'receive' or 'send' accordingly
  |            +-------------------------+
  |
  |--> case status
  +------> if req (shouldn't happen)
  |            +---------------+
  |            | ast_vhub_nuke | finalize all req on ep
  |            +---------------+
  |------> if direction and ack mismatch
  |----------> stall = true
  |
  |--> case stall
  |        +---------------+
  |------> | ast_vhub_nuke | finalize all req on ep
  |        +---------------+
  |
  |--> if stall
  |
  |        +--------+
  +------> | writel | stall the ep
           +--------+
```

```
+---------------+
| ast_vhub_nuke | : finalize all req on ep
+-|-------------+
  |
  |--> while ep queue still has something
  |
  |        +------------------+
  |------> | list_first_entry | get the 1st req
  |        +------------------+
  |        +---------------+
  +------> | ast_vhub_done | finalize req
           +---------------+
```

```
+----------------------+
| ast_vhub_ep0_do_send | : copy from req to ep buffer, and trigger send
+-|--------------------+
  |
  |--> label 'last_desc' if it's the status from gadget
  |
  |--> if we're done (can receive)
  |
  |        +--------+
  |------> | writel | write 'rx buff ready' to hw reg
  |        +--------+
  |        +---------------+
  |------> | ast_vhub_done | finalize req
  |        +---------------+
  |
  +------> return
  |
  |--> decide chunk size
  |
  |    +--------+
  |--> | memcpy | copy data from req to ep buffer
  |    +--------+
  |    +--------+
  +--> | writel | save chunck size and trigger send
       +--------+ 
```

```
+-------------------------+
| ast_vhub_ep0_do_receive | : copy from ep buffer to req, trigger 'receive' or 'send' accordingly
+-|-----------------------+
  |
  |--> calculate remaining space
  |
  |--> copy data from ep buffer to request
  |
  |--> update 'actual' size
  |
  |--> if we're done (no more to receive)
  |
  |        +--------+
  |------> | writel | write 'tx buf ready'
  |        +--------+
  |        +---------------+
  |------> | ast_vhub_done | finalize req
  |        +---------------+
  |
  |--> else
  |
  |        +-----------------------+
  +------> | ast_vhub_ep0_rx_prime | write 'rx buf ready'
           +-----------------------+
```

```
+---------------------------+
| ast_vhub_ep0_handle_setup | : handle req propery, pass to gadget driver if needed
+-|-------------------------+
  |
  |--> copy setup packet from 'ep0 setup'
  |
  |--> set ep0 state and direction
  |
  |--> if ep has no device yet
  |
  |------> if req type is standard
  |
  |            +--------------------------+
  |----------> | ast_vhub_std_hub_request | handle the request properly
  |            +--------------------------+
  |
  |------> elif req type is class
  |
  |            +----------------------------+
  |----------> | ast_vhub_class_hub_request | handle the request properly
  |            +----------------------------+
  |
  |--> elif req type is statndard
  |
  |        +--------------------------+
  |------> | ast_vhub_std_dev_request | handle the request properly
  |        +--------------------------+
  |
  |--> return if rc == data
  |
  |--> if rc == driver (pass req to drvier)
  |
  |------> call ->setup()
  |
  |--> elif rc == stall
  |
  |        +--------+
  |------> | writel | write 'stall'
  |        +--------+
  |
  |--> elif rc == complete
  |
  |        +--------+
  +------> | writel | write 'tx buf ready'
           +--------+ 
```

```
+--------------------------+
| ast_vhub_std_hub_request | : handle the request properly
+-|------------------------+
  |
  |--> get value/index/length from ctrl req
  |
  |--> determine vhub speed
  |
  |--> switch req
  |
  |--> case 'set address': blabla
  |
  |--> case 'get status': blabla
  |
  |--> case 'set/clear feature': blabla
  |
  |--> case 'get/set configuration': blabla
  |
  |--> case 'get descriptor': blabla
  |
  +--> case 'get/set interface': blabla
```

```
+----------------------------+
| ast_vhub_class_hub_request | : handle the request properly
+-|---- ---------------------+
  |
  |--> get value/index/length from ctrl req
  |
  |--> switch req
  |
  |--> case 'get status': blabla
  |
  |--> case 'get/set hub feature': blabla
  |
  |--> case 'get/set port feature': blabla
  |
  +--> case 'tt?': blabla
```

```
+--------------------------+
| ast_vhub_std_dev_request | : handle the request properly
+-|------------------------+
  |
  |--> get value and index from 'ctrl req'
  |
  |--> switch req
  |
  |--> case 'set address': blabla
  |
  |--> case 'get status': blabla
  |
  +--> case 'set/clear feature': blabla
```

```
+--------------+                                                                                   
| gadgets_make | : alloc gadget_info, set up groups, composite & dev, install driver               
+-|------------+                                                                                   
  |                                                                                                
  |--> alloc gadget_info                                                                           
  |                                                                                                
  |    +-----------------------------+                                                             
  |--> | config_group_init_type_name | set up root group right under /sys/kernel/config/usb_gadget/
  |    +-----------------------------+                                                             
  |    +-----------------------------+                                         +---------------+   
  |--> | config_group_init_type_name | set up 'functions' group                | function_make |   
  |    +-----------------------------+                                         +---------------+   
  |    +----------------------------+                                                              
  |--> | configfs_add_default_group | add 'functions' group to root group                          
  |    +----------------------------+                                                              
  |    +-----------------------------+                                         +------------------+
  |--> | config_group_init_type_name | set up 'configs' group                  | config_desc_make |
  |    +-----------------------------+                                         +------------------+
  |    +----------------------------+                                                              
  |--> | configfs_add_default_group | add 'configs' group to root group                            
  |    +----------------------------+                                                              
  |    +-----------------------------+                                                             
  |--> | config_group_init_type_name | set up 'strings' group                                      
  |    +-----------------------------+                                                             
  |    +----------------------------+                                                              
  |--> | configfs_add_default_group | add 'strings' group to root group                            
  |    +----------------------------+                                                              
  |    +-----------------------------+                                         +--------------+    
  |--> | config_group_init_type_name | set up 'os_desc' group                  | os_desc_link |    
  |    +-----------------------------+                                         +--------------+    
  |    +----------------------------+                                                              
  |--> | configfs_add_default_group | add 'os_desc' group to root group                            
  |    +----------------------------+                                                              
  |                                                                                                
  |--> set up 'composite'                                                                          
  |                                                                                                
  |--> set desc type as 'device' (not config, not interface, not endpoint...)                      
  |                                                                                                
  |--> set up 'composite dev'                                                                      
  |                                                                                                
  +--> gadget_driver = configfs_driver_template                                                    
```

```
+---------------+                                                                       
| function_make | : given name, alloc instance and set name, add instance to gadget_info
+-|-------------+                                                                       
  |                                                                                     
  |--> prepare name buf                                                                 
  |                                                                                     
  |    +---------------------------+                                                    
  |--> | usb_get_function_instance | find arg-matched func_driver to alloc instance     
  |    +---------------------------+                                                    
  |    +----------------------+                                                         
  |--> | config_item_set_name | set item name                                           
  |    +----------------------+                                                         
  |                                                                                     
  |--> get gadget_info from arg group                                                   
  |                                                                                     
  |    +---------------+                                                                
  +--> | list_add_tail | append func_instance to gadget_info                            
       +---------------+                                                                
```

```
+---------------------------+                                                          
| usb_get_function_instance | : find arg-matched func_driver to alloc instance         
+-|-------------------------+                                                          
  |    +-------------------------------+                                               
  |--> | try_get_usb_function_instance | find arg-matched func_driver to alloc instance
  |    +-------------------------------+                                               
  |                                                                                    
  +--> return if it's successful                                                       
```

```
+-------------------------------+                                                 
| try_get_usb_function_instance | : find arg-matched func_driver to alloc instance
+-|-----------------------------+                                                 
  |                                                                               
  |--> for each func_driver on 'func_list'                                        
  |                                                                               
  |------> continue if arg name mismatches driver name                            
  |                                                                               
  |------> ->alloc_inst(), e.g.,                                                  
  |        +-----------------+                                                    
  |        | hidg_alloc_inst | prepare opts                                       
  |        +-----------------+                                                    
  |                                                                               
  |------> save func_driver in func_instance                                      
  |                                                                               
  +------> break                                                                  
```

```
+------------------+                                                                                     
| config_desc_make | : prepare config and create items of name and 'strings', add config to composite dev
+-|----------------+                                                                                     
  |                                                                                                      
  |--> get gadget_info from arg group                                                                    
  |                                                                                                      
  |--> prepare name buf                                                                                  
  |                                                                                                      
  |--> alloc config and set up (e.g., config value)                                                      
  |                                                                                                      
  |    +-----------------------------+                                  +---------------------+          
  |--> | config_group_init_type_name | create item of arg name          | config_usb_cfg_link |          
  |    +-----------------------------+                                  +---------------------+          
  |    +-----------------------------+                                                                   
  |--> | config_group_init_type_name |  create item 'strings'                                            
  |    +-----------------------------+                                                                   
  |    +----------------------------+                                                                    
  |--> | configfs_add_default_group | add string group to root group                                     
  |    +----------------------------+                                                                    
  |    +---------------------+                                                                           
  +--> | usb_add_config_only | ensure config is in composite dev                                         
       +---------------------+                                                                           
```

```
+---------------------+                                              
| config_usb_cfg_link | : alloc function and append to config        
+-|-------------------+                                              
  |                                                                  
  |--> check if func_inst comes from the gadget (it should)          
  |                                                                  
  |    +------------------+                                          
  |--> | usb_get_function | get func_driver and call its ->alloc_func
  |    +------------------+                                          
  |    +---------------+                                             
  +--> | list_add_tail | append the allocated function to config     
       +---------------+                                             
```

```
+---------------------+                                           
| usb_add_config_only | : ensure config is in composite dev       
+-|-------------------+                                           
  |                                                               
  |--> return error if no config value                            
  |                                                               
  |--> for each composite config                                  
  |                                                               
  |------> return error if there's other config has the same value
  |                                                               
  |    +---------------+                                          
  +--> | list_add_tail | append config to composite dev           
       +---------------+                                          
```

```
+--------------+                                                   
| os_desc_link | : find target usb config and save in composite dev
+-|------------+                                                   
  |                                                                
  |--> find target config from composite dev                       
  |                                                                
  +--> save its usb config in composite dev                        
```

## <a name="system-startup"></a> System Startup

```
usb_common_init         : create /sys/kernel/debug/usb
usb_init                : register bus, notifier, intf driver 'usbfs_driver' & 'hub_driver', dev driver 'usb_generic_driver'
usb_udc_init            : register 'udc' class
ehci_hcd_init           : create /sys/kernel/debug/usb/ehci
ehci_platform_init      : determine 'ehci_platform_hc_driver', and register 'ehci_platform_driver'
usb_storage_driver_init : init template and register 'usb_storage_driver'
usb_serial_init         : prepare tty driver, register bus 'usb-serial'
usb_serial_module_init  : register usb intf driver, register arg drivers to bus 'usb-serial'
gadget_cfs_init         : init gadget_subsys and prepare config_fs for it
ast_vhub_driver_init    : register ast vhub driver
ecmmod_init             : disabled by default (CONFIG_USB_CONFIGFS_ECM)
gethmod_init            : disabled by default (CONFIG_USB_CONFIGFS_ECM_SUBSET)
rndismod_init           : disabled by default (CONFIG_USB_CONFIGFS_RNDIS)
mass_storagemod_init    : set up function driver by args, register the function driver
ffsmod_init             : disabled by default (CONFIG_USB_CONFIGFS_F_FS)
hidmod_init             : set up function driver by args, register the function driver
hid_init                : register bus and create /sys/kernel/debug/hid/
hid_generic_init        : register hid driver 'hid_generic', trigger the match between dev/drv
hid_init                : init quirks, register 'hid_driver'
```

```
+-----------------+                                         
| usb_common_init | : create /sys/kernel/debug/usb          
+-|---------------+                                         
  |    +--------------------+                               
  |--> | debugfs_create_dir | /sys/kernel/debug/usb?        
  |    +--------------------+                               
  |    +------------------+                                 
  +--> | ledtrig_usb_init | do nothing bc of disabled config
       +------------------+                                                           
```

```
+----------+                                                                                                     
| usb_init | : register bus, notifier, intf driver 'usbfs_driver' & 'hub_driver', dev driver 'usb_generic_driver'
+-|--------+                                                                                                     
  |    +-------------------+                                                                                     
  |--> | usb_init_pool_max | init pool_max = 64 in our case                                                      
  |    +-------------------+                                                                                     
  |    +------------------+                                                                                      
  |--> | usb_debugfs_init | create /sys/kernel/debug/usb/devices                                                 
  |    +------------------+                                                                                      
  |    +-------------------+                                                                                     
  |--> | usb_acpi_register | do nothing bc of disabled config                                                    
  |    +-------------------+                                                                                     
  |    +--------------+                                                                                          
  |--> | bus_register | 'usb_bus_type'                                                                           
  |    +--------------+                                                                                          
  |    +-----------------------+                                                                                 
  |--> | bus_register_notifier | register notifier for later file addition or removal under /sys/                
  |    +-----------------------+                                                                                 
  |    +----------------+                                                                                        
  |--> | usb_major_init | prepare and register cdev                                                              
  |    +----------------+                                                                                        
  |    +--------------+                                                                                          
  |--> | usb_register | register usb interface driver 'usbfs_driver'                                             
  |    +--------------+                                                                                          
  |    +----------------+                                                                                        
  |--> | usb_devio_init | init usb dev io                                                                        
  |    +----------------+                                                                                        
  |    +--------------+                                                                                          
  |--> | usb_hub_init | register usb interface driver 'hub_driver'                                               
  |    +--------------+                                                                                          
  |    +----------------------------+                                                                            
  +--> | usb_register_device_driver | register usb device driver 'usb_generic_driver' and probe devices          
       +----------------------------+                                                                            
```

```
+--------------+                                                
| usb_register | : register usb interface driver                
+-|------------+                                                
  |    +---------------------+                                  
  +--> | usb_register_driver | : register usb interface driver  
       +-|-------------------+                                  
         |                                                      
         |--> set up arg driver                                 
         |                                                      
         |    +-----------------+                               
         |--> | driver_register |                               
         |    +-----------------+                               
         |    +------------------------+                        
         |--> | usb_create_newid_files | create files under /sys
         |    +------------------------+                        
         |                                                      
         +--> print "%s: registered new interface driver %s\n"  
```

```
+----------------+                                             
| usb_devio_init | : init usb dev io                           
+-|--------------+                                             
  |    +------------------------+                              
  |--> | register_chrdev_region | reserve a range of dev_t     
  |    +------------------------+                              
  |    +-----------+                                           
  |--> | cdev_init | usbdev_file_operations                    
  |    +-----------+                                           
  |    +----------+                                            
  |--> | cdev_add | register cdev                              
  |    +----------+                                            
  |    +---------------------+                                 
  +--> | usb_register_notify | register notifer for dev removal
       +---------------------+                                 
```

```
+--------------+                                                  
| usb_hub_init | : register usb interface driver 'hub_driver'     
+-|------------+                                                  
  |    +--------------+                                           
  |--> | usb_register | register usb interface driver 'hub_driver'
  |    +--------------+                                           
  |    +-----------------+                                        
  +--> | alloc_workqueue | "usb_hub_wq"                           
       +-----------------+                                        
```

```
+----------------------------+                                                                 
| usb_register_device_driver | : register usb device driver and probe devices                  
+-|--------------------------+                                                                 
  |                                                                                            
  |--> set up arg new_udriver                                                                  
  |                                                                                            
  |    +-----------------+                                                                     
  |--> | driver_register |                                                                     
  |    +-----------------+                                                                     
  |                                                                                            
  +--> print "%s: registered new device driver %s\n"                                           
  |                                                                                            
  |--> for each device in usb bus                                                              
  |                                                                                            
  |        +---------------------------+                                                       
  +------> | __usb_bus_reprobe_drivers | check if driver and device match (reprobe is possible)
           +---------------------------+                                                       
```

```
+---------------------------+                                                                         
| __usb_bus_reprobe_drivers | : check if driver and device match (reprobe is possible)                
+-|-------------------------+                                                                         
  |                                                                                                   
  |--> return if the current driver isn't usb_generic_driver                                          
  |                                                                                                   
  |    +-----------------------+                                                                      
  |--> | usb_driver_applicable | check if driver and device match                                     
  |    +-----------------------+                                                                      
  |                                                                                                   
  |--> return if matched                                                                              
  |                                                                                                   
  |    +----------------+                                                                             
  +--> | device_reprobe | in case probing criteria of the device changes, it might need another driver
       +----------------+                                                                             
```

```
+---------------+                                                           
| ehci_hcd_init | : create /sys/kernel/debug/usb/ehci                       
+-|-------------+                                                           
  |                                                                         
  |--> print "ehci_hcd: USB 2.0 'Enhanced' Host Controller (EHCI) Driver"   
  |                                                                         
  |    +--------------------+                                               
  |--> | debugfs_create_dir | /sys/kernel/debug/usb/ehci                    
  |    +--------------------+                                               
  |    +---------------------------+                                        
  +--> | platform_register_drivers | register nothing bc of disabled configs
       +---------------------------+                                        
```

```
ehci-platform.c
+--------------------+                                                                           
| ehci_platform_init | : determine 'ehci_platform_hc_driver', and register 'ehci_platform_driver'
+-|------------------+                                                                           
  |                                                                                              
  |--> print ehci-platform: EHCI generic platform driver                                         
  |                                                                                              
  |    +------------------+                                                                      
  |--> | ehci_init_driver | determine 'ehci_platform_hc_driver' and overwrite some attributes    
  |    +------------------+                                                                      
  |    +--------------------------+                                                              
  +--> | platform_driver_register | register platform driver 'ehci_platform_driver'              
       +--------------------------+                                                              
```

```
+------------------+                                                                    
| ehci_init_driver | : determine 'ehci_platform_hc_driver' and overwrite some attributes
+-|----------------+                                                                    
  |                                                                                     
  |--> have global 'ehci_platform_hc_driver' point to 'ehci_hc_driver'                  
  |                                                                                     
  +--> apply 'platform_overrides' to overwrite some attributes                          
```

```
storage/usb.c                                                               
+-------------------------+                                                  
| usb_storage_driver_init | : init template and register 'usb_storage_driver'
+-|-----------------------+                                                  
  |                                                                          
  |--> init 'usb_stor_host_template'                                         
  |                                                                          
  +--> register interface driver 'usb_storage_driver'                        
```

```
+-----------------+                                                    
| usb_serial_init | : prepare tty driver, register bus 'usb-serial'    
+-|---------------+                                                    
  |    +------------------+                                            
  |--> | tty_alloc_driver | prepare tty driver                         
  |    +------------------+                                            
  |                                                                    
  |--> assign to 'usb_serial_tty_driver'                               
  |                                                                    
  |    +--------------+                                                
  |--> | bus_register | 'usb_serial_bus_type'                          
  |    +--------------+                                                
  |                                                                    
  |--> further set up 'usb_serial_tty_driver'                          
  |                                                                    
  |    +--------------------+                                          
  |--> | tty_set_operations | 'serial_ops'                             
  |    +--------------------+                                          
  |    +-----------------------------+                                 
  +--> | usb_serial_generic_register | do nothing bc of disabled config
       +-----------------------------+                                 
```

```
serial/pl2303.c                                                                                                    
+------------------------+                                                                                           
| usb_serial_module_init | : register usb intf driver, register arg drivers to bus 'usb-serial'                      
+-|----------------------+                                                                                           
  |    +-----------------------------+                                                                               
  +--> | usb_serial_register_drivers | : register usb intf driver, register arg drivers to bus 'usb-serial'          
       +-|---------------------------+                                                                               
         |                                                                                                           
         |--> alloc usb_driver                                                                                       
         |                                                                                                           
         |--> set up driver and install ops                                                                          
         |                                                                                                           
         |    +--------------+                                                                                       
         |--> | usb_register | register usb interface driver                                                         
         |    +--------------+                                                                                       
         |                                                                                                           
         |--> for each driver in arg 'serial_drivers'                                                                
         |                                                                                                           
         |------> save the newly allocated usb_driver ptr                                                            
         |                                                                                                           
         |        +---------------------+                                                                            
         |------> | usb_serial_register | add driver to 'usb_serial_driver_list', register driver to bus 'usb-serial'
         |        +---------------------+                                                                            
         |                                                                                                           
         |--> set id_table for match                                                                                 
         |                                                                                                           
         |    +---------------+                                                                                      
         +--> | driver_attach | try match device and probe (but I don't see the 'bus' set anywhere)                  
              +---------------+                                                                                      
```

```
+---------------------+                                                                              
| usb_serial_register | : add driver to 'usb_serial_driver_list', register driver to bus 'usb-serial'
+-|-------------------+                                                                              
  |    +----------------------------+                                                                
  |--> | usb_serial_operations_init | reset some fields                                              
  |    +----------------------------+                                                                
  |                                                                                                  
  |--> add driver to 'usb_serial_driver_list'                                                        
  |                                                                                                  
  |    +-------------------------+                                                                   
  +--> | usb_serial_bus_register | set driver bus type to 'usb-serial', register driver              
       +-------------------------+                                                                   
```

```
 +-----------------+                                                                                        
 | gadget_cfs_init | : init gadget_subsys and prepare config_fs for it                                      
 +-|---------------+                                                                                        
   |    +-------------------+                                                                               
   |--> | config_group_init |                                                                               
   |    +-------------------+                                                                               
   |    +-----------------------------+                                                                     
   +--> | configfs_register_subsystem | prepare sb of config_fs, alloc a folder and publish it to user space
        +-----------------------------+                                                                     
```

```
+----------------+
| ast_vhub_probe | : prepare vhub/ports/endpoints, ioremap, register isr, alloc dma buf, init hw
+-|--------------+
  |
  |--> alloc 'vhub' as dev resource
  |
  |--> get 'downstream ports' fom of_node, and it's 5
  |
  |--> alloc that many port struct for vhub
  |
  |--> get 'generic endpoints' fom of_node, and it's 0xf
  |
  |--> alloc that many endpoint struct for vhub
  |
  |    +-----------------------+
  |--> | devm_ioremap_resource | iomap hardware registers
  |    +-----------------------+
  |    +--------+
  |--> | writel | mask and ack all interrupts before installing handlers
  |    +--------+
  |    +------------------+
  |--> | platform_get_irq | get irq number from resource
  |    +------------------+
  |    +------------------+
  |--> | devm_request_irq | register isr
  |    +------------------+ ast_vhub_irq: handle dev irq
  |    +--------------------+
  |--> | dma_alloc_coherent | alloc a big dma buffer for everyone (ep0, port, vhub)
  |    +--------------------+
  |    +-------------------+
  |--> | ast_vhub_init_ep0 | init vhub ep0 (no device addtion involved)
  |    +-------------------+
  |
  |--> for each port
  |
  |        +-------------------+
  |------> | ast_vhub_init_dev | prepare vhub/endpoints/port/gadget, add devices
  |        +-------------------+
  |    +-------------------+
  |--> | ast_vhub_init_hub | init vhub's work and descriptors
  |    +-------------------+
  |    +------------------+
  |--> | ast_vhub_init_hw | init hw
  |    +------------------+
  |
  +--> print "Initialized virtual hub in USB%d mode\n"
```
  
```
+-------------------+
| ast_vhub_init_dev | : prepare vhub/endpoints/port/gadget, add devices
+-|-----------------+
  |
  |--> set up 'vhub dev'
  |
  |    +-------------------+
  |--> | ast_vhub_init_ep0 | (so each device also has an ep0?)
  |    +-------------------+
  |    +---------+
  |--> | kcalloc | alloc that many endpoints beside ep0 (at most 30)
  |    +---------+
  |    +---------+
  |--> | kzalloc | alloc device for port
  |    +---------+
  |    +--------------+
  |--> | dev_set_name |
  |    +--------------+
  |    +------------+
  |--> | device_add |
  |    +------------+
  |
  +--> set up gadget and install 'ast_vhub_udc_ops' (udc for usb device controller)
  |
  |    +--------------------+
  +--> | usb_add_gadget_udc | init gadget, prepare udc, add devices respectively, try binding udc to driver
  |    +--------------------+
  |
  +--> label 'registered' on 'vhub dev'
```

```
+--------------------+
| usb_add_gadget_udc | : init gadget, prepare udc, add devices respectively, try binding udc to driver
+-|------------------+
  |    +----------------------------+
  +--> | usb_add_gadget_udc_release | : init gadget, prepare udc, add devices respectively, try binding udc to driver
       +-|--------------------------+
         |    +-----------------------+
         |--> | usb_initialize_gadget | set name, init work
         |    +-----------------------+
         |    +----------------+
         +--> | usb_add_gadget | prepare udc, add two devices (gadge/udc), try binding udc to driver
              +----------------+
```

```
+----------------+
| usb_add_gadget | : prepare udc, add two devices (gadge/udc), try binding udc to driver
+-|--------------+
  |
  |--> alloc udc
  |
  |--> set up udc
  |
  |    +------------+
  |--> | device_add | add gadget
  |    +------------+
  |
  |--> relate udc and gadget
  |
  |    +---------------+
  |--> | list_add_tail | add udc to 'udc_list'
  |    +---------------+
  |    +------------+
  |--> | device_add | add udc
  |    +------------+
  |    +----------------------+
  |--> | usb_gadget_set_state | set state 'not attached', schedule gadget work
  |    +----------------------+
  |    +------------------------------+
  +--> | check_pending_gadget_drivers | for each driver in list, try binding udc to it
       +------------------------------+
```

```
+------------------------------+
| check_pending_gadget_drivers | ： for each driver in list, try binding udc to it
+-|----------------------------+
  |
  |--> for each driver on 'gadget_driver_pending_list'
  |
  |------> if match
  |
  |            +--------------------+
  |----------> | udc_bind_to_driver | relate udc/driver, set gadget speed and start udc
  |            +--------------------+
  |        +---------------+
  +------> | list_del_init | remove driver from the pending list
           +---------------+
```

```
+--------------------+
| udc_bind_to_driver | : relate udc/driver, set gadget speed, bind composite to gadget, and start udc
+-|------------------+
  |
  |--> save driver info in udc
  |
  |    +--------------------------+
  |--> | usb_gadget_udc_set_speed | call gadget ops to set speed
  |    +--------------------------+
  |
  |--> call driver->bind(), e.g.,
  |    +-------------------------+
  |    | configfs_composite_bind | bind composite to gadget (ready configs and functions)
  |    +-------------------------+
  |    +----------------------+
  |--> | usb_gadget_udc_start | call gadget's ->udc_start()
  |    +----------------------+
  |    +-----------------------------------+
  |--> | usb_gadget_enable_async_callbacks | call gadget's ->udc_async_callbacks() if it exists
  |    +-----------------------------------+
  |    +-------------------------+
  +--> | usb_udc_connect_control | let gadget connect to host if 'udc vbus' exists, otherwise disconnect
       +-------------------------+
```

```
+-------------------------+                                                                                       
| configfs_composite_bind | : bind composite to gadget (ready configs and functions)                              
+-|-----------------------+                                                                                       
  |                                                                                                               
  |--> relate gadget and composite_dev                                                                            
  |                                                                                                               
  |    +-----------------------+                                                                                  
  |--> | composite_dev_prepare | prepare req for composite, assign composite to gadget ep0 as private             
  |    +-----------------------+                                                                                  
  |                                                                                                               
  |--> return error if no config for composite                                                                    
  |                                                                                                               
  |--> check if each config has at least one function                                                             
  |                                                                                                               
  |--> if 'string' list isn't empty                                                                               
  |    |                                                                                                          
  |    |    +---------------------+                                                                               
  |    +--> | usb_gstrings_attach | duplicate strings from gadget_info and attach to composite                    
  |         +---------------------+                                                                               
  |    +---------------+                                                                                          
  |--> | gadget_is_otg | check if it's a on-the-go adapter (ignore the related code)                              
  |    +---------------+                                                                                          
  |                                                                                                               
  |--> for each config                                                                                            
  |    |                                                                                                          
  |    |    +---------------------+                                                                               
  |    |--> | usb_gstrings_attach | duplicate strings from config and attach to composite                         
  |    |    +---------------------+                                                                               
  |    |                                                                                                          
  |    |--> for each func in config                                                                               
  |    |    |                                                                                                     
  |    |    |--> remove func from list                                                                            
  |    |    |                                                                                                     
  |    |    |    +------------------+                                                                             
  |    |    +--> | usb_add_function | add functions to config, get endpoints, alloc req, install ops to, e.g., ecm
  |    |         +------------------+                                                                             
  |    |    +-------------------------+                                                                           
  |    |--> | usb_gadget_check_config | call ->check_config if it exists (not our case)                           
  |    |    +-------------------------+                                                                           
  |    |    +-------------------------+                                                                           
  |    +--> | usb_ep_autoconfig_reset | reset endpoints and gadget                                                
  |         +-------------------------+                                                                           
  |                                                                                                               
  +--> if 'use_os_string' is set                                                                                  
       |                                                                                                          
       |    +-------------------------------+                                                                     
       +--> | composite_os_desc_req_prepare | prepare req (and its buffer), relate it and composite_dev           
            +-------------------------------+                                                                     
```

```
+-------------------------------+                                                            
| composite_os_desc_req_prepare | : prepare req (and its buffer), relate it and composite_dev
+-|-----------------------------+                                                            
  |    +----------------------+                                                              
  |--> | usb_ep_alloc_request | alloc request                                                
  |    +----------------------+                                                              
  |                                                                                          
  |--> alloc buffer for req                                                                  
  |                                                                                          
  +--> relate composite_dev and req                                                          
```

```
+------------------+                                                                               
| usb_add_function | : add functions to config, get endpoints, alloc req, install ops to, e.g., ecm
+-|----------------+                                                                               
  |                                                                                                
  |--> relate function and config                                                                  
  |                                                                                                
  |--> if ->bind exists                                                                            
  |                                                                                                
  +------> call ->bind, e.g.,                                                                      
           +----------+                                                                            
           | ecm_bind | add functions to config, get endpoints, alloc req, install ops to ecm      
           +----------+                                                                            
```

```
+----------+                                                                           
| ecm_bind | : add functions to config, get endpoints, alloc req, install ops to ecm   
+-|--------+                                                                           
  |                                                                                    
  |--> get opts from 'function'                                                        
  |                                                                                    
  |--> if opts isn't bound yet                                                         
  |                                                                                    
  |        +-------------------+                                                       
  |------> | gether_set_gadget | relate net_dev and gadget                             
  |        +-------------------+                                                       
  |        +------------------------+                                                  
  |------> | gether_register_netdev | set mac to net_dev and register it, clear carrier
  |        +------------------------+                                                  
  |    +---------------------+                                                         
  |--> | usb_gstrings_attach | attach strings from ecm to composite                    
  |    +---------------------+                                                         
  |    +------------------+                                                            
  |--> | usb_interface_id | config->interface[id] = function, id++                     
  |    +------------------+                                                            
  |    +------------------+                                                            
  |--> | usb_interface_id | one for 'in', and one for 'out'?                           
  |    +------------------+                                                            
  |    +-------------------+                                                           
  |--> | usb_ep_autoconfig | get an available ep and set up (for 'in')                 
  |    +-------------------+                                                           
  |    +-------------------+                                                           
  |--> | usb_ep_autoconfig | for 'out'                                                 
  |    +-------------------+                                                           
  |    +-------------------+                                                           
  |--> | usb_ep_autoconfig | for 'notify'                                              
  |    +-------------------+                                                           
  |    +----------------------+                                                        
  +--> | usb_ep_alloc_request | alloc request for ecm                                  
  |    +----------------------+                                                        
  |                                                                                    
  |--> set up high-speed and super-speed from full-speed setting                       
  |                                                                                    
  |    +------------------------+                                                      
  |--> | usb_assign_descriptors | set up 'function' speed descriptors: ssp, ss, hs, fs 
  |    +------------------------+                                                      
  |                                                                                    
  +--> install ops to ecm port                                                         
       +----------+                                                                    
       | ecm_open | notify endpoint of the 'open'                                      
       +-----------+                                                                   
       | ecm_close | notify endpoint of the 'close'                                    
       +-----------+                                                                   
```

```
+----------+                                                     
| ecm_open | : notify endpoint of the 'open'                     
+-|--------+                                                     
  |                                                              
  |--> is_open = true                                            
  |                                                              
  |--> state = CONNECT                                           
  |                                                              
  |    +---------------+                                         
  +--> | ecm_do_notify | set up req, add to ep_queue for handling
       +---------------+                                         
```

```
+---------------+                                               
| ecm_do_notify | : set up req, add to ep_queue for handling    
+-|-------------+                                               
  |                                                             
  |--> switch state                                             
  |                                                             
  |--> case connect                                             
  |                                                             
  |------> set up event                                         
  |                                                             
  |------> state = 'speed'                                      
  |                                                             
  |--> case speed                                               
  |                                                             
  |------> set up event and data                                
  |                                                             
  |------> state = 'none'                                       
  |                                                             
  |    +--------------+                                         
  +--> | usb_ep_queue | set up req, add to ep_queue for handling
       +--------------+                                         
```

```
+--------------+                                                      
| usb_ep_queue | : set up req, add to ep_queue for handling           
+-|------------+                                                      
  |                                                                   
  +--> call ->queue, e.g.,                                            
       +--------------------+                                         
       | ast_vhub_epn_queue | set up req, add to ep_queue for handling
       +--------------------+                                         
```

```
+--------------------+                                                                                       
| ast_vhub_epn_queue | : set up req, add to ep_queue for handling                                            
+-|------------------+                                                                                       
  |                                                                                                          
  |--> set up req                                                                                            
  |                                                                                                          
  |--> append req to ep_queue                                                                                
  |                                                                                                          
  |--> if the queue was empty                                                                                
  |                                                                                                          
  |------> if it's desc_mode                                                                                 
  |                                                                                                          
  |            +------------------------+                                                                    
  +----------> | ast_vhub_epn_kick_desc | given req, set up descripters, kick hardware to process            
  |            +------------------------+                                                                    
  |                                                                                                          
  |------> else                                                                                              
  |                                                                                                          
  |            +-------------------+                                                                         
  +----------> | ast_vhub_epn_kick | prepare buf, save addr to hw reg, label 'active' on req, and kick and hw
               +-------------------+                                                                         
```

```
+-------------------+
| ast_vhub_init_hub | : init vhub's work and descriptors
+-|-----------------+
  |    +-----------+
  |--> | INIT_WORK | ast_vhub_wake_work: resume each port, wake up host
  |    +-----------+
  |    +--------------------+
  +--> | ast_vhub_init_desc | init descriptors of vhub
       +--------------------+
```

```
+--------------------+
| ast_vhub_wake_work | : resume each port, wake up host
+-|-- ---------------+
  |
  |--> for each port in vhub
  |
  |------> ensure the port is in 'suspend' status
  |
  |        +---------------------+
  |------> | ast_vhub_dev_resume | call vhub driver->resume()
  |        +---------------------+
  |    +---------------------------+
  +--> | ast_vhub_send_host_wakeup | write hw reg to wake up host
       +---------------------------+
```
  
```
+--------------------+
| ast_vhub_init_desc | : init descriptors of vhub
+-|------------------+
  |
  |--> init vhub dev descriptor
  |
  |--> init vhub config descriptor
  |
  |--> init vhub hub descriptor
  |
  +--> init vhub string descriptor
```

```
+------------------+
| ast_vhub_init_hw | : init hw
+-|----------------+
  |
  |--> enable phy
  |
  |--> set descriptor ring size
  |
  |--> reset all devices
  |
  |--> disable and clean up ack/nack interrupts
  |
  |--> have default setting for ep0, enable hw hub for ep1
  |
  |--> configure ep0 dma buffer
  |
  |--> clear address
  |
  |--> pull up hub (the effect of insertion?)
  |
  +--> enable some interrupts
```

```
+-------------+                                                               
| ecmmod_init | : set up function driver by args, register the function driver
+-|-----------+                                                               
  |                                                                           
  |--> prepare function driver (ecm_alloc_inst, ecm_alloc)                    
  |                                                                           
  |        +----------------+                                                 
  |        | ecm_alloc_inst | alloc opts, prepare netdev                      
  |        +----------------+                                                 
  |        +-----------+                                                      
  |        | ecm_alloc | alloc ecm, set up port and install ops               
  |        +-----------+                                                      
  |    +-----------------------+                                              
  +--> | usb_function_register |                                              
       +-----------------------+                                              
```

```
+----------------+                                                                                       
| ecm_alloc_inst | : alloc opts, prepare netdev                                                          
+---|------------+                                                                                       
    |                                                                                                    
    |--> alloc opts                                                                                      
    |                                                                                                    
    |--> install free func = ecm_free_inst                                                               
    |                                                                                                    
    |    +----------------------+                                                                        
    |--> | gether_setup_default | alloc netdev, set name, generate random mac for dev & host, install ops
    |    +----------------------+                                                                        
    |    +-----------------------------+                                                                 
    +--> | config_group_init_type_name | set item = ""?                                                  
         +-----------------------------+                                                                 
```

```
+----------------------+                                                                                      
| gether_setup_default | : alloc netdev, set name, generate random mac for dev & host, install ops            
+-|--------------------+                                                                                      
  |    +---------------------------+                                                                          
  +--> | gether_setup_name_default | : alloc netdev, set name, generate random mac for dev & host, install ops
       +-|-------------------------+                                                                          
         |    +----------------+                                                                              
         |--> | alloc_etherdev | alloc net_dev                                                                
         |    +----------------+                                                                              
         |                                                                                                    
         |--> label mac type as 'random'                                                                      
         |                                                                                                    
         |    +---------------------+                                                                         
         |--> | skb_queue_head_init | init skb list                                                           
         |    +---------------------+                                                                         
         |                                                                                                    
         |--> decide net_dev name                                                                             
         |                                                                                                    
         |    +-----------------+                                                                             
         |--> | eth_random_addr | generate random mac for dev                                                 
         |    +-----------------+                                                                             
         |                                                                                                    
         |--> print "using random %s ethernet address\n", "self"                                              
         |                                                                                                    
         |    +-----------------+                                                                             
         |--> | eth_random_addr | generate random mac for host                                                
         |    +-----------------+                                                                             
         |                                                                                                    
         |--> print "using random %s ethernet address\n", "host"                                              
         |                                                                                                    
         |--> install 'eth_netdev_ops' for net_dev                                                            
         |                                                                                                    
         |--> install 'ops' for eth tool                                                                      
         |                                                                                                    
         |--> set dev type = 'gadget'                                                                         
         |                                                                                                    
         +--> set max/min mtu                                                                                 
```

```
+-----------+                                            
| ecm_alloc | : alloc ecm, set up port and install ops   
+-|---------+                                            
  |                                                      
  |--> alloc ecm                                         
  |                                                      
  |--> get opts from arg fi                              
  |                                                      
  |    +--------------------------+                      
  +--> | gether_get_host_addr_cdc | get mac in cdc format
  |    +--------------------------+                      
  |                                                      
  +--> set up port and install ops                       
```

```
function/f_mass_storage.c                                                                   
+----------------------+                                                                     
| mass_storagemod_init | : set up function driver by args, register the function driver      
+-|--------------------+                                                                     
  |                                                                                          
  |--> prepare function driver (fsg_alloc_inst, fsg_alloc)                                   
  |        +----------------+                                                                
  |        | fsg_alloc_inst | alloc opts, set up 'func_inst' and 'common', return 'func_inst'
  |        +----------------+                                                                
  |        +-----------+                                                                     
  |        | fsg_alloc | alloc fsg, set up 'function' and return it                          
  |        +-----------+                                                                     
  |    +-----------------------+                                                             
  +--> | usb_function_register | register function driver to 'func_list'                     
       +-----------------------+                                                             
```

```
+----------------+                                                                             
| fsg_alloc_inst | : alloc opts, set up 'func_inst' and 'common', return 'func_inst'           
+-|--------------+                                                                             
  |                                                                                            
  |--> alloc opts                                                                              
  |                                                                                            
  |--> install free func 'fsg_free_inst'                                                       
  |                                                                                            
  |    +------------------+                                                                    
  |--> | fsg_common_setup | ensure 'common' exists and init it                                 
  |    +------------------+                                                                    
  |    +----------------------------+                                                          
  |--> | fsg_common_set_num_buffers | alloc buf heads and buffer to replace the one of 'common'
  |    +----------------------------+                                                          
  |                                                                                            
  |--> print ", version: " FSG_DRIVER_VERSION "\n"                                             
  |                                                                                            
  |    +-----------------------+                                                               
  |--> | fsg_common_create_lun | prepare lun, save in 'common'                                 
  |    +-----------------------+                                                               
  |    +-----------------------------+                                                         
  |--> | config_group_init_type_name | ???                                                     
  |    +-----------------------------+                                                         
  |    +-----------------------------+                                                         
  |--> | config_group_init_type_name | ???                                                     
  |    +-----------------------------+                                                         
  |    +----------------------------+                                                          
  +--> | configfs_add_default_group | ???                                                      
       +----------------------------+                                                          
```

```
+----------------------------+                                                            
| fsg_common_set_num_buffers | : alloc buf heads and buffer to replace the one of 'common'
+-|--------------------------+                                                            
  |                                                                                       
  |--> alloc buf_head * n                                                                 
  |                                                                                       
  |--> alloc buffer for each buf_head                                                     
  |                                                                                       
  |    +--------------------------+                                                       
  |--> | _fsg_common_free_buffers | free buf heads and their buffer                       
  |    +--------------------------+                                                       
  |                                                                                       
  +--> install the newly generated buf heads to 'common'                                  
```

```
+-----------------------+                                                             
| fsg_common_create_lun | : prepare lun, save in 'common'                             
+-|---------------------+                                                             
  |                                                                                   
  |--> alloc and setup lun                                                            
  |                                                                                   
  |--> if common->sysfs has no value                                                  
  |                                                                                   
  |------> set lun name                                                               
  |                                                                                   
  |--> else                                                                           
  |                                                                                   
  |------> set lun name                                                               
  |                                                                                   
  |        +-----------------+                                                        
  |------> | device_register | register lun device                                    
  |        +-----------------+                                                        
  |                                                                                   
  |--> save lun in 'common'                                                           
  |                                                                                   
  |--> if cfg has name                                                                
  |                                                                                   
  |        +--------------+                                                           
  |------> | fsg_lun_open | set up arg curlun by file attributes and block/sector info
  |        +--------------+                                                           
  |                                                                                   
  +--> prepare lun file info and print it                                             
```

```
+--------------+                                                             
| fsg_lun_open | : set up arg curlun by file attributes and block/sector info
+-|------------+                                                             
  |    +-----------+                                                         
  |--> | filp_open |                                                         
  |    +-----------+                                                         
  |                                                                          
  +--> set up arg curlun by file attributes and block/sector info            
```

```
function/f_hid.c                                                             
+-------------+                                                               
| hidmod_init | : set up function driver by args, register the function driver
+-|-----------+                                                               
  |                                                                           
  |--> prepare function driver based on                                       
  |        +-----------------+                                                
  |        | hidg_alloc_inst | prepare opts                                   
  |        +-----------------+                                                
  |        +------------+                                                     
  |        | hidg_alloc | alloc hidg, set up it based on opts, install ops    
  |        +------------+                                                     
  |    +-----------------------+                                              
  +--> | usb_function_register | register function driver to 'func_list'      
       +-----------------------+                                              
```

```
+-----------------+                                                                 
| hidg_alloc_inst | : prepare opts                                                  
+-|---------------+                                                                 
  |                                                                                 
  |--> alloc opts                                                                   
  |                                                                                 
  |--> set free_func = hidg_free_inst                                               
  |                                                                                 
  |--> if hidg_ida isn't set yet                                                    
  |                                                                                 
  |        +------------+                                                           
  |------> | ghid_setup | create class 'hidg' and request a range of dev_t          
  |        +------------+                                                           
  |    +----------------+                                                           
  |--> | hidg_get_minor | get a minor from requested dev_t range, and assign to opts
  |    +----------------+                                                           
  |    +-----------------------------+                                              
  +--> | config_group_init_type_name | ???                                          
       +-----------------------------+                                              
```

```
+------------+                                                   
| ghid_setup | : create class 'hidg' and request a range of dev_t
+-|----------+                                                   
  |    +--------------+                                          
  |--> | class_create | 'hidg'                                   
  |    +--------------+                                          
  |    +---------------------+                                   
  |--> | alloc_chrdev_region | request dev_t for 'hidg'          
  |    +---------------------+                                   
  |                                                              
  +--> save requested dev_t somewhere                            
```

```
+------------+                                                   
| hidg_alloc | : alloc hidg, set up it based on opts, install ops
+--|---------+                                                   
   |                                                             
   |--> alloc hidg                                               
   |                                                             
   |--> get of opts that contains arg func_inst                  
   |                                                             
   |--> set up hidg based on opts                                
   |                                                             
   +--> install operations to hidg                               
```

```
hid/hid-core.c                                             
+----------+                                                 
| hid_init | : register bus and create /sys/kernel/debug/hid/
+-|--------+                                                 
  |    +--------------+                                      
  |--> | bus_register | 'hid_bus_type'                       
  |    +--------------+                                      
  |    +-------------+                                       
  |--> | hidraw_init | do nothing bc of disabled config      
  |    +-------------+                                       
  |    +----------------+                                    
  +--> | hid_debug_init | create /sys/kernel/debug/hid/      
       +----------------+                                                            
```

```
hid/hid-generic.c                                                                         
+------------------+                                                                       
| hid_generic_init | : register hid driver 'hid_generic', trigger the match between dev/drv
+-|----------------+                                                                       
  |    +---------------------+                                                             
  +--> | hid_register_driver |                                                             
       +-|-------------------+                                                             
         |    +-----------------------+                                                    
         +--> | __hid_register_driver |                                                    
              +-|---------------------+                                                    
                |                                                                          
                |--> set bus = hid_bus_type                                                
                |                                                                          
                |    +-----------------+                                                   
                +--> | driver_register |                                                   
                     +-|---------------+                                                   
                       |                                                                   
                       |--> for each driver on bus                                         
                       |                                                                   
                       |        +------------------------+                                 
                       +------> | __hid_bus_driver_added |                                 
                                +-|----------------------+                                 
                                  |                                                        
                                  |--> for each device on bus                              
                                  |                                                        
                                  |        +---------------------------+                   
                                  +------> | __hid_bus_reprobe_drivers | try to match      
                                           +---------------------------+                   
```

```
usbhid/hid-core.c                                                      
+----------+                                                            
| hid_init | : init quirks, register 'hid_driver'                       
+-|--------+                                                            
  |    +-----------------+                                              
  |--> | hid_quirks_init | for each quirk: ensure it's in 'dquirks_list'
  |    +-----------------+                                              
  |    +--------------+                                                 
  |--> | usb_register | register usb interface driver                   
  |    +--------------+                                                 
  |                                                                     
  +--> print "usbhid: USB HID core driver"                              
```

```
+-----------------+                                                                              
| hid_quirks_init | : for each quirk: ensure it's in 'dquirks_list'                              
+-|---------------+                                                                              
  |                                                                                              
  |--> for each quirk param                                                                      
  |                                                                                              
  |------> fetch (vendor, product, quirks) from param                                            
  |                                                                                              
  |        +-------------------+                                                                 
  +------> | hid_modify_dquirk | prepare quirk_new based on arg id, ensure it's in 'dquirks_list'
           +-------------------+                                                                 
```

```
+-------------------+                                                                   
| hid_modify_dquirk | : prepare quirk_new based on arg id, ensure it's in 'dquirks_list'
+-|-----------------+                                                                   
  |                                                                                     
  |--> alloc hid_dev                                                                    
  |                                                                                     
  |--> alloc quirk_new                                                                  
  |                                                                                     
  |--> set up hid_dev and quirk_new based on arg id                                     
  |                                                                                     
  |--> for each node on dquirks_list                                                    
  |                                                                                     
  |        +------------------+                                                         
  |------> | hid_match_one_id |                                                         
  |        +------------------+                                                         
  |                                                                                     
  |------> if match                                                                     
  |                                                                                     
  |----------> replace node with quirk_new for a better match                           
  |                                                                                     
  |--> if not match (new node)                                                          
  |                                                                                     
  +------> append quirk_new to the end of 'dquirks_list'                                
```
  
## <a name="reference"></a> Reference

- [T. Petazzoni, Using USB gadget drivers](https://bootlin.com/doc/legacy/usb-gadget/usb_gadget_drivers.pdf)
- [M. Porter, Kernel USB Gadget Configfs Interface](https://elinux.org/images/e/ef/USB_Gadget_Configfs_API_0.pdf)
