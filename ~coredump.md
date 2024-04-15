```
fs/coredump.c                                                                        
+-------------+                                                                       
| do_coredump | : given core_pattern, setup and call usermode helper, call ->core_dump
+-|-----------+                                                                       
  |    +---------------+                                                              
  |--> | coredump_wait | kill other thtreads, wait for them to be inactive (?)        
  |    +---------------+                                                              
  |    +-----------------+                                                            
  |--> | format_corename | alloc argv, fill data to cn and argv                       
  |    +-----------------+                                                            
  |                                                                                   
  |--> if ispipe (our case)                                                           
  |    |                                                                              
  |    |--> alloc helper argv                                                         
  |    |                                                                              
  |    |--> setup helper argv from previous cn and argv                               
  |    |                                                                              
  |    |    +---------------------------+                                             
  |    |--> | call_usermodehelper_setup |                                             
  |    |    +---------------------------+                                             
  |    |    +--------------------------+                                              
  |    +--> | call_usermodehelper_exec | (e.g., systemd-coredump)                     
  |         +--------------------------+                                              
  |                                                                                   
  |--> else                                                                           
  |    -                                                                              
  |    +--> (skip)                                                                    
  |                                                                                   
  |    +---------------+                                                              
  |--> | unshare_files |                                                              
  |    +---------------+                                                              
  |    +-------------------+                                                          
  |--> | dump_vma_snapshot | traverse vma list and save info in cprm                  
  |    +-------------------+                                                          
  |                                                                                   
  |--> call ->core_dump, e.g.,                                                        
  |    +---------------+                                                              
  |    | elf_core_dump |                                                              
  |    +---------------+                                                              
  |                                                                                   
  +--> if is pipe                                                                     
       |                                                                              
       |    +-----------------------+                                                 
       +--> | wait_for_dump_helpers | wait for pipe reader                            
            +-----------------------+                                                 
```

```
fs/coredump.c                                                                                          
+---------------+                                                                                       
| coredump_wait | : kill other thtreads, wait for them to be inactive (?)                               
+-|-------------+                                                                                       
  |                                                                                                     
  |--> setup core_state                                                                                 
  |                                                                                                     
  |    +-------------+                                                                                  
  |--> | zap_threads | setup task signal fields, kill other threads, clear 'sigpending', set 'dump core'
  |    +-------------+                                                                                  
  |                                                                                                     
  +--> if at least one thread got killed                                                                
       |                                                                                                
       |    +---------------------------+                                                               
       |--> | wait_for_completion_state | wait for what?                                                
       |    +---------------------------+                                                               
       |                                                                                                
       +--> wait for other threads to be inactive                                                       
```

```
fs/coredump.c                                                                                                                
+-------------+                                                                                                               
| zap_threads | : setup task signal fields, kill other threads, clear 'sigpending', set 'dump core'                           
+-|-----------+                                                                                                               
  |                                                                                                                           
  +--> if it's not group exit                                                                                                 
       |                                                                                                                      
       |                                                                                                                      
       |--> save core_state in signal                                                                                         
       |                                                                                                                      
       |    +-------------+                                                                                                   
       |--> | zap_process | setup current task signal fields, send 'kill' to other thread of signal, return # of killed thread
       |    +-------------+                                                                                                   
       |    +-----------------------+                                                                                         
       |--> | clear_tsk_thread_flag | clear flag 'sigpending'                                                                 
       |    +-----------------------+                                                                                         
       |                                                                                                                      
       +--> set flag 'dump core'                                                                                              
```

```
fs/coredump.c                                                                                                      
+-------------+                                                                                                     
| zap_process | : setup current task signal fields, send 'kill' to other thread of signal, return # of killed thread
+-|-----------+                                                                                                     
  |                                                                                                                 
  |--> setup task signal fields                                                                                     
  |                                                                                                                 
  +--> for each thread of the signal                                                                                
       |                                                                                                            
       |    +---------------------------+                                                                           
       +--> | task_clear_jobctl_pending | clear job ctrl pending bits                                               
       |    +---------------------------+                                                                           
       |                                                                                                            
       +--> if thread isn't current task                                                                            
            |                                                                                                       
            |    +-----------+                                                                                      
            |--> | sigaddset | add 'kill' signal                                                                    
            |    +-----------+                                                                                      
            |    +----------------+                                                                                 
            +--> | signal_wake_up |  wake up thread to face the kill                                                
                 +----------------+                                                                                 
```

```
fs/coredump.c                                                                                             
+-----------------+                                                                                        
| format_corename | : alloc argv, fill data to argv and cn                                                 
+-|---------------+                                                                                        
  |                                                                                                        
  |--> given core_pattern, check if it's pipe (true in our case)                                           
  |                                                                                                        
  |--> alloc argv                                                                                          
  |                                                                                                        
  +--> fill data to argv and cn                                                                            
                                                                                                           
                                                                                                           
                                                                                                           
                                                                                                           
                  |/usr/lib/systemd/systemd-coredump %P %u %g %s %t %c %h                                  
                                                                                                           
                  |+--------------------------------  |  |  |  |  |  |  -                                  
                  ||                                  |  |  |  |  |  |  +-> hostname                       
                  ||                                  |  |  |  |  |  +----> cpu the task ran on            
                  ||                                  |  |  |  |  +-------> time of coredump               
                  ||                                  |  |  |  +----------> signal that caused the coredump
                  ||                                  |  |  +-------------> gid                            
                  ||                                  |  +----------------> uid                            
                  ||                                  +-------------------> global pid                     
                  |+------------------------------------------------------> core name                      
                  +-------------------------------------------------------> pipe                           
```

### systemd-coredump

```
1. kernel detects the task crashes; refers to /proc/sys/kernel/core_pattern on what to do.
2. call usermode helper (systemd-coredump, 1st instance) and kernel sends coredump data to it thru pipe (stdin).
3. the 1st instance of systemd-coredump collects meta data and send to the specific socket (/run/systemd/coredump), along with input fd (stdin).
4. since systemd listens to the specific socket, it performs socket activation upon connection and spawns 2nd instance of systemd-coredump.
5. the 2nd instance of systemd-coredump recevies meta data and that input fd (stdin); creates corefile on disk accordingly.
```

```
src/coredump/coredump.c                                                                                             
+-----+                                                                                                              
| run | : 1st instance sends data, 2nd instance recieves and writes core file to disk                                
+-|---+                                                                                                              
  |                                                                                                                  
  |--> parse_config                                                                                                  
  |                                                                                                                  
  |--> if from kernel                                                                                                
  |    |                                                                                                             
  |    |--> if '--backtrace' is specified                                                                            
  |    |    |                                                                                                        
  |    |    |    +-------------------+                                                                               
  |    |    +--> | process_backtrace | (skip)                                                                        
  |    |         +-------------------+                                                                               
  |    |                                                                                                             
  |    +--> else                                                                                                     
  |         |                                                                                                        
  |         |    +----------------+                                                                                  
  |         +--> | process_kernel | prepare iovw, gather info, send to socket of '/run/systemd/coredump'             
  |              +----------------+ (spawned by kernel, parent is kthreadd)                                          
  |                                                                                                                  
  +--> else                                                                                                          
       |                                                                                                             
       |    +----------------+                                                                                       
       +--> | process_socket | receive data (e.g., input fd), generate core file from input_fd, send info to journald
            +----------------+ (spawned by systemd thru socket activation)                                           
```

```
src/coredump/coredump.c                                                                 
+----------------+                                                                       
| process_kernel | : prepare iovw, gather info, send to socket of '/run/systemd/coredump'
+-|--------------+                                                                       
  |    +----------+                                                                      
  |--> | iovw_new | alloc iovw                                                           
  |    +----------+                                                                      
  |                                                                                      
  |--> fill "MESSAGE_ID and "PRIORITY in iovw                                            
  |                                                                                      
  |    +-------------------------------+                                                 
  |--> | gather_pid_metadata_from_argv | fill argv in iovw                               
  |    +-------------------------------+                                                 
  |    +---------------------+                                                           
  |--> | gather_pid_metadata | fill task metadata in invw                                
  |    +---------------------+                                                           
  |                                                                                      
  |--> if it's journald or systemd that crashes                                          
  |    |                                                                                 
  |    |    +-----------------+                                                          
  |    +--> | submit_coredump | (skip)                                                   
  |         +-----------------+                                                          
  |                                                                                      
  +--> else                                                                              
       |                                                                                 
       |    +------------+                                                               
       +--> | send_iovec | setup socket of '/run/systemd/coredump', send iov/fd          
            +------------+                                                               
```

```
src/coredump/coredump.c                                             
+------------+                                                       
| send_iovec | : setup socket of '/run/systemd/coredump', send iov/fd
+-|----------+                                                       
  |    +--------+                                                    
  |--> | socket | alloc a unix socket                                
  |    +--------+                                                    
  |    +-------------------+                                         
  |--> | connect_unix_path | "/run/systemd/coredump"                 
  |    +-------------------+                                         
  |                                                                  
  |--> for each entry in iovw                                        
  |    |                                                             
  |    |--> prepare msg of that entry                                
  |    |                                                             
  |    |    +---------+                                              
  |    +--> | sendmsg |                                              
  |         +---------+                                              
  |    +-------------+                                               
  +--> | send_one_fd | send fd (e.g., stdin) thru control msg        
       +-------------+                                               
```

```
src/coredump/coredump.c                                                                                   
+----------------+                                                                                         
| process_socket | : receive data (e.g., input fd), generate core file from input_fd, send info to journald
+-|--------------+                                                                                         
  |                                                                                                        
  |--> endless loop                                                                                        
  |    |                                                                                                   
  |    |--> prepare iov & msg as container                                                                 
  |    |                                                                                                   
  |    |    +--------------+                                                                               
  |    |--> | recvmsg_safe | receive msg                                                                   
  |    |    +--------------+                                                                               
  |    |                                                                                                   
  |    |--> if received len is 0 (last part, there's input_fd only)                                        
  |    |    -                                                                                              
  |    |    +--> save input_fd                                                                             
  |    |         break                                                                                     
  |    |                                                                                                   
  |    +--> put iovec to iovw                                                                              
  |                                                                                                        
  |    +--------------+                                                                                    
  |--> | save_context | save iovw and other info in context                                                
  |    +--------------+                                                                                    
  |    +-----------------+                                                                                 
  +--> | submit_coredump | generate core file from input_fd, send info to journald                         
       +-----------------+                                                                                 
```

```
src/coredump/coredump.c                                                                                            
+-----------------+                                                                                                 
| submit_coredump | : generate core file from input_fd, send info to journald                                       
+-|---------------+                                                                                                 
  |    +-----------------+                                                                                          
  |--> | coredump_vacuum | if vaccum is necessary, delete some core files in /var/lib/systemd/coredump              
  |    +-----------------+                                                                                          
  |    +------------------------+                                                                                   
  +--> | save_external_coredump | open empty file, copy data from arg input_fd to it                                
  |    +------------------------+                                                                                   
  |    +-----------------+                                                                                          
  |--> | coredump_vacuum | (again, but exclude the newly created one)                                               
  |    +-----------------+                                                                                          
  |                                                                                                                 
  |--> if coredump_fd is valid                                                                                      
  |    |                                                                                                            
  |    |    +------------------+                                                                                    
  |    +--> | parse_elf_object | bc of disabled config, print "elfutils disabled, parsing ELF objects not supported"
  |         +------------------+                                                                                    
  |                                                                                                                 
  |--> print, e.g., 'Process 441 (adcsensor) of user 0 dumped core.'                                                
  |                                                                                                                 
  |--> collect other info                                                                                           
  |                                                                                                                 
  |    +------------------+                                                                                         
  +--> | sd_journal_sendv | send info to journald                                                                   
       +------------------+                                                                                         
```

```
src/coredump/coredump-vacuum.c                                                                  
+-----------------+                                                                              
| coredump_vacuum | : if vaccum is necessary, delete some core files in /var/lib/systemd/coredump
+-|---------------+                                                                              
  |                                                                                              
  |--> open "/var/lib/systemd/coredump"                                                          
  |                                                                                              
  +--> endless loop                                                                              
       |                                                                                         
       |    +-----------+                                                                        
       |--> | rewinddir |                                                                        
       |    +-----------+                                                                        
       |                                                                                         
       |--> for each dirent                                                                      
       |    |                                                                                    
       |    |    +--------------------+                                                          
       |    +--> | uid_from_file_name | get uid from corefile                                    
       |    |    +--------------------+                                                          
       |    |    +--------------------------+                                                    
       |    |--> | hashmap_ensure_allocated | ensure we have a hashmap                           
       |    |    +--------------------------+                                                    
       |    |                                                                                    
       |    |--> given uid, get entry from hashmap                                               
       |    |                                                                                    
       |    |--> if got                                                                          
       |    |    -                                                                               
       |    |    +--> update 'oldest' fields of entry if new timestamp is older                  
       |    |                                                                                    
       |    +--> else                                                                            
       |         -                                                                               
       |         +--> prepare node of timestamp                                                  
       |                                                                                         
       |    +------------------+                                                                 
       |--> | vacuum_necessary | determine if necessary to vacuum                                
       |    +------------------+                                                                 
       |                                                                                         
       |--> return if not                                                                        
       |                                                                                         
       |    +---------------------+                                                              
       +--> | unlinkat_deallocate | delete file                                                  
            +---------------------+                                                              
```

```
src/coredump/coredump.c                                                               
+------------------------+                                                             
| save_external_coredump | : open empty file, copy data from arg input_fd to it        
+-|----------------------+                                                             
  |    +-----------+                                                                   
  |--> | parse_uid |                                                                   
  |    +-----------+                                                                   
  |    +---------------+                                                               
  |--> | make_filename | file name = /var/lib/systemd/coredump/core.%s.%s.%s.%s.%s     
  |    +---------------+                                                               
  |    +-----------------------+                                                       
  |--> | open_tmpfile_linkable | open file, also create another tmp link pointing to it
  |    +-----------------------+                                                       
  |                                                                                    
  +--> check if tmp file is in temporary fs (probably not in our case)                 
  |                                                                                    
  |    +------------+                                                                  
  +--> | copy_bytes | copy data from arg input_fd to tmp file                          
       +------------+                                                                  
```
