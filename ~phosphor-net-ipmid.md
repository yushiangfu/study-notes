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
