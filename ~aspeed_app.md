### ast-video

```
 video/main.cc                                                            
 [main]                                                                   
 |                                                                        
 |--> [get_driver_version] get version (in our case: v2)                  
 |                                                                        
 +--> [main_v2] parse arguments, get frame, send to client or save to file
```

```
 video/main2.cc                                                           
 [main_v2] : parse arguments, get frame, send to client or save to file   
 |                                                                        
 |--> alloc 'Video'                                                       
 |                                                                        
 |--> handle argumetns                                                    
 |       m: mode                                                          
 |       a: format                                                        
 |       q: quality                                                       
 |       p: sampling                                                      
 |       f: fps                                                           
 |       s: streaming                                                     
 |       c: times                                                         
 |       h: help                                                          
 |       t: test                                                          
 |       i: id                                                            
 |                                                                        
 +--> if streaming                                                        
 |    |                                                                   
 |    |--> [net_setup] setup socket, listen and accept                    
 |    |                                                                   
 |    |--> [Video::start] configure the video device                      
 |    |                                                                   
 |    |--> endless loop                                                   
 |    |    |                                                              
 |    |    |--> [Video::getFrame] dequeue buf, enqueue other available buf
 |    |    |                                                              
 |    |    +--> [transfer] send frames to client                          
 |    |    |                                                              
 |    |    +--> resize if needed                                          
 |    |                                                                   
 |    +--> [Video::stop] turn off streaming, reest structure              
 |                                                                        
 +--> else (not streaming)                                                
      |                                                                   
      |--> [Video::start] configure the video device                      
      |                                                                   
      |--> [Video::getFrame] dequeue buf, enqueue other available buf     
      |                                                                   
      |--> [save2file] save frame to 'capture%d.jpg'                      
      |                                                                   
      +--> [Video::stop] turn off streaming, reest structure              
```

```
 video/main.cc                                
 [net_setup] : setup socket, listen and accept
 |                                            
 |--> alloc buffer                            
 |                                            
 |--> alloc socket                            
 |                                            
 |--> [setsockopt] set size of send-buffer    
 |                                            
 |--> [bind] bind socket to addr              
 |                                            
 |--> [listen]                                
 |                                            
 +--> [accept]                                                             
 video/main.cc                                
 [net_setup] : setup socket, listen and accept
 |                                            
 |--> alloc buffer                            
 |                                            
 |--> alloc socket                            
 |                                            
 |--> [setsockopt] set size of send-buffer    
 |                                            
 |--> [bind] bind socket to addr              
 |                                            
 |--> [listen]                                
 |                                            
 +--> [accept]                                
```

```
 video/ikvm_video.cpp                                 
 [Video::start] : configure the video device          
 |                                                    
 |--> open '/dev/video0'                              
 |                                                    
 |--> ioctl to get capability                         
 |                                                    
 |--> ioctl to get video format                       
 |                                                    
 |--> ioctl to set video format                       
 |                                                    
 |--> ioctl to set video parameters (type, frame rate)
 |                                                    
 |--> ioctl to set video control (compression quality)
 |                                                    
 |--> ioctl to set video control (sampling)           
 |                                                    
 |--> ioctl to set video control (mode, quality)      
 |                                                    
 +--> [resize]                                        
```

```
 video/ikvm_video.cpp                                    
 [Video::resize] : get buffer and queue it back          
 |                                                       
 |--> if any buffer contains data, set need_resize = true
 |                                                       
 |--> if need_resize, ioctl to turn off stream           
 |                                                       
 |--> reset buffer (unmap, init structure)               
 |                                                       
 |--> if need_resize, ioctl to set timing (?)            
 |                                                       
 |--> ioctl to request streaming buffers                 
 |                                                       
 |--> for each buffer                                    
 |    |                                                  
 |    |--> setup 'buf' to query video data               
 |    |                                                  
 |    |--> buffer[i] = mmap(buf)                         
 |    |                                                  
 |    +--> ioctl to queue buf (?)                        
 |                                                       
 +--> ioctl to turn on streaming                         
```

```
 video/ikvm_video.cpp                                        
 [Video::getFrame] : dequeue buf, enqueue other available buf
 |                                                           
 |--> [select]                                               
 |                                                           
 |--> ioctl to dequeue buf                                   
 |    |                                                      
 |    |--> update last_frame_index                           
 |    |                                                      
 |    +--> update buffers[i]                                 
 |                                                           
 +--> for each buffer                                        
      |                                                      
      |--> if i == last_frame_idx, continue                  
      |                                                      
      +--> if buffers[i] isn't queued                        
           -                                                 
           +--> ioctl to queue 'buf'                         
```

### hid_gadget_app

```
 usb/hid_gadget_test.c                                                          
 [main]                                                                         
 |                                                                              
 |--> open dev file                                                             
 |                                                                              
 +--> endless loop                                                              
      |                                                                         
      |--> [select]                                                             
      |                                                                         
      +--> if input is from dev file                                            
      |    -                                                                    
      |    +--> read report (?)                                                 
      |                                                                         
      +--> if input is from stdin                                               
           |                                                                    
           |--> if keyboard                                                     
           |    -                                                               
           |    +--> [keyboard_fill_report] convert buffer data to keyboard code
           |                                                                    
           |--> elif mouse                                                      
           |    -                                                               
           |    +--> [mouse_fill_report] convert buffer data to mouse code      
           |                                                                    
           |--> elif joystick                                                   
           |    -                                                               
           |    +--> [joystick_fill_report] convert buffer data to joystick code
           |                                                                    
           |--> write report to dev file                                        
           |                                                                    
           +--> if no hold, clear report and write again                        
```

```
 usb/hid_gadget_test.c                                        
 [keyboard_fill_report] : convert buffer data to keyboard code
 -                                                            
 +--> for each token                                          
      |                                                       
      |--> if option, convert to code                         
      |                                                       
      |--> if data, convert to code                           
      |                                                       
      +--> if modifier, convert to code                       
```

### i2c-slave-mqueue

```
 i2c-slave-mqueue/i2c-slave-mqueue.c
 [main]                             
 |                                  
 |--> open dev file                 
 |                                  
 +--> endless loop                  
      |                             
      |--> [poll]                   
      |                             
      |--> [read]                   
      |                             
      |--> [clock_gettime]          
      |                             
      +--> print timestamp and data 
```

### i2c-test

```
 i2c-test/i2c-test.c                                                                
 [main]                                                                             
 |                                                                                  
 |--> handle arguments                                                              
 |        h: help                                                                   
 |        a: slave addr                                                             
 |        m: master dev name                                                        
 |        s: slave dev name                                                         
 |        c: loop count                                                             
 |        l: transfer len                                                           
 |        r: fill buffer with random data                                           
 |        d: debug                                                                  
 |                                                                                  
 |--> if master info is provided                                                    
 |    -                                                                             
 |    +--> [pthread_create]                                                         
 |         [i2c_master_thread] write data to master dev (transfer to slave)         
 |                                                                                  
 +--> if slave info is provided                                                     
      -                                                                             
      +--> [pthread_create]                                                         
           [i2c_slave_thread] receive data from slave dev, compare with ground truth
```

```
 i2c-test/i2c-test.c                                               
 [i2c_master_thread] : write data to master dev (transfer to slave)
 |                                                                 
 |--> open master dev file                                         
 |                                                                 
 |--> ioctl to set slave addr                                      
 |                                                                 
 +--> endless loop                                                 
      |                                                            
      |--> write data to dev file                                  
      |                                                            
      +--> if anything goes wrong, break                           
```
