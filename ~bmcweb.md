```
src/webserver_main.cpp
+-----+
| run | : register lots of uri handlers
+-|---+
  |    +--------------------------+
  |--> | webassets::requestRoutes | for each file (html, css, ...) under /usr/share/www/, determine (web_path, callback) and register to rules
  |    +--------------------------+
  |    +-------------------------+
  |--> | obmc_kvm::requestRoutes | configure and prepare ops for "/kvm/0"
  |    +-------------------------+
  |    +-------------------------+
  |--> | redfish::RedfishService | prepare redfish_service
  |    +-------------------------+
  |    +-------------------------+
  |--> | HttpClient::getInstance | get http-client handler
  |    +-------------------------+
  |    +----------------------------------+
  |--> | EventServiceManager::getInstance | get event-service-manager handler
  |    +----------------------------------+
  |    +-----------------------------+
  |--> | dbus_monitor::requestRoutes | configure and prepare ops for "/subscribe"
  |    +-----------------------------+
  |    +-----------------------------+
  |--> | image_upload::requestRoutes | configure and prepare ops for "/upload/image/<str>" and "/upload/image"
  |    +-----------------------------+
  |    +-------------------------------+
  |--> | openbmc_mapper::requestRoutes | configure and prepare ops for "/bus/", "/bus/system/", "/list/", "/xyz/<path>"
  |    +-------------------------------+                               "/org/<path>", "/download/dump/<str>/",
  |                                                                    "/bus/system/<str>/", "/bus/system/<str>/<path>"
  |    +-----------------------------+
  |--> | obmc_console::requestRoutes | configure and prepare ops for "/console0"
  |    +-----------------------------+
  |    +------------------------+
  +--> | obmc_vm::requestRoutes | configure and prepare ops for "/vm/0/0"
  |    +------------------------+
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
  +--> | EventServiceManager::startEventLogMonitor | start event monitors on dir (/var/log) and file (/var/log/redfish)
  |    +-------------------------------------------+
  |    +------------------------------------------+
  +--> | hostname_monitor::registerHostnameSignal | register callback to hostname property change
       +------------------------------------------+
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

```
+-------------------------------------------+                                                                     
| EventServiceManager::startEventLogMonitor | : start event monitors on dir (/var/log) and file (/var/log/redfish)
+-|-----------------------------------------+                                                                     
  |    +-------------------+                                                                                      
  |--> | inotify_add_watch | add watch to "/var/log"                                                              
  |    +-------------------+                                                                                      
  |    +-------------------+                                                                                      
  |--> | inotify_add_watch | add watch to "/var/log/redfish"                                                      
  |    +-------------------+                                                                                      
  |    +-----------------------------------------------+                                                          
  +--> | EventServiceManager::watchRedfishEventLogFile | add watcher (for dir) or send out event logs (for file)  
       +-----------------------------------------------+                                                          
```

```
+-----------------------------------------------+                                                                                          
| EventServiceManager::watchRedfishEventLogFile | : add watcher (for dir) or send out event logs (for file)                                
+-|---------------------------------------------+                                                                                          
  |                                                                                                                                        
  +--> ->async_read_some                                                                                                                   
          +-------------------------------------------------------------------------------------------------------------------------------+
          |while we haven't handled all the data                                                                                          |
          ||                                                                                                                              |
          ||--> if the event is dir_watch_desc                                                                                            |
          ||    |                                                                                                                         |
          ||    |--> if it's about 'create'                                                                                               |
          ||    |    |                                                                                                                    |
          ||    |    |    +------------------+                                                                                            |
          ||    |    |--> | inotify_rm_watch | remove existing watcher                                                                    |
          ||    |    |    +------------------+                                                                                            |
          ||    |    |    +-------------------+                                                                                           |
          ||    |    |--> | inotify_add_watch | add new watcher for redfish-event-log file                                                |
          ||    |    |    +-------------------+                                                                                           |
          ||    |    |    +-----------------------------------------------+                                                               |
          ||    |    |--> | EventServiceManager::resetRedfishFilePosition | reset file offset                                             |
          ||    |    |    +-----------------------------------------------+                                                               |
          ||    |    |    +--------------------------------------------+                                                                  |
          ||    |    +--> | EventServiceManager::readEventLogsFromFile | read msg from files, prepare json format and send out event logs |
          ||    |         +--------------------------------------------+                                                                  |
          ||    |                                                                                                                         |
          ||    +--> elif 'delete' or 'move to'                                                                                           |
          ||         |                                                                                                                    |
          ||         |    +------------------+                                                                                            |
          ||         +--> | inotify_rm_watch | remove existing watcher                                                                    |
          ||              +------------------+                                                                                            |
          ||                                                                                                                              |
          |+--> elif the event is file_watch_desc                                                                                         |
          |     -                                                                                                                         |
          |     +--> if it's about 'modify'                                                                                               |
          |          |                                                                                                                    |
          |          |    +--------------------------------------------+                                                                  |
          |          +--> | EventServiceManager::readEventLogsFromFile | read msg from files, prepare json format and send out event logs |
          |               +--------------------------------------------+                                                                  |
          |+-----------------------------------------------+                                                                              |
          || EventServiceManager::watchRedfishEventLogFile | (recursive)                                                                  |
          |+-----------------------------------------------+                                                                              |
          +-------------------------------------------------------------------------------------------------------------------------------+
```

```
+--------------------------------------------+                                                                   
| EventServiceManager::readEventLogsFromFile | : read msg from files, prepare json format and send out event logs
+-|------------------------------------------+                                                                   
  |                                                                                                              
  |--> open file "/var/log/redfish"                                                                              
  |                                                                                                              
  |--> move file offset to the next log to be read                                                               
  |                                                                                                              
  |--> while we get still read a line                                                                            
  |    |                                                                                                         
  |    |--> update file offset                                                                                   
  |    |                                                                                                         
  |    |    +-----------------------------+                                                                      
  |    |--> | event_log::getUniqueEntryID | given current timestamp, determine entry id                          
  |    |    +-----------------------------+                                                                      
  |    |    +------------------------------+                                                                     
  |    |--> | event_log::getEventLogParams | get msg_id and timestamp from log_entry                             
  |    |    +------------------------------+                                                                     
  |    |    +-------------------------------------+                                                              
  |    |--> | event_log::getRegistryAndMessageKey | get registry_name and msg_key from msg_id                    
  |    |    +-------------------------------------+                                                              
  |    |                                                                                                         
  |    +--> save the msg attributes in 'eventRecords'                                                            
  |                                                                                                              
  |    +---------------------------------------------------+                                                     
  |--> | EventServiceManager::deleteTerminatedSubcriptions | delete terminated subscriptions                     
  |    +---------------------------------------------------+                                                     
  |                                                                                                              
  +--> for each entry in subscriptions_map                                                                       
       -                                                                                                         
       +--> if format_type is 'event'                                                                            
            |                                                                                                    
            |    +--------------------------------------+                                                        
            +--> | Subscription::filterAndSendEventLogs | prepare json format and send out event logs            
                 +--------------------------------------+                                                        
```

```
+-----------------------------+                                              
| event_log::getUniqueEntryID | : given current timestamp, determine entry id
+-|---------------------------+                                              
  |                                                                          
  |--> get timestamp from log entry                                          
  |                                                                          
  |--> increment index if curr_timestamp == prev_timestamp                   
  |                                                                          
  |--> prev_timestamp = curr_timestamp                                       
  |                                                                          
  +--> determine entry_id: <curr_timestamp>_<index>                          
```

```
+---------------------------------------------------+                                                                                                   
| EventServiceManager::deleteTerminatedSubcriptions | : delete terminated subscriptions                                                                 
+-|-------------------------------------------------+                                                                                                   
  |                                                                                                                                                     
  |--> for each item in subscriptions_map                                                                                                               
  |    -                                                                                                                                                
  |    +--> if it's terminated                                                                                                                          
  |         -                                                                                                                                           
  |         +--> add to delete_ids                                                                                                                      
  |                                                                                                                                                     
  |--> for each id in delete_ids                                                                                                                        
  |    |                                                                                                                                                
  |    |--> given id, get item from subscriptions_map                                                                                                   
  |    |                                                                                                                                                
  |    |--> erase it from subscriptions_map & subscriptions_config_map                                                                                  
  |    |                                                                                                                                                
  |    |    +-----------------+                                                                                                                         
  |    +--> | sd_journal_send | log event for subscription deletion                                                                                     
  |         +-----------------+                                                                                                                         
  |                                                                                                                                                     
  +--> if delete_ids does have something                                                                                                                
       |                                                                                                                                                
       |    +-------------------------------------------------+                                                                                         
       |--> | EventServiceManager::updateNoOfSubscribersCount | if we still have subscriber(s), register callback for metric_report, otherwise cancel it
       |    +-------------------------------------------------+                                                                                         
       |    +----------------------------------------------+                                                                                            
       +--> | EventServiceManager::persistSubscriptionData | prepare json data and write to persistent file                                             
            +----------------------------------------------+                                                                                            
```

```
+-------------------------------------------------+                                                                                           
| EventServiceManager::updateNoOfSubscribersCount | : if we still have subscriber(s), register callback for metric_report, otherwise cancel it
+-|-----------------------------------------------+                                                                                           
  |                                                                                                                                           
  |--> for each item in subscriptions_map                                                                                                     
  |    -                                                                                                                                      
  |    +--> update statistics                                                                                                                 
  |                                                                                                                                           
  |--> if we still have subscribers                                                                                                           
  |    |                                                                                                                                      
  |    |    +-------------------------------------------------+                                                                               
  |    +--> | EventServiceManager::registerMetricReportSignal | register callback for metric_report                                           
  |         +-------------------------------------------------+                                                                               
  |                                                                                                                                           
  +--> else                                                                                                                                   
       |                                                                                                                                      
       |    +---------------------------------------------------+                                                                             
       +--> | EventServiceManager::unregisterMetricReportSignal | unregister callback for metric_report                                       
            +---------------------------------------------------+                                                                             
```

```
+-------------------------------------------------+                                      
| EventServiceManager::registerMetricReportSignal | : register callback for metric_report
+-|-----------------------------------------------+                                      
  |                                                                                      
  |--> prepare match rule and callback                                                   
  |                                                                                      
  |    +-------------------------------------------+                                     
  +--> | EventServiceManager::getReadingsForReport | get readings for report             
       +-------------------------------------------+                                     
```

```
+-------------------------------------------+                                          
| EventServiceManager::getReadingsForReport | : get readings for report                
+-|-----------------------------------------+                                          
  |                                                                                    
  |--> get info from msg                                                               
  |                                                                                    
  |    +---------------------------------------------------+                           
  |--> | EventServiceManager::deleteTerminatedSubcriptions | (recursive)               
  |    +---------------------------------------------------+                           
  |                                                                                    
  +--> for each entry in subscriptions_map                                             
       -                                                                               
       +--> if format_type is metric_report                                            
            |                                                                          
            |    +------------------------------------+                                
            +--> | Subscription::filterAndSendReports | if not filtered out, send event
                 +------------------------------------+                                
```

```
+----------------------------------------------+                                                 
| EventServiceManager::persistSubscriptionData | : prepare json data and write to persistent file
+-|--------------------------------------------+                                                 
  |                                                                                              
  |--> set up service_config attributes                                                          
  |                                                                                              
  |    +-----------------------+                                                                 
  +--> | ConfigFile::writeData | prepare json data and write to persistent file                  
       +-----------------------+                                                                 
```

```
+-----------------------+                                                 
| ConfigFile::writeData | : prepare json data and write to persistent file
+-|---------------------+                                                 
  |                                                                       
  |--> prepare json data                                                  
  |                                                                       
  |--> for each auth_tokens                                               
  |    -                                                                  
  |    +--> save in sessions                                              
  |                                                                       
  |--> for each subscriptions_config_map                                  
  |    |                                                                  
  |    |--> for each header                                               
  |    |    -                                                             
  |    |    +--> save value in haders                                     
  |    |                                                                  
  |    +--> save attributes in subscriptions                              
  |                                                                       
  +--> write data to persistent_file                                      
```

```
+--------------------------------------+                                              
| Subscription::filterAndSendEventLogs | : prepare json format and send out event logs
+-|------------------------------------+                                              
  |                                                                                   
  |--> for each log_entry in event_records                                            
  |    |                                                                              
  |    |--> get attributes from log_entry                                             
  |    |                                                                              
  |    |--> continue if this log_entry is filtered out                                
  |    |                                                                              
  |    |--> add log_entry to log_entry_array                                          
  |    |                                                                              
  |    |    +--------------------------------+                                        
  |    +--> | event_log::formatEventLogEntry | format log_entry as json data          
  |         +--------------------------------+                                        
  |                                                                                   
  |--> prepare msg in json format                                                     
  |                                                                                   
  +--> ->sendEvent                                                                    
```

```
+------------------------------------------+                                                       
| hostname_monitor::registerHostnameSignal | : register callback to hostname property change       
+-|----------------------------------------+                                                       
  |                                                                                                
  +--> prepare match rule and callback                                                             
          +---------------------------------------------------------------------------------------+
          |+------------------------------------+                                                 |
          || hostname_monitor::onPropertyUpdate | load certificate and verify it, install to dbus |
          |+------------------------------------+                                                 |
          +---------------------------------------------------------------------------------------+
```

```
+------------------------------------+                                                  
| hostname_monitor::onPropertyUpdate | : load certificate and verify it, install to dbus
+-|----------------------------------+                                                  
  |                                                                                     
  |--> read msg                                                                         
  |                                                                                     
  |--> load certificate ("/etc/ssl/certs/https/server.pem")                             
  |                                                                                     
  |    +-----------------+                                                              
  |--> | X509_get_pubkey | get public key                                               
  |    +-----------------+                                                              
  |    +-------------+                                                                  
  |--> | X509_verify |                                                                  
  |    +-------------+                                                                  
  |    +-----------------------------------+                                            
  |--> | ensuressl::generateSslCertificate | "/tmp/hostname_cert.tmp"                   
  |    +-----------------------------------+                                            
  |    +--------------------------------------+                                         
  +--> | hostname_monitor::installCertificate | install certificate to dbus             
       +--------------------------------------+                                         
```
