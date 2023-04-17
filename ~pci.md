```
pci_realloc_setup_params:           duplicate some variables from init section
pcibus_class_init:                  register pci bus class
pci_driver_init:                    register both 'pci' and 'pcie_port' buses
pci_slot_init:                      create kset 'slots' for pci bus
pci_apply_final_quirks:             apply patch?
pci_proc_init:                      create /proc/bus/pci and attach device info
pcie_portdrv_init:                  register port services and port driver
pci_stub_init:                      (probably do nothing)
aspeed_pcie_driver_init:            register aspeed_pcie_driver
  aspeed_pcie_probe:                prepare host bridge/bus, register them, recursively scan buses, and then add devices
pci_resource_alignment_sysfs_init:  create sys files 
pci_sysfs_init:                     for each pci_dev: create sys files    
```

```
drivers/pci/proc.c                                                                        
+---------------+                                                                          
| pci_proc_init | : create /proc/bus/pci and attach device info                            
+-|-------------+                                                                          
  |                                                                                        
  |--> create /proc/bus/pci                                                                
  |                                                                                        
  +--> for each pci_dev on bus                                                             
       |                                                                                   
       |    +------------------------+                                                     
       +--> | pci_proc_attach_device | determine file name (slot.func) and create proc file
            +------------------------+                                                     
```

```
drivers/pci/proc.c                                                              
+------------------------+                                                       
| pci_proc_attach_device | : determine file name (slot.func) and create proc file
+-|----------------------+                                                       
  |                                                                              
  |--> if bus doesn't have dir yet                                               
  |    -                                                                         
  |    +--> determine name and create proc dir                                   
  |                                                                              
  +--> determine file name (slot.func) and create proc file                      
```

```
drivers/pci/pcie/portdrv_pci.c                                           
+-------------------+                                                     
| pcie_portdrv_init | : register port services and port driver            
+-|-----------------+                                                     
  |    +--------------------+                                             
  |--> | pcie_init_services | register port services that can probe device
  |    +--------------------+                                             
  |    +---------------------+                                            
  +--> | pci_register_driver | register pcie port driver                  
       +---------------------+                                            
```

```
drivers/pci/controller/pcie-aspeed.c                                                                            
+-------------------+                                        
| aspeed_pcie_probe | : prepare host bridge/bus, register them, recursively scan buses, and then add devices
+-|-----------------+                                        
  |    +----------------------------+                                                                            
  |--> | devm_pci_alloc_host_bridge | : set up bridge (install irq ops, parse and add pci ranges as dev resource)
  |    +----------------------------+                                                                            
  |    +-------------------+                                                                                     
  |--> | aspeed_pcie_setup | io-remap, parse dt properties, init hw regs, register isr                           
  |    +-------------------+                                                                                     
  |                                                                                                              
  |--> install host ops (aspeed_pcie_ops)                                                                        
  |                                                                                                              
  |--> try to get gpio_desc "perst-ep-in"                                                                        
  |                                                                                                              
  |--> if got                                                                                                    
  |    |                                                                                                         
  |    |--> install isr for perst-ep-in                                                                          
  |    |    +----------------------+                                                                             
  |    |    | pcie_rst_irq_handler | remove all pci_dev on list, toggle perst_rc_out, rescan bus                 
  |    |    +----------------------+                                                                             
  |    |                                                                                                         
  |    +--> init work                                                                                            
  |         +------------------------+                                                                           
  |         | aspeed_pcie_reset_work | remove all pci_dev on list, toggle perst_rc_out, rescan bus               
  |         +------------------------+                                                                           
  |    +----------------+                                                                                        
  +--> | pci_host_probe | prepare bus, register bridge/bus, recursively scan buses, and then add devices
       +----------------+                                                                                        
```

```
drivers/pci/probe.c                                                                                   
+----------------+                                                                                     
| pci_host_probe | : prepare bus, register bridge/bus, recursively scan buses, and then add devices    
+-|--------------+                                                                                     
  |    +--------------------------+                                                                    
  |--> | pci_scan_root_bus_bridge | prepare bus, register bridge/bus, recursively scan child bus       
  |    +--------------------------+                                                                    
  |                                                                                                    
  |--> request iomem or ioport resource                                                                
  |                                                                                                    
  |    +---------------------+                                                                         
  +--> | pci_bus_add_devices | recursively for each pci_dev: create sysfs/proc files & attach to driver
       +---------------------+                                                                         
```

```
drivers/pci/probe.c                                                                                             
+--------------------------+                                                                                     
| pci_scan_root_bus_bridge | : prepare bus, register bridge/bus, recursively scan child bus                      
+-|------------------------+                                                                                     
  |                                                                                                              
  |--> for each window in bridge, try to find one with 'bus' type resource                                       
  |                                                                                                              
  |    +--------------------------+                                                                              
  |--> | pci_register_host_bridge | prepare bus for bridge, register both bridge/bus, add bus to 'pci_root_buses'
  |    +--------------------------+                                                                              
  |                                                                                                              
  |--> if not found (not 'bus' type resource)                                                                    
  |    |                                                                                                         
  |    |--> print "No busn resource found for root bus, will use [bus %02x-ff]\n"                                
  |    |                                                                                                         
  |    |    +-------------------------+                                                                          
  |    +--> | pci_bus_insert_busn_res | request and reserve resource                                             
  |         +-------------------------+                                                                          
  |    +--------------------+                                                                                    
  +--> | pci_scan_child_bus | scan pci devices on the bus, recursively scan bridges                              
       +--------------------+                                                                                    
```

```
drivers/pci/probe.c                                                                                        
+--------------------------+                                                                                
| pci_register_host_bridge | : prepare bus for bridge, register both bridge/bus, add bus to 'pci_root_buses'
+-|------------------------+                                                                                
  |    +---------------+                                                                                    
  |--> | pci_alloc_bus | alloc pci_bus for bridge                                                           
  |    +---------------+                                                                                    
  |                                                                                                         
  |--> determine bridge name "pci%04x:%02x"                                                                 
  |                                                                                                         
  |    +------------+                                                                                       
  |--> | device_add | add to bus, send 'add dev' to notifier, probe device                                  
  |    +------------+                                                                                       
  |                                                                                                         
  |--> determine bus name "pci%04x:%02x"                                                                    
  |                                                                                                         
  |    +-----------------+                                                                                  
  |--> | device_register | add to bus, send 'add dev' to notifier, probe device                             
  |    +-----------------+                                                                                  
  |                                                                                                         
  |--> print "PCI host bridge to bus %s\n"                                                                  
  |                                                                                                         
  |--> for each initial resource                                                                            
  |    |                                                                                                    
  |    |--> add to bus                                                                                      
  |    |                                                                                                    
  |    +--> print "root bus resource %pR%s\n"                                                               
  |                                                                                                         
  +--> add bus to 'pci_root_buses'                                                                          
```

```
drivers/pci/probe.c                                                                                      
+----------------------------+                                                                            
| devm_pci_alloc_host_bridge | : set up bridge (install irq ops, parse and add pci ranges as dev resource)
+-|--------------------------+                                                                            
  |    +-----------------------+                                                                          
  |--> | pci_alloc_host_bridge | prepare 'bridge'                                                         
  |    +-----------------------+                                                                          
  |    +--------------------------+                                                                       
  |--> | devm_add_action_or_reset | add action 'release' to device as resource                            
  |    +--------------------------+                                                                       
  |    +-------------------------+                                                                        
  +--> | devm_of_pci_bridge_init | install irq ops to bridge, parse pci ranges as dev resource            
       +-------------------------+                                                                        
```

```
drivers/pci/of.c                                                                                                    
+-------------------------+                                                                                          
| devm_of_pci_bridge_init | : install irq ops to bridge, parse pci ranges as dev resource                            
+-|-----------------------+                                                                                          
  |                                                                                                                  
  |--> install irq ops to bridge                                                                                     
  |                                                                                                                  
  |    +---------------------------------+                                                                           
  +--> | pci_parse_request_of_pci_ranges | for each range in dts, add to dev as resource and request region in io_mem
       +---------------------------------+                                                                           
```

```
drivers/pci/of.c                                                                                               
+---------------------------------+                                                                             
| pci_parse_request_of_pci_ranges | : for each range in dts, add to dev as resource and request region in io_mem
+-|-------------------------------+                                                                             
  |    +---------------------------------------+                                                                
  |--> | devm_of_pci_get_host_bridge_resources | parse pci ranges and add to dev as resource                    
  |    +---------------------------------------+                                                                
  |    +--------------------------------+                                                                       
  |--> | devm_request_pci_bus_resources | for each resource list, request and region in io_mem (or io_port)     
  |    +--------------------------------+                                                                       
  |                                                                                                             
  +--> for each resource if list                                                                                
       -                                                                                                        
       +--> if it's io type (not mem type), perform remap                                                       
```

```
drivers/pci/of.c                                                                                            
+---------------------------------------+                                                                    
| devm_of_pci_get_host_bridge_resources | : parse pci ranges and add to dev as resource                      
+-|-------------------------------------+                                                                    
  |                                                                                                          
  |--> alloc 'bus_range' as dev resource                                                                     
  |                                                                                                          
  |--> print "host bridge %pOF ranges:\n"                                                                    
  |                                                                                                          
  |    +------------------------+                                                                            
  |--> | of_pci_parse_bus_range | parse bus range in dts and save in 'bus_range'                             
  |    +------------------------+                                                                            
  |    +------------------+                                                                                  
  |--> | pci_add_resource | add to resource list                                                             
  |    +------------------+                                                                                  
  |    +--------------------------+                                                                          
  |--> | of_pci_range_parser_init | set up parser and get property 'ranges' from dts                         
  |    +--------------------------+                                                                          
  |                                                                                                          
  |--> for each discontinuous range                                                                          
  |    |                                                                                                     
  |    |    +-------------------------+                                                                      
  |    |--> | of_pci_range_parser_one | parse range, translate from bus addr to cpu addr, save in arg 'range'
  |    |    +-------------------------+                                                                      
  |    |                                                                                                     
  |    |--> print range info "  %6s %#012llx..%#012llx -> %#012llx\n"                                        
  |    |                                                                                                     
  |    |    +--------------------------+                                                                     
  |    |--> | of_pci_range_to_resource | wrap range as resource                                              
  |    |    +--------------------------+                                                                     
  |    |    +--------------+                                                                                 
  |    |--> | devm_kmemdup | add resource to dev                                                             
  |    |    +--------------+                                                                                 
  |    |    +-------------------------+                                                                      
  |    +--> | pci_add_resource_offset | add the above resource to arg 'resource' list as well                
  |         +-------------------------+                                                                      
  |                                                                                                          
  +--> similarly handle dma ranges (if there's any)                                                          
```

```
drivers/pci/controller/pcie-aspeed.c                                                                             
+-------------------+                                                                                             
| aspeed_pcie_setup | : io-remap, parse dt properties, init hw regs, register isr                                 
+-|-----------------+                                                                                             
  |    +--------------------------------+                                                                         
  |--> | devm_platform_ioremap_resource | io-remap                                                                
  |    +--------------------------------+                                                                         
  |                                                                                                               
  |--> parse dt properties and save in private                                                                    
  |                                                                                                               
  |    +-----------------------+                                                                                  
  |--> | aspeed_pcie_port_init | init hw registers                                                                
  |    +-----------------------+                                                                                  
  |    +-----------------------------+                                                                            
  |--> | aspeed_pcie_init_irq_domain | (skip)                                                                     
  |    +-----------------------------+                                                                            
  |    +----------------------------------+                                                                       
  +--> | irq_set_chained_handler_and_data | install isr for pcie irq                                              
       +----------------------------------+ +--------------------------+                                          
                                            | aspeed_pcie_intr_handler | read hw reg and handle irq(s) accordingly
                                            +--------------------------+                                          
```

```
drivers/pci/controller/pcie-aspeed.c                                                      
+--------------------------+                                                               
| aspeed_pcie_intr_handler | : read hw reg and handle irq(s) accordingly                   
+-|------------------------+                                                               
  |                                                                                        
  |--> read interrupt status from hw reg                                                   
  |                                                                                        
  +--> for each set bit (at most 4)                                                        
       |                                                                                   
       |    +---------------------------+                                                  
       +--> | generic_handle_domain_irq | given hwirq, get irq desc and call ->handle_irq()
            +---------------------------+                                                  
```

```
drivers/pci/controller/pcie-aspeed.c                                                        
+----------------------+                                                                     
| pcie_rst_irq_handler | : remove all pci_dev on list, toggle perst_rc_out, rescan bus       
+-|--------------------+                                                                     
  |    +-----------------------+                                                             
  +--> | schedule_delayed_work |                                                             
       +------------------------+                                                            
       | aspeed_pcie_reset_work | remove all pci_dev on list, toggle perst_rc_out, rescan bus
       +------------------------+                                                            
```

```
drivers/pci/controller/pcie-aspeed.c                                                                    
+------------------------+                                                                               
| aspeed_pcie_reset_work | : remove all pci_dev on list, toggle perst_rc_out, rescan bus                 
+-|----------------------+                                                                               
  |                                                                                                      
  |--> for each dev on bus                                                                               
  |    |                                                                                                 
  |    |    +--------------------------------+                                                           
  |    |--> | pci_stop_and_remove_bus_device | remove pci dev (and its children if there's any) from list
  |    |    +--------------------------------+                                                           
  |    |                                                                                                 
  |    +--> send cmd to ensure it won't generate new request                                             
  |                                                                                                      
  |--> if perst_rc_out exists, toggle it (-> 0 -> 1)                                                     
  |                                                                                                      
  |--> read hw reg and print link status                                                                 
  |                                                                                                      
  |    +----------------+                                                                                
  +--> | pci_rescan_bus | scan all pci_dev, assign resources (io/mmio), attach to drivers                
       +----------------+                                                                                
```

```
drivers/pci/controller/pcie-aspeed.c                                                                  
+----------------+                                                                                     
| pci_rescan_bus | : scan all pci_dev, assign resources (io/mmio), attach to drivers                   
+-|--------------+                                                                                     
  |    +--------------------+                                                                          
  |--> | pci_scan_child_bus | scan pci devices on the bus, recursively scan bridges                    
  |    +--------------------+                                                                          
  |    +-------------------------------------+                                                         
  |--> | pci_assign_unassigned_bus_resources | set up io/mmio for bridges                              
  |    +-------------------------------------+                                                         
  |    +---------------------+                                                                         
  +--> | pci_bus_add_devices | recursively for each pci_dev: create sysfs/proc files & attach to driver
       +---------------------+                                                                         
```

```
drivers/pci/probe.c                                                                                    
+--------------------+                                                                                  
| pci_scan_child_bus | : scan pci devices on the bus, recursively scan bridges                          
+-|------------------+                                                                                  
  |    +---------------------------+                                                                    
  +--> | pci_scan_child_bus_extend | : scan pci devices on the bus, recursively scan bridges            
       +-|-------------------------+                                                                    
         |                                                                                              
         |--> for each dev (0 ~ 255)                                                                    
         |    |                                                                                         
         |    |    +---------------+                                                                    
         |    +--> | pci_scan_slot | ensure there's one pci_dev for each function on the slot           
         |         +---------------+                                                                    
         |                                                                                              
         |--> for each pci bridge that is configured                                                    
         |    |                                                                                         
         |    |    +------------------------+                                                           
         |    +--> | pci_scan_bridge_extend | prepare child pci_bus, add to list of parent, scan the bus
         |         +------------------------+                                                           
         |                                                                                              
         +--> for each pci bridge that needs to be reconfigured (2nd pass)                              
              -                                                                                         
              +--> (skip)                                                                               
```

```
drivers/pci/probe.c                                                                                
+---------------+                                                                                   
| pci_scan_slot | : ensure there's one pci_dev for each function on the slot                        
+-|-------------+                                                                                   
  |    +------------------------+                                                                   
  |--> | pci_scan_single_device | ensure pci_dev is there (if not, prepare one and register it)     
  |    +------------------------+                                                                   
  |                                                                                                 
  +--> for other function(s) of the slot                                                            
       |                                                                                            
       |    +------------------------+                                                              
       +--> | pci_scan_single_device | ensure pci_dev is there (if not, prepare one and register it)
            +------------------------+                                                              
```

```
drivers/pci/probe.c                                                                      
+------------------------+                                                                
| pci_scan_single_device | : ensure pci_dev is there (if not, prepare one and register it)
+-|----------------------+                                                                
  |    +--------------+                                                                   
  |--> | pci_get_slot | given dev_fn, find matched dev on target bus                      
  |    +--------------+                                                                   
  |                                                                                       
  |--> if found, return                                                                   
  |                                                                                       
  |    +-----------------+                                                                
  |--> | pci_scan_device | prepare pci_dev (bdf, port type, slot, ...)                    
  |    +-----------------+                                                                
  |    +----------------+                                                                 
  +--> | pci_device_add | add pci_dev to list in pci_bus, register dev                    
       +----------------+                                                                 
```

```
drivers/pci/probe.c                                                                    
+-----------------+                                                                     
| pci_scan_device | : prepare pci_dev (bdf, port type, slot, ...)                       
+-|---------------+                                                                     
  |    +----------------------------+                                                   
  |--> | pci_bus_read_dev_vendor_id |                                                   
  |    +----------------------------+                                                   
  |    +---------------+                                                                
  |--> | pci_alloc_dev | set up 'pci_dev' struct                                        
  |    +---------------+                                                                
  |    +------------------+                                                             
  +--> | pci_setup_device | set port type, assign slot, read more info based on hdr type
       +------------------+                                                             
```

```
drivers/pci/probe.c                                                               
+------------------+                                                               
| pci_setup_device | : set port type, assign slot, read more info based on hdr type
+-|----------------+                                                               
  |                                                                                
  |--> set up pci_dev                                                              
  |                                                                                
  |    +--------------------+                                                      
  |--> | set_pcie_port_type | set port type (downstream or upstream)               
  |    +--------------------+                                                      
  |    +---------------------+                                                     
  |--> | pci_dev_assign_slot | assign slot to dev                                  
  |    +---------------------+                                                     
  |                                                                                
  |--> print "[%04x:%04x] type %02x class %#08x\n"                                 
  |                                                                                
  |--> switch hdr type                                                             
  |--> case standard                                                               
  |    |    +--------------+                                                       
  |    |--> | pci_read_irq | read irq from cfg and save in pci_dev                 
  |    |    +--------------+                                                       
  |    |    +----------------+                                                     
  |    |--> | pci_read_bases | read a pci bar                                      
  |    |    +----------------+                                                     
  |    |    +-------------------+                                                  
  |    +--> | pci_subsystem_ids | read subsystem ids and save in pci_dev           
  |         +-------------------+                                                  
  |--> case bridge                                                                 
  |    ---> read bridge related info and save in pci_dev                           
  +--> case cardbus                                                                
       ---> (similar to standard operation)                                        
```

```
drivers/pci/probe.c                                           
+--------------------+                                         
| set_pcie_port_type | : set port type (downstream or upstream)
+-|------------------+                                         
  |    +---------------------+                                 
  |--> | pci_find_capability | find pos of capability in header
  |    +---------------------+                                 
  |    +-----------------------+                               
  |--> | pci_read_config_dword | read config                   
  |    +-----------------------+                               
  |    +---------------------+                                 
  |--> | pci_upstream_bridge | find upstream bridge            
  |    +---------------------+                                 
  |                                                            
  +--> correct downstream/upstream type if necessary           
```

```
drivers/pci/probe.c                                             
+----------------+                                               
| pci_device_add | : add pci_dev to list in pci_bus, register dev
+-|--------------+                                               
  |    +----------------------+                                  
  |--> | pci_configure_device | (skip)                           
  |    +----------------------+                                  
  |                                                              
  |--> set up dev (from device framework)                        
  |                                                              
  |    +------------------------------------+                    
  |--> | pci_reassigndev_resource_alignment | (skip)             
  |    +------------------------------------+                    
  |    +-----------------------+                                 
  |--> | pci_init_capabilities | (skip)                          
  |    +-----------------------+                                 
  |                                                              
  |--> add pci_dev to pci_bus (list)                             
  |                                                              
  |    +--------------------+                                    
  |--> | pci_set_msi_domain | (skip)                             
  |    +--------------------+                                    
  |    +------------+                                            
  +--> | device_add |                                            
       +------------+                                            
```

```
drivers/pci/probe.c                                                                   
+------------------------+                                                             
| pci_scan_bridge_extend | : prepare child pci_bus, add to list of parent, scan the bus
+-|----------------------+                                                             
  |    +-----------------------+                                                       
  |--> | pci_read_config_dword | read buses (primary + secondary + subordinate)        
  |    +-----------------------+                                                       
  |    +-----------------+                                                             
  |--> | pci_add_new_bus | prepare child pci_bus, add to list of parent (pci_bus)      
  |    +-----------------+                                                             
  |    +-------------------------+                                                     
  |--> | pci_bus_insert_busn_res | reserve resource (bus range)                        
  |    +-------------------------+                                                     
  |    +--------------------+                                                          
  +--> | pci_scan_child_bus | (recursive)                                              
       +--------------------+                                                          
```

```
drivers/pci/probe.c                                                                                      
+-----------------+                                                                                       
| pci_add_new_bus | : prepare child pci_bus, add to list of parent (pci_bus)                              
+-|---------------+                                                                                       
  |    +---------------------+                                                                            
  |--> | pci_alloc_child_bus | prepare child bus (set parent, install ops, set up resources), register dev
  |    +---------------------+                                                                            
  |                                                                                                       
  +--> add to list of parent (pci_bus)                                                                    
```

```
drivers/pci/probe.c                                                                                 
+---------------------+                                                                              
| pci_alloc_child_bus | : prepare child bus (set parent, install ops, set up resources), register dev
+--|------------------+                                                                              
   |    +---------------+                                                                            
   |--> | pci_alloc_bus | alloc child (pci_bus)                                                      
   |    +---------------+                                                                            
   |                                                                                                 
   |--> set parent (another bus, not the bridge dev)                                                 
   |                                                                                                 
   |--> instasl bus ops                                                                              
   |                                                                                                 
   |--> set up child's resources based on arg bridge's                                               
   |                                                                                                 
   |    +-----------------+                                                                          
   +--> | device_register | register the bus (dev)                                                   
        +-----------------+                                                                          
```

```
drivers/pci/setup-bus.c                                            
+-------------------------------------+                             
| pci_assign_unassigned_bus_resources | : set up io/mmio for bridges
+-|-----------------------------------+                             
  |                                                                 
  |--> (ignore size handling)                                       
  |                                                                 
  |    +----------------------------+                               
  +--> | __pci_bus_assign_resources | set up io or mmio for bridges 
       +----------------------------+                               
```

```
drivers/pci/bus.c                                                                                
+---------------------+                                                                           
| pci_bus_add_devices | : recursively for each pci_dev: create sysfs/proc files & attach to driver
+-|-------------------+                                                                           
  |                                                                                               
  |--> for each pci_dev on the given bus                                                          
  |    |                                                                                          
  |    |    +--------------------+                                                                
  |    +--> | pci_bus_add_device | create sysfs/proc files, attach device to driver, lable 'added'
  |         +--------------------+                                                                
  |                                                                                               
  +--> for each handled pci_dev on the given bus                                                  
       |                                                                                          
       |--> get child dev                                                                         
       |                                                                                          
       |    +---------------------+                                                               
       +--> | pci_bus_add_devices | (recursively apply to child dev)                              
            +---------------------+                                                               
```

```
drivers/pci/bus.c                                                                      
+--------------------+                                                                  
| pci_bus_add_device | : create sysfs/proc files, attach device to driver, lable 'added'
+-|------------------+                                                                  
  |    +----------------------------+                                                   
  |--> | pci_create_sysfs_dev_files | create sysfs files                                
  |    +----------------------------+                                                   
  |    +------------------------+                                                       
  |--> | pci_proc_attach_device | create proc files                                     
  |    +------------------------+                                                       
  |    +----------------------+                                                         
  |--> | pci_bridge_d3_update | (skip)                                                  
  |    +----------------------+                                                         
  |    +---------------+                                                                
  |--> | device_attach | attach device for driver to probe                              
  |    +---------------+                                                                
  |                                                                                     
  +--> label flag 'added'                                                               
```
