```                                                                
 src/mdrv2_main.cpp                                             
 [main]                                                         
 |                                                              
 |--> request name 'xyz.openbmc_project.Smbios.MDR_V2'          
 |                                                              
 +--> [MDRV2] parse smbios file, register method 'GetRecordType'
```

```                                                                                    
 include/mdrv2.hpp                                                                  
 [MDRV2] : parse smbios file, register method 'GetRecordType'                       
 |                                                                                  
 |--> [MDRV2::agentSynchronizeData] read smbios file, update cpu/dimm/pcie_slot info
 |                                                                                  
 +--> register method 'GetRecordType'                                               
```

```                                                                                 
 src/mdrv2.cpp                                                                   
 [MDRV2::agentSynchronizeData] : read smbios file, update cpu/dimm/pcie_slot info
 |                                                                               
 |--> [MDRV2::readDataFromFlash] read header and remaining data from smbios file 
 |                                                                               
 +--> [MDRV2::systemInfoUpdate] from smbios, update cpu/dimm/pcie_slot info      
```

```                                                                        
 src/mdrv2.cpp                                                          
 [MDRV2::systemInfoUpdate] : from smbios, update cpu/dimm/pcie_slot info
 |                                                                      
 |--> setup match rule to call this function                            
 |                                                                      
 |--> [MDRV2::getTotalCpuSlot] count cpu# from smbios data              
 |                                                                      
 |--> for each cpu                                                      
 |    -                                                                 
 |    +--> [Cpu::infoUpdate] update cpu info                            
 |                                                                      
 |--> [MDRV2:::cs]              count dimm# from smbios data            
 |                                                                      
 |--> for each dimm                                                     
 |    -                                                                 
 |    +--> [Dimm::memoryInfoUpdate] update dimm info                    
 |                                                                      
 |--> [MDRV2::getTotalPcieSlot] count pcie_slot# from smbios data       
 |                                                                      
 |--> for each pcie_slot                                                
 |    -                                                                 
 |    +--> [Pcie::pcieInfoUpdate] update pcie_slot info                 
 |                                                                      
 +--> [System]                                                          
```

```                                                                    
 src/mdrv2.cpp                                                      
 [MDRV2::getTotalCpuSlot] : count cpu# from smbios data             
 -                                                                  
 +--> endless loop                                                  
      |                                                             
      |--> [getSMBIOSTypePtr] given type, get next node of that type
      |                                                             
      +--> count++                                                  
      |                                                             
      +--> [smbiosNextPtr] get next mpde                            
```

```                                                        
 src/cpu.cpp                                            
 [Cpu::infoUpdate] : update cpu info                    
 |                                                      
 |--> [Cpu::socket] fill info of cpu socket             
 |                                                      
 |--> set 'present' and 'functional' accordingly        
 |                                                      
 |--> if cpu isn't present, return                      
 |                                                      
 |--> set 'family', 'manufacturer', and 'id'            
 |                                                      
 |--> further set 'effectiveFamily' and 'effectiveModel'
 |                                                      
 |--> set other fields                                  
 |                                                      
 +--> setup association                                 
```
