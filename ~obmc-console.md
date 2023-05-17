- server

```
console-server.c                                                                                 
+------+                                                                                          
| main |                                                                                          
+-|----+                                                                                          
  |                                                                                               
  |--> handle options                                                                             
  |                                                                                               
  |--> alloc and set up 'console' (poll fd, ring buffer, ...)                                     
  |                                                                                               
  |    +-------------+                                                                            
  |--> | config_init | determine file name, parse it to set up 'config'                           
  |    +-------------+                                                                            
  |                                                                                               
  |--> determine tty-name (from options or config file)                                           
  |                                                                                               
  |    +----------+                                                                               
  |--> | tty_init | parse config & set lpc attrs, open tty dev and configure it, set up poll fd   
  |    +----------+                                                                               
  |    +-----------+                                                                              
  |--> | dbus_init | (skip)                                                                       
  |    +-----------+                                                                              
  |    +---------------+                                                                          
  |--> | handlers_init | for each compiled handler, call its ->init()                             
  |    +---------------+                                                                          
  |    +-------------+                                                                            
  +--> | run_console | endless loop: poll data, copy to ring buffer, call each consumer to take it
       +-------------+                                                                            
```

```
console-server.c                                                                         
+----------+                                                                              
| tty_init | : parse config & set lpc attrs, open tty dev and configure it, set up poll fd
+-|--------+                                                                              
  |                                                                                       
  |--> get 'lpc-address' from saved config                                                
  |                                                                                       
  |--> get 'sirq' from saved config                                                       
  |                                                                                       
  |--> get 'baud' from saved config                                                       
  |                                                                                       
  |    +-----------------+                                                                
  |--> | tty_find_device | find tty dev and save in 'console' struct                      
  |    +-----------------+                                                                
  |    +-------------+                                                                    
  +--> | tty_init_io | write lpc attrs, open tty dev and configure it, set up poll fd     
       +-------------+                                                                    
```

```
console-server.c                                                               
+-------------+                                                                 
| tty_init_io | : write lpc attrs, open tty dev and configure it, set up poll fd
+-|-----------+                                                                 
  |                                                                             
  |--> set sirq/lpc_address to sysfs attributes                                 
  |    (e.g., /sys/devices/platform/ahb/ahb:apb/1e787000.serial)                
  |                                                                             
  |--> open tty_dev and save fd                                                 
  |                                                                             
  |    +------------------+                                                     
  |--> | tty_init_termios | configure terminal                                  
  |    +------------------+                                                     
  |                                                                             
  +--> set up poll fd                                                           
```

```
console-server.c                                    
+------------------+                                 
| tty_init_termios | : configure terminal            
+-|----------------+                                 
  |    +-----------+                                 
  |--> | tcgetattr | get terminal parameters         
  |    +-----------+                                 
  |    +------------+                                
  |--> | cfsetspeed | given baud rate, set io speed  
  |    +------------+                                
  |    +-----------+                                 
  |--> | cfmakeraw | set mode 'raw'　（means no ldisc?)
  |    +-----------+                                 
  |    +-----------+                                 
  +--> | tcsetattr | set terminal parameters         
       +-----------+                                 
```

```
console-server.c                                                                                 
+-------------+                                                                                   
| run_console | : endless loop: poll data, copy to ring buffer, call each consumer to take it     
+-|-----------+                                                                                   
  |                                                                                               
  |--> register handler to 'SIGINT'                                                               
  |                                                                                               
  +--> endless loop                                                                               
       |                                                                                          
       |    +------+                                                                              
       |--> | poll |                                                                              
       |    +------+                                                                              
       |                                                                                          
       |--> if pollfd[n] has revents                                                              
       |    |                                                                                     
       |    |    +------------------+                                                             
       |    +--> | ringbuffer_queue | write data to ring buffer, call ->poll_fn() of each consumer
       |         +------------------+                                                             
       |    +--------------+                                                                      
       +--> | call_pollers | call each poller if it has revent                                    
            +--------------+                                                                      
```

- client
