```
+------+                                                                                                                                                       
| main |                                                                                                                                                       
+-|----+                                                                                                                                                       
  |    +--------------------------+                                                                                                                            
  |--> | webassets::requestRoutes | for each file (html, css, ...) under /usr/share/www/, determine (web_path, callback) and register to rules                 
  |    +--------------------------+                                                                                                                            
  |    +-------------------------+                                                                                                                             
  |--> | obmc_kvm::requestRoutes | configure and prepare ops for "/kvm/0"                                                                                      
  |    +-------------------------+                                                                                                                             
  |    +---------------------------------+                                                                                                                     
  |--> | eventservice_sse::requestRoutes | configure and prepare ops for "/redfish/v1/EventService/Subscriptions/SSE"                                          
  |    +---------------------------------+                                                                                                                     
  |    +------------------------+                                                                                                                              
  |--> | redfish::requestRoutes | configure and prepare ops for "/redfish/"                                                                                    
  |    +------------------------+                                                                                                                              
  |    +-----------------------------+                                                                                                                         
  |--> | dbus_monitor::requestRoutes | configure and prepare ops for "/subscribe"                                                                              
  |    +-----------------------------+                                                                                                                         
  |    +-----------------------------+                                                                                                                         
  |--> | image_upload::requestRoutes | configure and prepare ops for "/upload/image/<str>" and "/upload/image"                                                 
  |    +-----------------------------+                                                                                                                         
  |    +-------------------------------+                                                                                                                       
  |--> | openbmc_mapper::requestRoutes | configure and prepare ops for "/bus/", "/bus/system/", "/list/", "/xyz/<path>"                                        
  |    +-------------------------------+                               "/org/<path>", "/download/dump/<str>/", "/bus/system/<str>/", "/bus/system/<str>/<path>"
  |    +-----------------------------+                                                                                                                         
  |--> | obmc_console::requestRoutes | configure and prepare ops for "/console0"                                                                               
  |    +-----------------------------+                                                                                                                         
  |    +------------------------+                                                                                                                              
  +--> | obmc_vm::requestRoutes | configure and prepare ops for "/vm/0/0"                                                                                      
  |    +------------------------+                                                                                                                              
  |                                                                                                                                                            
  |--> if insecure_disable_xss_prevention                                                                                                                      
  |    |                                                                                                                                                       
  |    |    +-------------------------------+                                                                                                                  
  |    +--> | cors_preflight::requestRoutes | configure and prepare ops for "<str>"                                                                            
  |         +-------------------------------+                                                                                                                  
  |    +-----------------------------+                                                                                                                         
  |--> | login_routes::requestRoutes | configure and prepare ops for "/login"                                                                                  
  |    +-----------------------------+                                                                                                                         
  |    +-------------+                                                                                                                                         
  |--> | setupSocket | get a socket and save in App obj                                                                                                        
  |    +-------------+                                                                                                                                         
  |    +--------------------------+                                                                                                                            
  |--> | nbd_proxy::requestRoutes | configure and prepare ops for "/nbd/<str>"                                                                                 
  |    +--------------------------+                                                                                                                            
  |    +-------------------------------------------+                                                                                                           
  +--> | EventServiceManager::startEventLogMonitor |                                                                                                           
  |    +-------------------------------------------+                                                                                                           
  |    +------------------------------------------+                                                                                                            
  +--> | hostname_monitor::registerHostnameSignal |                                                                                                            
       +------------------------------------------+                                                                                                            
```

```
+-------------------------------------------+               
| EventServiceManager::startEventLogMonitor |               
+-|-----------------------------------------+               
  |    +-------------------+                                
  |--> | inotify_add_watch | add watch to "/var/log"        
  |    +-------------------+                                
  |    +-------------------+                                
  |--> | inotify_add_watch | add watch to "/var/log/redfish"
  |    +-------------------+                                
  |    +--------------------------+                         
  +--> | watchRedfishEventLogFile |                         
       +--------------------------+                         
```

```
+--------------------------+                                                                                                             
| webassets::requestRoutes | : for each file (html, css, ...) under /usr/share/www/, determine (web_path, callback) and register to rules
+-|------------------------+                                                                                                             
  |                                                                                                                                      
  |--> types = css, html, js, png, woff, woff2, gif, ico                                                                                 
  |            ttf, svg, eot, xml, json, jpg, jpeg, map                                                                                  
  |                                                                                                                                      
  |--> root_path = "/usr/share/www/"                                                                                                     
  |                                                                                                                                      
  +--> for each item in paths                                                                                                            
       |                                                                                                                                 
       |--> if it's a hidden dir or symlink                                                                                              
       |    -                                                                                                                            
       |    +--> disable recursion                                                                                                       
       |                                                                                                                                 
       +--> elif it's a regular file                                                                                                     
            |                                                                                                                            
            |--> adjust web_path if necessary                                                                                            
            |                                                                                                                            
            |--> determine content_type                                                                                                  
            |                                                                                                                            
            |    +-------------------+                                                                                                   
            +--> | App::routeDynamic | prepare callback and register to rules                                                            
                 +-------------------+                                                                                                   
                     +------------------------------------------+                                                                        
                     |prepare response header                   |                                                                        
                     |                                          |                                                                        
                     |prepare body based on given absolute_path |                                                                        
                     +------------------------------------------+                                                                        
```

```
+-------------+                                              
| setupSocket | : get a socket and save in App obj           
+-|-----------+                                              
  |    +---------------+                                     
  |--> | sd_listen_fds | ???                                 
  |    +---------------+                                     
  |                                                          
  |--> if listen_fd == 1 (successful?)                       
  |    |                                                     
  |    |    +-------------------+                            
  |    |--> | sd_is_socket_inet | (check if socket is valid?)
  |    |    +-------------------+                            
  |    |                                                     
  |    |--> if valid (start webserver on socket)             
  |    |    |                                                
  |    |    |    +-------------+                             
  |    |    +--> | App::socket | save socket in obj          
  |    |         +-------------+                             
  |    |                                                     
  |    +--> else (start webserver on port)                   
  |         |                                                
  |         |    +-----------+                               
  |         +--> | App::port | save port (18080) in obj      
  |              +-----------+                               
  |                                                          
  +--> else                                                  
       |                                                     
       |    +-----------+                                    
       +--> | App::port | save port (18080) in obj           
            +-----------+                                    
```
