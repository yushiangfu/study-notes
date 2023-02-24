# pldmd

```
pldmd/pldmd.cpp                                                                                                               
+------+                                                                                                                       
| main |                                                                                                                       
+-|----+                                                                                                                       
  |                                                                                                                            
  |--> handle options ('verbose' only)                                                                                         
  |                                                                                                                            
  +--> create local socket                                                                                                     
  |                                                                                                                            
  |    +--------------------------+                                                                                            
  |--> | pldm::utils::readHostEID | get host eid from file                                                                     
  |    +--------------------------+                                                                                            
  |                                                                                                                            
  |--> if host eid exists                                                                                                      
  |    |                                                                                                                       
  |    |--> prepare host_pdr_handler                                                                                           
  |    |    +--------------------------------+                                                                                 
  |    |    | HostPDRHandler::HostPDRHandler | parse host_fru_json, add each entity info to host_pdr_handler                   
  |    |    +--------------------------------+                                                                                 
  |    |                                                                                                                       
  |    +--> prepare dbus_to_pldm_event_handler                                                                                 
  |                                                                                                                            
  |--> prepare oem_platform_handler                                                                                            
  |                                                                                                                            
  |    +--------------------------+                                                                                            
  |--> | Invoker::registerHandler | register request handler to target type (e.g., pldm_bios)                                  
  |    +--------------------------+                                                                                            
  |                                                                                                                            
  |--> prepare fru_handler                                                                                                     
  |                                                                                                                            
  |--> prepare platform_handler                                                                                                
  |                                                                                                                            
  |    +--------------------------+                                                                                            
  |--> | Invoker::registerHandler | register platform_handler to target type (e.g., pldm_platform)                             
  |    +--------------------------+                                                                                            
  |    +--------------------------+                                                                                            
  |--> | Invoker::registerHandler | register oem_platform_handler to target type (e.g., pldm_base)                             
  |    +--------------------------+                                                                                            
  |    +--------------------------+                                                                                            
  |--> | Invoker::registerHandler | register fru_handler to target type (e.g., pldm_fru)                                       
  |    +--------------------------+                                                                                            
  |    +--------+                                                                                                              
  +--> | part 0 | connect to mctp server, request service name, prepare callback for pollin, save host fw condition, enter loop
       +--------+                                                                                                              
```

```
host-bmc/host_pdr_handler.cpp                                                                    
+--------------------------------+                                                                
| HostPDRHandler::HostPDRHandler | : parse host_fru_json, add each entity info to host_pdr_handler
+-|------------------------------+                                                                
  |                                                                                               
  +--> if file host_fru_json exists                                                               
       |                                                                                          
       |--> parse json file                                                                       
       |                                                                                          
       |--> for each entity, add (entity_type, entity) into host_pdr_handler                      
       |                                                                                          
       +--> prepare match fule for properties change                                              
               +-------------------------------------------+                                      
               |service: $bus                              |                                      
               |object: "/xyz/openbmc_project/state/host0" |                                      
               |iface: "xyz.openbmc_project.State.Host"    |                                      
               +-------------------------------------------+-------------------------+            
               |if prop 'current_host_state' changes                                 |            
               |-                                                                    |            
               |+--> if value becomes "xyz.openbmc_project.State.Host.HostState.Off" |            
               |     -                                                               |            
               |     +--> delete all the remote terminus information                 |            
               +---------------------------------------------------------------------+            
```

```
pldmd/pldmd.cpp                                                                                                                            
+--------+                                                                                                                                  
| part 0 | : connect to mctp server, request service name, prepare callback for pollin, save host fw condition, enter loop                  
+-|------+                                                                                                                                  
  |                                                                                                                                         
  |--> connect to socket (unix, addr = mctp-mux)                                                                                            
  |                                                                                                                                         
  |--> write mctp_msg_type (pldm)                                                                                                           
  |                                                                                                                                         
  |--> prepare mctp_discovery_handler                                                                                                       
  |    +------------------------------+                                                                                                     
  |    | MctpDiscovery::MctpDiscovery | discover (query_dev_id) each object in mctp service                                                 
  |    +------------------------------+                                                                                                     
  |                                                                                                                                         
  |--> prepare callback                                                                                                                     
  |       +--------------------------------------------------------------------------------------------------------+                        
  |       |recv (peek)                                                                                             |                        
  |       |                                                                                                        |                        
  |       |recv (to local 'request_msg')                                                                           |                        
  |       |                                                                                                        |                        
  |       |+--------------+                                                                                        |                        
  |       || processRxMsg | if incoming msg ins't response: we prepare response, or otherwise handle that response |                        
  |       |+--------------+                                                                                        |                        
  |       |                                                                                                        |                        
  |       |if we prepared the response                                                                             |                        
  |       ||                                                                                                       |                        
  |       ||--> set up iov[]                                                                                       |                        
  |       ||                                                                                                       |                        
  |       ||    +------------+                                                                                     |                        
  |       ||--> | setsockopt | set buffer size                                                                     |                        
  |       ||    +------------+                                                                                     |                        
  |       ||    +---------+                                                                                        |                        
  |       |+--> | sendmsg |                                                                                        |                        
  |       |     +---------+                                                                                        |                        
  |       +--------------------------------------------------------------------------------------------------------+                        
  |                                                                                                                                         
  |--> request name "xyz.openbmc_project.PLDM"                                                                                              
  |                                                                                                                                         
  +--> have the above callback handle sockfd's pollin                                                                                       
  |                                                                                                                                         
  |--> if host_pdr_handler exists                                                                                                           
  |    |                                                                                                                                    
  |    |    +------------------------------------------+                                                                                    
  |    +--> | HostPDRHandler::setHostFirmwareCondition | request (get pldm ver), request (get pdr), build sensor map, request (get readings)
  |         +------------------------------------------+                                                                                    
  |                                                                                                                                         
  +--> enter loop                                                                                                                           
```

```
requester/mctp_endpoint_discovery.cpp                                                           
+------------------------------+                                                                 
| MctpDiscovery::MctpDiscovery | : discover (query_dev_id) each object in mctp service           
+-|----------------------------+                                                                 
  |                                                                                              
  |--> prepare method call and call it                                                           
  |       +--------------------------------------------+                                         
  |       |service: "xyz.openbmc_project.MCTP.Control" |                                         
  |       |object: "/xyz/openbmc_project/mctp"         |                                         
  |       |iface: "org.freedesktop.DBus.ObjectManager" |                                         
  |       |method: "GetManagedObjects"                 |                                         
  |       +--------------------------------------------+                                         
  |                                                                                              
  |--> for each object in reply_msg                                                              
  |    -                                                                                         
  |    +--> for each iface in object                                                             
  |         -                                                                                    
  |         +--> if iface is "xyz.openbmc_project.MCTP.Endpoint"                                 
  |              -                                                                               
  |              +--> if it has prop "EID" and "SupportedMessageTypes"                           
  |                   -                                                                          
  |                   +--> find 'pldm' type, if found: save to local var 'eids'                  
  |                                                                                              
  +--> if eids contains something                                                                
       |                                                                                         
       |    +------------------------------+                                                     
       +--> | Manager::handleMCTPEndpoints | prepare request (query_dev_id), register to handlers
            +------------------------------+                                                     
```

```
fw-update/manager.hpp                                                                       
+------------------------------+                                                             
| Manager::handleMCTPEndpoints | : prepare request (query_dev_id), register to handlers      
+-|----------------------------+                                                             
  |    +-------------------------------+                                                     
  +--> | InventoryManager::discoverFDs | prepare request (query_dev_id), register to handlers
       +-------------------------------+                                                     
```

```
fw-update/inventory_manager.cpp                                                                 
+-------------------------------+                                                                
| InventoryManager::discoverFDs | : prepare request (query_dev_id), register to handlers         
+-|-----------------------------+                                                                
  |                                                                                              
  |--> given eid, get instance id                                                                
  |                                                                                              
  |--> encode request                                                                            
  |                                                                                              
  |    +--------------------------+                                                              
  +--> | Handler::registerRequest | register request (query_dev_id) to 'handlers'                
       +--------------------------+ +------------------------------------------+                 
                                    | InventoryManager::queryDeviceIdentifiers |                 
                                    +------------------------------------------+                 
                                      ready desciptor info for eid, send request (get_fw_params),
                                      parse response (component info)                            
```

```
fw-update/inventory_manager.cpp                                                                                                          
+------------------------------------------+                                                                                              
| InventoryManager::queryDeviceIdentifiers | : ready desciptor info for eid, send request (get_fw_params), parse response (component info)
+-|----------------------------------------+                                                                                              
  |    +--------------------------------------+                                                                                           
  |--> | decode_query_device_identifiers_resp |                                                                                           
  |    +--------------------------------------+                                                                                           
  |                                                                                                                                       
  |--> for each descriptor                                                                                                                
  |    |                                                                                                                                  
  |    |--> if type != fw_update_vendor_defined                                                                                           
  |    |    -                                                                                                                             
  |    |    +--> save (type, data) in descriptors                                                                                         
  |    |                                                                                                                                  
  |    |--> else                                                                                                                          
  |    |    |                                                                                                                             
  |    |    |    +----------------------------------------+                                                                               
  |    |    |--> | decode_vendor_defined_descriptor_value |                                                                               
  |    |    |    +----------------------------------------+                                                                               
  |    |    |                                                                                                                             
  |    |    +--> save (type, data) in descriptors                                                                                         
  |    |                                                                                                                                  
  |    +--> advance to next descriptor                                                                                                    
  |                                                                                                                                       
  |--> add (eid, descriptors) to descriptor_map                                                                                           
  |                                                                                                                                       
  |    +----------------------------------------------------+                                                                             
  +--> | InventoryManager::sendGetFirmwareParametersRequest | send request (get_fw_params), get response, parse and save component info   
       +----------------------------------------------------+                                                                             
```

```
fw-update/inventory_manager.cpp                                                                                                  
+----------------------------------------------------+                                                                            
| InventoryManager::sendGetFirmwareParametersRequest | : send request (get_fw_params), get response, parse and save component info
+-|--------------------------------------------------+                                                                            
  |    +------------------------------------+                                                                                     
  |--> | encode_get_firmware_parameters_req |                                                                                     
  |    +------------------------------------+                                                                                     
  |    +--------------------------+                                                                                               
  +--> | Handler::registerRequest | register request (get_fw_params) to 'handlers'                                                
       +--------------------------+ +-----------------------------------------+                                                   
                                    | InventoryManager::getFirmwareParameters | parse component info and save to info_map         
                                    +-----------------------------------------+                                                   
```

```
requester/handler.hpp                                       
+--------------------------+                                 
| Handler::registerRequest | : register request to 'handlers'
+-|------------------------+                                 
  |                                                          
  |--> prepare callback for timeout                          
  |                                                          
  |--> prepare request_interface and timer                   
  |                                                          
  |--> let request_interface start                           
  |                                                          
  |--> let timer start                                       
  |                                                          
  +--> key = (request, handler, timer), and save in handlers 
```

```
fw-update/inventory_manager.cpp                                                       
+-----------------------------------------+                                            
| InventoryManager::getFirmwareParameters | : parse component info and save to info_map
+-|---------------------------------------+                                            
  |    +-------------------------------------+                                         
  |--> | decode_get_firmware_parameters_resp |                                         
  |    +-------------------------------------+                                         
  |                                                                                    
  |--> for each component                                                              
  |    |                                                                               
  |    |    +------------------------------------------------+                         
  |    +--> | decode_get_firmware_parameters_resp_comp_entry |                         
  |    |    +------------------------------------------------+                         
  |    |                                                                               
  |    |--> save (class_id, entry_id) in component_info                                
  |    |                                                                               
  |    +--> advance to next component                                                  
  |                                                                                    
  +--> save (eid, component_info) in component_info_map                                
```

```
pldmd/pldmd.cpp                                                                                                  
+--------------+                                                                                                  
| processRxMsg | : if incoming msg ins't response: we prepare response, or otherwise handle that response         
+-|------------+                                                                                                  
  |    +--------------------+                                                                                     
  |--> | unpack_pldm_header |                                                                                     
  |    +--------------------+                                                                                     
  |                                                                                                               
  |--> if msg_type isn't 'response'                                                                               
  |    |                                                                                                          
  |    |--> if pldm_type isn't 'fw_update'                                                                        
  |    |    |                                                                                                     
  |    |    |    +-----------------+                                                                              
  |    |    +--> | Invoker::handle | given pldm_type, call the corresponding handler to process it                
  |    |         +-----------------+                                                                              
  |    |                                                                                                          
  |    +--> else                                                                                                  
  |         |                                                                                                     
  |         |    +------------------------+                                                                       
  |         +--> | Manager::handleRequest | given command, decode & check & prepare response msg                  
  |              +------------------------+                                                                       
  |                                                                                                               
  +--> else if msg_type is 'response'                                                                             
       |                                                                                                          
       |    +-------------------------+                                                                           
       +--> | Handler::handleResponse | key=(eid,instance_id,type,cmd), get handler from handlers[key] and call it
            +-------------------------+                                                                           
```

```
fw-update/manager.hpp                                                                                                   
+------------------------+                                                                                               
| Manager::handleRequest | : given command, decode & check & prepare response msg                                        
+-|----------------------+                                                                                               
  |                                                                                                                      
  |    fw-update/update_manager.cpp                                                                                      
  |    +------------------------------+                                                                                  
  +--> | UpdateManager::handleRequest | : given command, decode & check & prepare response msg                           
       +-|----------------------------+                                                                                  
         |                                                                                                               
         +--> if we know the eid                                                                                         
              |                                                                                                          
              |--> switch arg command                                                                                    
              |--> case request_fw_data                                                                                  
              |    -    +------------------------------+                                                                 
              |    +--> | DeviceUpdater::requestFwData | decode request to check length/offset, prepare response msg     
              |         +------------------------------+                                                                 
              |--> case transfer_complete                                                                                
              |    -    +---------------------------------+                                                              
              |    +--> | DeviceUpdater::transferComplete | decode request to check transfer result, prepare response msg
              |         +---------------------------------+                                                              
              |--> case verify_complete                                                                                  
              |    -    +-------------------------------+                                                                
              |    +--> | DeviceUpdater::verifyComplete | decode request to check verify result, prepare response msg    
              |         +-------------------------------+                                                                
              +--> case apply_complete                                                                                   
                   -    +-------------------------------+                                                                
                   +--> | DeviceUpdater::verifyComplete | decode request to check apply result, prepare response msg     
                        +-------------------------------+                                                                
```

```
fw-update/device_updater.cpp                                                                 
+------------------------------+                                                              
| DeviceUpdater::requestFwData | : decode request to check length/offset, prepare response msg
+-|----------------------------+                                                              
  |    +----------------------------------+                                                   
  |--> | decode_request_firmware_data_req |                                                   
  |    +----------------------------------+                                                   
  |                                                                                           
  +--> check length, offset                                                                   
  |                                                                                           
  |--> build response msg (pldm header, completion code, ...)                                 
  |                                                                                           
  |    +-----------------------------------+                                                  
  +--> | encode_request_firmware_data_resp |                                                  
       +-----------------------------------+                                                  
```

```
requester/handler.hpp                                                                                  
+-------------------------+                                                                             
| Handler::handleResponse | : key=(eid,instance_id,type,cmd), get handler from handlers[key] and call it
+-|-----------------------+                                                                             
  |                                                                                                     
  |--> build key by eid/instance_id/type/cmd                                                            
  |                                                                                                     
  +--> if 'handlers' has such key                                                                       
       |                                                                                                
       |--> request_handler = handlers[key]                                                             
       |                                                                                                
       |--> stop request and timer                                                                      
       |                                                                                                
       |--> call request_handler                                                                        
       |                                                                                                
       +--> remove key from 'handlers'                                                                  
```

```
host-bmc/host_pdr_handler.cpp                                                                                                    
+------------------------------------------+                                                                                      
| HostPDRHandler::setHostFirmwareCondition | : request (get pldm ver), request (get pdr), build sensor map, request (get readings)
+-|----------------------------------------+                                                                                      
  |    +------------------------+                                                                                                 
  |--> | encode_get_version_req |                                                                                                 
  |    +------------------------+                                                                                                 
  |                                                                                                                               
  |--> prepare get_pldm_ver_handler                                                                                               
  |       +-------------------------------------------------------------------------------------------------------------------+   
  |       |+----------------------------+                                                                                     |   
  |       || HostPDRHandler::getHostPDR | send req (get pdr), build sensor map, send reqs (get reading) and update dbus props |   
  |       |+----------------------------+                                                                                     |   
  |       +-------------------------------------------------------------------------------------------------------------------+   
  |                                                                                                                               
  |    +--------------------------+                                                                                               
  +--> | Handler::registerRequest | register request (get_pldm_version) to 'handlers'                                             
       +--------------------------+ callback = prepared handler                                                                   
```

```
host-bmc/host_pdr_handler.cpp                                                                                          
+----------------------------+                                                                                          
| HostPDRHandler::getHostPDR | : send req (get pdr), build sensor map, send reqs (get reading) and update dbus props    
+-|--------------------------+                                                                                          
  |                                                                                                                     
  |--> determine record_handle                                                                                          
  |                                                                                                                     
  |    +--------------------+                                                                                           
  |--> | encode_get_pdr_req |                                                                                           
  |    +--------------------+                                                                                           
  |    +--------------------------+                                                                                     
  +--> | Handler::registerRequest | register request (get_pdr) to 'handlers'                                            
       +--------------------------+ +---------------------------------+                                                 
                                    | HostPDRHandler::processHostPDRs |                                                 
                                    +---------------------------------+                                                 
                                    get all host pdr, build sensor map, send requests (get reading) and update dbus prop
```

```
host-bmc/host_pdr_handler.cpp                                                                                                          
+---------------------------------+                                                                                                     
| HostPDRHandler::processHostPDRs | : get all host pdr, build sensor map, send requests (get reading) and update dbus prop              
+-|-------------------------------+                                                                                                     
  |    +---------------------+                                                                                                          
  |--> | decode_get_pdr_resp |                                                                                                          
  |    +---------------------+                                                                                                          
  |    +---------------------+                                                                                                          
  |--> | decode_get_pdr_resp | (different params)                                                                                       
  |    +---------------------+                                                                                                          
  |                                                                                                                                     
  |--> if type is 'entity_association'                                                                                                  
  |    |                                                                                                                                
  |    |    +-----------------------------------------+                                                                                 
  |    +--> | HostPDRHandler::mergeEntityAssociations | add to entity to association tree                                               
  |         +-----------------------------------------+                                                                                 
  |                                                                                                                                     
  |--> else                                                                                                                             
  |    |                                                                                                                                
  |    |--> prepare pdr_terminus_handle based on type                                                                                   
  |    |                                                                                                                                
  |    |--> if type is terminus_locator_pdr                                                                                             
  |    |    -                                                                                                                           
  |    |    +--> add tl_pdr to tl_pdr_info                                                                                              
  |    |                                                                                                                                
  |    +--> update or add pdr_terminus_handle in repo                                                                                   
  |                                                                                                                                     
  |--> if there's no next record                                                                                                        
  |    |                                                                                                                                
  |    |    +--------------------------------------+                                                                                    
  |    |--> | HostPDRHandler::parseStateSensorPDRs | ensure terminus id exists, save (entry, info) in sensor map                        
  |    |    +--------------------------------------+                                                                                    
  |    |                                                                                                                                
  |    +--> if host is up                                                                                                               
  |         |                                                                                                                           
  |         |    +------------------------------------+                                                                                 
  |         +--> | HostPDRHandler::setHostSensorState | for each state sensor pdr: send req (get_readings) & handle resp (set dbus prop)
  |              +------------------------------------+                                                                                 
  |                                                                                                                                     
  +--> else                                                                                                                             
      |                                                                                                                                 
      |    +----------------------------+                                                                                               
      +--> | HostPDRHandler::getHostPDR | (for next record)                                                                             
           +----------------------------+                                                                                               
```

```
host-bmc/host_pdr_handler.cpp
+-----------------------------------------+                                    
| HostPDRHandler::mergeEntityAssociations | : add to entity to association tree
+-|---------------------------------------+                                    
  |    +-------------------------------------+                                 
  |--> | pldm_entity_association_pdr_extract |                                 
  |    +-------------------------------------+                                 
  |                                                                            
  |--> for each entity                                                         
  |    |                                                                       
  |    |--> given type, find parent                                            
  |    |                                                                       
  |    |--> find parent node in tree                                           
  |    |                                                                       
  |    +--> add entity to node (a.k.a. merge)                                  
  |                                                                            
  +--> if we did merge anything                                                
       -                                                                       
       +--> update pdr repo to refrect that                                    
```

```
host-bmc/host_pdr_handler.cpp                                                                                           
+------------------------------------+                                                                                   
| HostPDRHandler::setHostSensorState | : for each state sensor pdr: send req (get_readings) & handle resp (set dbus prop)
+-|----------------------------------+                                                                                   
  |                                                                                                                      
  +--> for each state_sensor_pdr                                                                                         
       |                                                                                                                 
       |--> get pdr from its data                                                                                        
       |                                                                                                                 
       +--> for each (terminus_handle, terminus_info) in tl_pdr_info                                                     
            -                                                                                                            
            +--> if pdr_handle == terminus_handle                                                                        
                 |                                                                                                       
                 |    +--------------------------------------+                                                           
                 |--> | encode_get_state_sensor_readings_req |                                                           
                 |    +--------------------------------------+                                                           
                 |                                                                                                       
                 |--> prepare response_handler                                                                           
                 |       +---------------------------------------------------------------------------------+             
                 |       |+---------------------------------------+                                        |             
                 |       || decode_get_state_sensor_readings_resp |                                        |             
                 |       |+---------------------------------------+                                        |             
                 |       |                                                                                 |             
                 |       |for each sensor                                                                  |             
                 |       ||                                                                                |             
                 |       ||    +----------------------------+                                              |             
                 |       ||--> | emitStateSensorEventSignal | send signal to pldm service                  |             
                 |       ||    +----------------------------+                                              |             
                 |       ||    +----------------------------------------+                                  |             
                 |       |+--> | HostPDRHandler::handleStateSensorEvent | given state, set dbus properties |             
                 |       |     +----------------------------------------+                                  |             
                 |       +---------------------------------------------------------------------------------+             
                 |    +--------------------------+                                                                       
                 +--> | Handler::registerRequest | register request (get_state_sensor_readings) to 'handlers'            
                      +--------------------------+ callback = prepared response_handler                                  
```

### pldmtool

```
pldmtool/pldm_cmd_helper.cpp                                                                                             
+------------------------+                                                                                                
| CommandInterface::exec | : create msg, send request & receive response, parse it                                        
+-|----------------------+                                                                                                
  |                                                                                                                       
  |--> given path/interface, get target service through obj mapper                                                        
  |                                                                                                                       
  |--> prepare method call                                                                                                
  |       +--------------------------------------------+                                                                  
  |       |service: $service                           |                                                                  
  |       |object: "/xyz/openbmc_project/pldm"         |                                                                  
  |       |iface: "xyz.openbmc_project.PLDM.Requester" |                                                                  
  |       |method: "GetInstanceId"                     |                                                                  
  |       +--------------------------------------------+                                                                  
  |                                                                                                                       
  |--> call it, and get reply                                                                                             
  |                                                                                                                       
  |--> call ->createRequestMsg(), e.g.,                                                                                   
  |    +--------------------------------+                                                                                 
  |    | GetPLDMTypes::createRequestMsg | encode request accordingly                                                      
  |    +--------------------------------+                                                                                 
  |                                                                                                                       
  |    +--------------------------------+                                                                                 
  |--> | CommandInterface::pldmSendRecv | set up socket (mctp-mux), connect to server, send/receive pldm msg, close socket
  |    +--------------------------------+                                                                                 
  |                                                                                                                       
  +--> call ->parseResponseMsg(), e.g.,                                                                                   
       +--------------------------------+                                                                                 
       | GetPLDMTypes::parseResponseMsg | parse response and print out                                                    
       +--------------------------------+                                                                                 
```

```
pldmtool/pldm_cmd_helper.cpp                                                                                        
+--------------------------------+                                                                                   
| CommandInterface::pldmSendRecv | : set up socket (mctp-mux), connect to server, send/receive pldm msg, close socket
+-|------------------------------+                                                                                   
  |                                                                                                                  
  |--> if mctp_eid != pldm_entity_id                                                                                 
  |    |                                                                                                             
  |    |    +-----------+                                                                                            
  |    |--> | pldm_open | set up socket (mctp-mux), connect to server, write 'pldm_type'                             
  |    |    +-----------+                                                                                            
  |    |    +----------------+                                                                                       
  |    |--> | pldm_send_recv | send and receive pldm msg                                                             
  |    |    +----------------+                                                                                       
  |    |                                                                                                             
  |    +--> copy response data to argument                                                                           
  |                                                                                                                  
  +--> else                                                                                                          
       |                                                                                                             
       |    +------------------+                                                                                     
       +--> | mctpSockSendRecv | set up socket (mctp-mux), connect to server, send/receive msg, shutdown socket      
       |    +------------------+                                                                                     
       |                                                                                                             
       +--> skip mctp header                                                                                         
```

```
src/requester/pldm.c                                        
+----------------+                                           
| pldm_send_recv | : send and receive pldm msg               
+-|--------------+                                           
  |    +-----------+                                         
  |--> | pldm_send | set up iov[] and send msg through socket
  |    +-----------+                                         
  |                                                          
  +--> endless loop                                          
       |                                                     
       |    +-----------+                                    
       |--> | pldm_recv | read pldm msg if there's nany      
       |    +-----------+                                    
       |                                                     
       +--> if received, break loop                          
```

```
src/requester/pldm.c                                 
+-----------+                                         
| pldm_recv | : read pldm msg if there's nany         
+-|---------+                                         
  |    +---------------+                              
  +--> | pldm_recv_any | read pldm msg if there's nany
       +---------------+                              
```

```
src/requester/pldm.c                                                        
+---------------+                                                            
| pldm_recv_any | : read pldm msg if there's nany                            
+-|-------------+                                                            
  |    +-----------+                                                         
  +--> | mctp_recv | read mctp socket if there's data, check if it's pldm msg
       +-----------+                                                         
```

```
pldmtool/pldm_cmd_helper.cpp                                                                  
+------------------+                                                                           
| mctpSockSendRecv | : set up socket, connect to server, send/receive pldm msg, shutdown socket
+-|----------------+                                                                           
  |                                                                                            
  |--> open socket "mctp-mux"                                                                  
  |                                                                                            
  |--> connect to server                                                                       
  |                                                                                            
  |--> write pldm type to socket                                                               
  |                                                                                            
  |--> check if it's able to receive                                                           
  |                                                                                            
  |--> receive response                                                                        
  |                                                                                            
  +--> shutdown socket                                                                         
```

```
pldmtool/pldm_base_cmd.cpp                                      
+--------------------------------+                               
| GetPLDMTypes::parseResponseMsg | : parse response and print out
+-|------------------------------+                               
  |    +-----------------------+                                 
  +--> | decode_get_types_resp |                                 
  |    +-----------------------+                                 
  |    +----------------+                                        
  +--> | printPldmTypes |                                        
       +----------------+                                        
```
