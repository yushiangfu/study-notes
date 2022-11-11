```
+------+                                                                                                  
| main |                                                                                                  
+-|----+                                                                                                  
  |    +-----------------------+                                                                          
  |--> | sd_bus_default_system | connect to system bus                                                    
  |    +-----------------------+                                                                          
  |    +-----------------------+                                                                          
  |--> | ipmi::ipmiChannelInit | (??? seems it's implemented elsewhere)                                   
  |    +-----------------------+                                                                          
  |    +-------------------+                                                                              
  |--> | setInterfaceIndex | currentInterfaceIndex = channel                                              
  |    +-------------------+                                                                              
  |    +----------------------+                                                                           
  |--> | Manager::managerInit | init manager and set up timer for clearning up                            
  |    +----------------------+                                                                           
  |    +-------------------------------------+                                                            
  |--> | command::registerGUIDChangeCallback | prepare callback to get system guid                        
  |    +-------------------------------------+                                                            
  |    +------------------------+                                                                         
  |--> | command::getSystemGUID | get system guid                                                         
  |    +------------------------+                                                                         
  |    +-------------------------------------------+                                                      
  |--> | sol::registerSolConfChangeCallbackHandler | prepare callback to update cache for sol conf change 
  |    +-------------------------------------------+                                                      
  |    +-------------------------------+                                                                  
  |--> | command::sessionSetupCommands | prepare commands for session setup and register to commandTable[]
  |    +-------------------------------+                                                                  
  |    +--------------------------------+                                                                 
  |--> | sol::command::registerCommands | prepare commands for sol and register to commandTable[]         
  |    +--------------------------------+                                                                 
  |    +------------------------+                                                                         
  |--> | EventLoop::setupSocket | ensure a udp_socket in hand, bind to network iface, request dbus service
  |    +------------------------+                                                                         
  |    +---------------------------+                                                                      
  +--> | EventLoop::startEventLoop | prepare handlers for signal and ipmi request                         
       +---------------------------+                                                                      
```

```
+----------------------+                                                      
| Manager::managerInit | : init manager and set up timer for clearning up     
+-|--------------------+                                                      
  |                                                                           
  |--> prepare manager                                                        
  |                                                                           
  |    +-----------------------------+                                        
  |--> | Manager::setNetworkInstance | set ipmiNetworkInstance = index        
  |    +-----------------------------+                                        
  |                                                                           
  |--> add session to 'essionsMap'                                            
  |                                                                           
  |    +------------------------+                                             
  +--> | scheduleSessionCleaner | set up timer (3s) to clear up stale sessions
       +------------------------+                                             
```

```
+-----------------------------+                                  
| Manager::setNetworkInstance | : set ipmiNetworkInstance = index
+-|---------------------------+                                  
  |                                                              
  |--> which channel and index are still valid                   
  |                                                              
  |        +----------------------+                              
  |------> | ipmi::getChannelInfo |                              
  |        +----------------------+                              
  |                                                              
  |------> if the medium_type is lan8032                         
  |                                                              
  |----------> get interface index and check if it == channel    
  |                                                              
  |----------> if it does, ipmiNetworkInstance = index           
  |                                                              
  |----------> ipmiNetworkChannelNumList[index] = channel        
  |                                                              
  +----------> index++                                           
  |                                                              
  +------> channel++                                             
```

```
+-------------------------------+                                                                    
| command::sessionSetupCommands | : prepare commands for session setup and register to commandTable[]
+-|-----------------------------+                                                                    
  |                                                                                                  
  |--> prepare commands for session set up                                                           
  |                                                                                                  
  |--> for each command                                                                              
  |                                                                                                  
  |        +------------------------+                                                                
  +------> | Table::registerCommand | ensure commandTable[command_id] is registered                  
           +------------------------+                                                                
```

```
+------------------------+                                                                           
| EventLoop::setupSocket | : ensure a udp_socket in hand, bind to network iface, request dbus service
+-|----------------------+                                                                           
  |                                                                                                  
  |--> determine iface                                                                               
  |                                                                                                  
  |--> ensure we have a udp_socket in hand (from supplied one and set up one)                        
  |                                                                                                  
  |    +--------------+                                                                              
  |--> | ::setsockopt | bind to device (network interface)                                           
  |    +--------------+                                                                              
  |    +--------------+                                                                              
  |--> | ::setsockopt | get packet info                                                              
  |    +--------------+                                                                              
  |    +--------------+                                                                              
  |--> | ::setsockopt | get packet info (ipv6)                                                       
  |    +--------------+                                                                              
  |                                                                                                  
  +--> request service: "xyz.openbmc_project.Ipmi.Channel." + channel (e.g., eth0)                   
```

```
+---------------------------+                                               
| EventLoop::startEventLoop | : prepare handlers for signal and ipmi request
+-|-------------------------+                                               
  |                                                                         
  +--> prepare signal handler for SIGINT and SIGTERM                        
       +------------------+                                                 
       | startRmcpReceive | prepare callback for ipmi request handling      
       +------------------+                                                 
```

```
+------------------+                                                                             
| startRmcpReceive | : prepare callback for ipmi request handling                                
+-|----------------+                                                                             
  |                                                                                              
  +--> prepare callback for rmcp_receive                                                         
       +-----------------------------+                                                           
       | EventLoop::handleRmcpPacket | read packet, update session data, forward request to ipmid
       +-----------------------------+                                                           
```

```
+-----------------------------+                                                               
| EventLoop::handleRmcpPacket | : read packet, update session data, forward request to ipmid  
+-|---------------------------+                                                               
  |                                                                                           
  |--> init handler by channel                                                                
  |                                                                                           
  |    +--------------------------+                                                           
  +--> | Handler::processIncoming | read packet, update session data, forward request to ipmid
       +--------------------------+                                                           
```

```
+--------------------------+                                                                   
| Handler::processIncoming | : read packet, update session data, forward request to ipmid      
+-|------------------------+                                                                   
  |    +------------------+                                                                    
  |--> | Handler::receive | read packet and flatten it                                         
  |    +------------------+                                                                    
  |    +-------------------------+                                                             
  |--> | Handler::updSessionData | update session data                                         
  |    +-------------------------+                                                             
  |    +-------------------------+                                                             
  +--> | Handler::executeCommand | get info from session, forward request to ipmid through dbus
       +-------------------------+                                                             
```

```
+-------------------------+                                                                  
| Handler::executeCommand | : get info from session, forward request to ipmid through dbus   
+-|-----------------------+                                                                  
  |                                                                                          
  |--> get cmd_id from in_msg                                                                
  |                                                                                          
  |--> check session                                                                         
  |                                                                                          
  |    +-----------------------+                                                             
  +--> | Table::executeCommand | get info from session, forward request to ipmid through dbus
       +-----------------------+                                                             
```

```
+-----------------------+                                                               
| Table::executeCommand | : get info from session, forward request to ipmid through dbus
+-|---------------------+                                                               
  |                                                                                     
  |--> lookup cmd in 'commandTable' by id                                               
  |                                                                                     
  |--> if not found (this might be the regular path...)                                 
  |                                                                                     
  |------> get user_id, priviledge, session_id from session                             
  |                                                                                     
  |        call service: "xyz.openbmc_project.Ipmi.Host"                                
  |             object: "/xyz/openbmc_project/Ipmi"                                     
  |             interface: "xyz.openbmc_project.Ipmi.Server"                            
  |             method: "execute"                                                       
  |                                                                                     
  |--> else                                                                             
  |                                                                                     
  |        +-------------------------------+                                            
  +------> | NetIpmidEntry::executeCommand | call functor                               
           +-------------------------------+                                            
```

```
+--------------------------+                                                                               
| command::activatePayload | : update sol params in manager, set channel in session, ready payload instance
+-|------------------------+                                                                               
  |                                                                                                        
  |--> return error if payload type != sol                                                                 
  |                                                                                                        
  |    +-----------------------------+                                                                     
  |--> | Manager::updateSOLParameter | get properties from target obj to update the sol parameters         
  |    +-----------------------------+                                                                     
  |                                                                                                        
  |--> return if it's disabled                                                                             
  |                                                                                                        
  |--> get session                                                                                         
  |                                                                                                        
  |    +--------------------------+                                                                        
  |--> | ipmi::ipmiUserGetUserId  | get user id (where's the definition)                                   
  |    +--------------------------+                                                                        
  |    +------------------------------+                                                                    
  |--> | Handler::setChannelInSession | save channel in session                                            
  |    +------------------------------+                                                                    
  |    +-------------------------------+                                                                   
  |--> | Manager::startPayloadInstance | ensure that's a console_socket ready, add context to map          
  |    +-------------------------------+                                                                   
  |                                                                                                        
  +--> set up size, port_num in response                                                                   
```

```
+-------------------------------+                                                                                                    
| Manager::startPayloadInstance | : ensure that's a console_socket ready, add context to map                                         
+-|-----------------------------+                                                                                                    
  |                                                                                                                                  
  |--> if payload_map is empty                                                                                                       
  |                                                                                                                                  
  |        +---------------------------+                                                                                             
  |------> | Manager::startHostConsole | set up console_socket, register callback for payload_instance, wait to read data from socket
  |        +---------------------------+                                                                                             
  |                                                                                                                                  
  +--> prepare context of payload and add to map                                                                                     
```

```
+---------------------------+                                                                                               
| Manager::startHostConsole | : set up console_socket, register callback for payload_instance, wait to read data from socket
+-|-------------------------+                                                                                               
  |                                                                                                                         
  |--> if console_socket isn't ready                                                                                        
  |    |                                                                                                                    
  |    |    +----------------------------+                                                                                  
  |    +--> | Manager::initConsoleSocket | set up console_socket and connect                                                
  |         +----------------------------+                                                                                  
  |                                                                                                                         
  |--> if callback hasn't registeired yet                                                                                   
  |    |                                                                                                                    
  |    |    +----------------------------------+                                                                            
  |    +--> | registerSOLServiceChangeCallback | stop all payload instances if property 'Enable' is set to false            
  |         +----------------------------------+                                                                            
  |                                                                                                                         
  +--> ->async_wait                                                                                                         
          +-------------------------------------------------------------------------------+                                 
          |+------------------------------+                                               |                                 
          || Manager::consoleInputHandler | read data from host_console_socket to manager |                                 
          |+------------------------------+                                               |                                 
          |+---------------------------+                                                  |                                 
          || Manager::startHostConsole | (recursive)                                      |                                 
          |+---------------------------+                                                  |                                 
          +-------------------------------------------------------------------------------+                                 
```

```
+------------------------------+                                                
| Manager::consoleInputHandler | : read data from host_console_socket to manager
+-|----------------------------+                                                
  |                                                                             
  |--> get read_size from cmd                                                   
  |                                                                             
  |--> prepare buffer                                                           
  |                                                                             
  |--> read data from host_console_socket to buffer                             
  |                                                                             
  +--> save buffer in manager (?)                                               
```

```
+----------------------------------+                                                                         
| registerSOLServiceChangeCallback | : stop all payload instances if property 'Enable' is set to false       
+-|--------------------------------+                                                                         
  |    +-----------------------------+                                                                       
  |--> | ipmid_get_sd_bus_connection | get system dbus                                                       
  |    +-----------------------------+                                                                       
  |                                                                                                          
  |--> given (obj, iface), get service (which is xyz.openbmc_project.Control.Service.Manager)                
  |                                                                                                          
  +--> prepare match rule and callback                                                                       
          +-------------------------------------------------------------------------------------------------+
          |get property 'Enabled'                                                                           |
          |                                                                                                 |
          |if it's set to 'false'                                                                           |
          ||                                                                                                |
          ||    +---------------------------------+                                                         |
          |+--> | Manager::stopAllPayloadInstance | erase all payload instances from map, stop host console |
          |     +---------------------------------+                                                         |
          +-------------------------------------------------------------------------------------------------+
```

```
+-----------------------------+                                                              
| Manager::updateSOLParameter | : get properties from target obj to update the sol parameters
+-|---------------------------+                                                              
  |                                                                                          
  |--> iface = "xyz.openbmc_project.Ipmi.SOL"                                                
  |                                                                                          
  |--> object: "/xyz/openbmc_project/ipmi/sol/" + ethdevice                                  
  |                                                                                          
  |--> if sol_service isn't set yet                                                          
  |    |                                                                                     
  |    |                  +------------------+                                               
  |    +--> sol_service = | ipmi::getService | get service that contains the arg object      
  |                       +------------------+                                               
  |    +----------------------------+                                                        
  |--> | ipmi::getAllDbusProperties | get properties of iface(xyz.openbmc_project.Ipmi.SOL)  
  |    +----------------------------+                                                        
  |                                                                                          
  +--> update sol::Manager member variables by these properties                              
```
