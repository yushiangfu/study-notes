```
pci_realloc_setup_params:           duplicate some variables from init section
pcibus_class_init:                  register pci bus class
pci_driver_init:                    register both 'pci' and 'pcie_port' buses
pci_slot_init:                      create kset 'slots' for pci bus
pci_apply_final_quirks:             apply patch?
pci_proc_init:                      create /proc/bus/pci and attach device info
pcie_portdrv_init:                  register port services and port driver
pci_stub_init:                      (probably do nothing)
aspeed_pcie_driver_init
pci_resource_alignment_sysfs_init:  
pci_sysfs_init
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
