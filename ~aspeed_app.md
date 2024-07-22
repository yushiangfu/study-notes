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

### i3c-test

```
skip
```

### mctp-ast

```
 mctp/mctp.c                                                                       
 [main]                                                                            
 |                                                                                 
 |--> handle arguments                                                             
 |        h: help                                                                  
 |        n: dev nme                                                               
 |        t: transfer                                                              
 |        r: receive                                                               
 |        i: eid                                                                   
 |        b: bus                                                                   
 |        d: dev                                                                   
 |        f: func                                                                  
 |        o: routing                                                               
 |        l: len                                                                   
 |        c: loop count                                                            
 |        s: som                                                                   
 |        e: eom                                                                   
 |                                                                                 
 |--> [aspeed_mctp_init] alloc astpcie and open dev                                
 |                                                                                 
 |--> [aspeed_mctp_register_default_handler] register default handler              
 |                                                                                 
 |--> [aspeed_mctp_get_mtu] ioctl to get mtu                                       
 |                                                                                 
 +--> endless loop                                                                 
      |                                                                            
      |--> if transfer                                                             
      |    -                                                                       
      |    +--> [aspeed_mctp_tx] setup xfer and write to dev                       
      |                                                                            
      +--> else (receive)                                                          
           -                                                                       
           +--> [aspeed_mctp_rx] read packet from dev, check if payload is expected
```

```
 mctp/mctp.c                                      
 [aspeed_mctp_tx] : setup xfer and write to dev   
 |                                                
 |--> alloc xfer buffer                           
 |                                                
 |--> setup header and fill payload               
 |                                                
 |--> while we have remaining len of data > mtu   
 |    |                                           
 |    |--> adjust xfer fields                     
 |    |                                           
 |    +--> [aspeed_mctp_send] write xfer to dev   
 |                                                
 +--> while there are still remaining len (<= mtu)
      |                                           
      |--> adjust xfer fields                     
      |                                           
      +--> [aspeed_mctp_send] write xfer to dev   
```

```
 mctp/mctp.c                                                          
 [aspeed_mctp_rx] : read packet from dev, check if payload is expected
 |                                                                    
 |--> [wait_for_message] poll                                         
 |                                                                    
 |--> [aspeed_mctp_recv] read data from dev                           
 |                                                                    
 +--> check if payload is expected                                    
```

### mctp-i3c

```
 mctp-i3c/mctp-i3c.c                                        
 [main] : setup 'xfer', transfer data                       
 |                                                          
 |--> handle arguments                                      
 |        h: help                                           
 |        d: device                                         
 |        p: calculate pec                                  
 |        r: read                                           
 |        w: write                                          
 |                                                          
 |--> open device                                           
 |                                                          
 |--> handle arguments                                      
 |    |                                                     
 |    |--> case read                                        
 |    |                                                     
 |    |    [rx_args_to_xfer] alloc buffer, save in xfer     
 |    |                                                     
 |    +--> case write                                       
 |                                                          
 |         [w_args_to_xfer] handle write data, finalize xfer
 |                                                          
 +--> for each xfer                                         
      -                                                     
      +--> [i3c_mctp_priv_xfer] transfer data               
```

```
 mctp-i3c/mctp-i3c.c                                      
 [w_args_to_xfer] : handle write data, finalize xfer      
 |                                                        
 |--> save write data into array                          
 |                                                        
 |--> if pec is specified                                 
 |    |                                                   
 |    |--> alloc buffer                                   
 |    |                                                   
 |    |--> convert string (in array) to number (in buffer)
 |    |                                                   
 |    +--> calculate pec, append to buffer                
 |                                                        
 |--> else                                                
 |    |                                                   
 |    |--> alloc buffer                                   
 |    |                                                   
 |    +--> convert string (in array) to number (in buffer)
 |                                                        
 +--> finalize xfer                                       
```

```
 mctp-i3c/mctp-i3c.c                             
 [i3c_mctp_priv_xfer] : transfer data            
 |                                               
 |--> if read_no_write                           
 |    |                                          
 |    |--> [wait_for_message] wait for input data
 |    |                                          
 |    +--> [i3c_mctp_recv] read from dev         
 |                                               
 +--> else                                       
      -                                          
      +--> [i3c_mctp_send] write to dev          
```

```
 mctp-i3c/mctp-i3c.c                     
 [wait_for_message] : wait for input data
 -                                       
 +--> whlie received == false            
      |                                  
      |--> [i3c_mctp_poll] poll for input
      |                                  
      +--> received = true               
```

### mctp-skt-recv

```
 mctp-socket/mctp-recv.c                          
 [main]                                           
 |                                                
 |--> handle arguments: debug, skip, type         
 |                                                
 |--> allock socket, setup addr and bind to socket
 |                                                
 +--> endless loop                                
      |                                           
      |--> peek and ensure buffer is enough       
      |                                           
      |--> [recvfrom]                             
      |                                           
      |--> if 'skip' is specified, continue       
      |                                           
      |--> unset 'tag owner'                      
      |                                           
      +--> [sendto] echo msg back                 
```

### mctp-skt-req

```
 mctp-socket/mctp-req.c                                                        
 [main]                                                                        
 |                                                                             
 |--> handle arguments: eid, net, ifindex, len, type, data, lladdr, skip, debug
 |                                                                             
 +--> [mctp_req] prepare socket, send req, receive resp                        
```

```
 mctp-socket/mctp-req.c                             
 [mctp_req] : prepare socket, send req, receive resp
 |                                                  
 |--> alloc socket and setup addr                   
 |                                                  
 |--> alloc rx buffer                               
 |                                                  
 |--> if arg 'lladdrlen' is provided                
 |    -                                             
 |    +--> adjust addr and set socket opt           
 |                                                  
 |--> [sendto]                                      
 |                                                  
 |--> peek incoming data, ensure rx buffer is enough
 |                                                  
 +--> [recvfrom]                                    
```

### md

```
 mem_utils/md.c                          
 [main]                                  
 |                                       
 |--> open /dev/mem                      
 |                                       
 |--> handle arguments: phy_addr and size
 |                                       
 |--> [mmap]                             
 |                                       
 |--> calc target virt_addr              
 |                                       
 +--> [print_buffer]                     
```

### mw

```
skip
```

### oob-pch-test

```
 espi_test/oob-pch-test.c                                     
 [main]                                                       
 |                                                            
 |--> handle arguments                                        
 |        h: help                                             
 |        d: dev path                                         
 |        t: temp                                             
 |        r: rtc                                              
 |                                                            
 |--> alloc packet                                            
 |                                                            
 |--> open dev file                                           
 |                                                            
 +--> switch request                                          
      case temp                                               
      -                                                       
      +--> [oob_req_pch_temp] fill req, send out, receive resp
                                                              
      case rtc                                                
      -                                                       
      +--> [oob_req_pch_rtc] fill req, send out, receive resp 
```

```
 espi_test/oob-pch-test.c                             
 [oob_req_pch_temp] : fill req, send out, receive resp
 |                                                    
 |--> fill request                                    
 |                                                    
 |--> ioctl to send request                           
 |                                                    
 +--> ioctl to receive response                       
```

### otp

```
 otp/otp.c                                                              
 [main]                                                                 
 |                                                                      
 |--> open /dev/aspeed-otp                                              
 |                                                                      
 |--> [chip_version] get chip version                                   
 |                                                                      
 |--> init info_cb accordingly                                          
 |                                                                      
 |--> [otp_read_conf]                                                   
 |                                                                      
 |--> fill protection_status based on config                            
 |                                                                      
 |--> if subcmd is 'read'                                               
 |    +--> [do_otpread] given arg (conf, data, or strap), read and print
 |                                                                      
 |--> elif subcmd is 'info'                                             
 |    +--> [do_otpinfo] given arg, read target info and print           
 |                                                                      
 |--> elif subcmd is 'pb'                                               
 |    +--> [do_otppb] given mode (conf, strap, or data), perform otp    
 |                                                                      
 |--> elif subcmd is 'protect'                                          
 |    +--> [do_otpprotect] program the conf                             
 |                                                                      
 |--> elif subcmd is 'scuprotect'                                       
 |    +--> [do_otp_scuprotect] program the conf                         
 |                                                                      
 |--> elif subcmd is 'prog'                                             
 |    +--> [do_otpprog] read image file, check and program image to otp 
 |                                                                      
 |--> elif subcmd is 'update'                                           
 |    +--> [do_otpupdate] update rev_id                                 
 |                                                                      
 |--> elif subcmd is 'rid'                                              
 |    +--> [do_otprid] print sw rev_id & otp rev_id                     
 |                                                                      
 |--> elif subcmd is 'retire'                                           
 |    +--> [do_otpretire] program retire_id to otp, verify              
 |                                                                      
 |--> elif subcmd is 'verify'                                           
 |    +--> [do_otpverify] try different keys to verify the image        
 |                                                                      
 +--> elif subcmd is 'invalid'                                          
      +--> [do_otpinvalid] invalidate a key                             
```

```
 otp/otp.c                                                      
 [do_otpread] : given arg (conf, data, or strap), read and print
 |                                                              
 |--> parse args to determine offset/count                      
 |                                                              
 |--> if arg[1] is 'conf'                                       
 |    -                                                         
 |    +--> [otp_print_conf] read conf and print                 
 |                                                              
 |--> elif arg[1] is 'data'                                     
 |    -                                                         
 |    +--> [otp_print_data] read data and print                 
 |                                                              
 +--> elif arg[1] is 'strap'                                    
      -                                                         
      +--> [otp_print_strap] read strap and print               
```

```
 otp/otp.c                                  
 [otp_print_conf] : read conf and print     
 |                                          
 |--> [otp_read_conf_buf] ioctl to read conf
 |                                          
 +--> print conf                            
```

```
 otp/otp.c                                          
 [otp_print_strap] : read strap and print           
 |                                                  
 |--> [otp_read_strap] read conf and fill 'otpstrap'
 |                                                  
 +--> print info                                    
```

```
 otp/otp.c                                       
 [otp_read_strap] : read conf and fill 'otpstrap'
 |                                               
 |--> init arg 'otpstrap'                        
 |                                               
 |--> [otp_read_conf_buf] ioctl to read conf     
 |                                               
 +--> fill 'otpstrap' accordingly                
```

```
 otp/otp.c                                            
 [do_otpinfo] : given arg, read target info and print 
 |                                                    
 |--> if arg is 'conf'                                
 |    -                                               
 |    +--> [otp_print_conf_info] read conf and print  
 |                                                    
 |--> elif arg is 'strap'                             
 |    -                                               
 |    +--> [otp_print_strap_info] read strap and print
 |                                                    
 |--> elif arg is 'scu'                               
 |    -                                               
 |    +--> [otp_print_scu_info] read conf and print   
 |                                                    
 +--> elif arg is 'key'                               
      -                                               
      +--> [otp_print_key_info] read data and print   
```

```
 otp/otp.c                                  
 [otp_print_conf_info] : read conf and print
 |                                          
 |--> [otp_read_conf_buf] ioctl to read conf
 |                                          
 +--> print                                 
```

```
 otp/otp.c                                                
 [do_otpprotect] : program the conf                       
 |                                                        
 |--> determine program addr                              
 |                                                        
 |--> [otp_read_conf] ioctl to read conf from program addr
 |                                                        
 +--> [otp_prog_conf_b] program the conf                  
```

```
 otp/otp.c                                                     
 [do_otpprog] : read image file, check and program image to otp
 |                                                             
 |--> handle arguments: path, force                            
 |                                                             
 |--> alloc buffer, read file into it                          
 |                                                             
 +--> [otp_prog_image] check and program image to otp          
```

```
 otp/otp.c                                                                                  
 [otp_prog_image] : check and program image to otp                                          
 |                                                                                          
 |--> parse header to setup image_layout                                                    
 |                                                                                          
 |--> [otp_verify_image] calculate digest, compare to know if it matches                    
 |                                                                                          
 |--> if image has 'data'                                                                   
 |    |                                                                                     
 |    |--> [otp_read_data_buf] read data                                                    
 |    |                                                                                     
 |    +--> [otp_check_data_image] given 'data', check if image can be programmed into otp   
 |                                                                                          
 |--> if image has 'conf'                                                                   
 |    |                                                                                     
 |    |--> [otp_read_conf] read conf                                                        
 |    |                                                                                     
 |    +--> [otp_check_conf_image] given 'conf', check if image can be programmed into otp   
 |                                                                                          
 |--> if image has 'strap'                                                                  
 |    |                                                                                     
 |    |--> [otp_read_strap] read strap                                                      
 |    |                                                                                     
 |    +--> [otp_check_strap_image] given 'strap', check if image can be programmed into otp 
 |                                                                                          
 |--> if image has 'scu pro'                                                                
 |    |                                                                                     
 |    |--> [otp_read_conf] read scu_pro                                                     
 |    |                                                                                     
 |    +--> [otp_check_scu_image] given ' scu pro', check if image can be programmed into otp
 |                                                                                          
 |--> if specified, print data/strap/conf/scu_pro                                           
 |                                                                                          
 +--> program data/strap/scu_pro/conf sequentially                                          
```

```
 otp/otp.c                                                           
 [otp_verify_image] : calculate digest, compare to know if it matches
 |                                                                   
 |--> given version, calculate digest accordingly                    
 |                                                                   
 +--> compare the calculated digest with pass-in one                 
```

```
 otp/otp.c                                     
 [do_otpupdate] : update rev_id                
 |                                             
 |--> parse arguments: force, update_num       
 |                                             
 +--> [otp_update_rid] update rev_id and verify
```

```
 otp/otp.c                                      
 [otp_update_rid] : update rev_id and verify    
 |                                              
 |--> [otp_read_conf_buf] read otp rev_id       
 |                                              
 |--> [sw_revid] get sw rev_id                  
 |                                              
 |--> [otp_print_revid] print otp rev_id        
 |                                              
 |--> for target range [rid_num, update_num)    
 |    -                                         
 |    +--> [otp_prog_conf_b] program conf to otp
 |                                              
 |--> [otp_read_conf_buf] read otp rev_id       
 |                                              
 +--> [otp_print_revid] print otp rev_id        
```

```
 otp/otp.c                                 
 [do_otprid] : print sw rev_id & otp rev_id
 |                                         
 |--> [otp_read_conf_buf] read otp rev_id  
 |                                         
 |--> [sw_revid] read sw rev_id            
 |                                         
 |--> print sw rev_id                      
 |                                         
 +--> [otp_print_revid] print otp rev_id   
```

```
 otp/otp.c                                             
 [do_otpretire] : program retire_id to otp, verify     
 |                                                     
 |--> handle arguments: force, retire_id               
 |                                                     
 +--> [otp_retire_key] program retire_id to otp, verify
```

```
 otp/otp.c                                          
 [otp_retire_key] : program retire_id to otp, verify
 |                                                  
 |--> [otp_read_conf] read otp_cfg                  
 |                                                  
 |--> [sec_key_num] get key num                     
 |                                                  
 |--> [otp_prog_conf_b] program retire_id to otp    
 |                                                  
 +--> verify if it's retired                        
```

```
 otp/otp.c                                                          
 [do_otpverify] : try different keys to verify the image            
 |                                                                  
 |--> parse arguments: path                                         
 |                                                                  
 |--> open file                                                     
 |                                                                  
 |--> alloc buffer and read file into it                            
 |                                                                  
 +--> [otp_verify_boot_image] try different keys to verify the image
```

```
 otp/otp.c                                                         
 [otp_verify_boot_image] : try different keys to verify the image  
 |                                                                 
 |--> [otp_read_data_buf] read 'data'                              
 |                                                                 
 |--> [parse_config] read config to setup sb_info                  
 |                                                                 
 |--> [parse_data] read data to setup key_list                     
 |                                                                 
 |--> read otp rev_id                                              
 |                                                                 
 |--> check rev_id                                                 
 |                                                                 
 |--> [sb_sha] calculate digest                                    
 |                                                                 
 +--> for each key                                                 
      |                                                            
      |--> [mode2_verify] decrypt signature and compare with digest
      |                                                            
      +--> if pass, break                                          
```

```
 otp/otp.c                                  
 [do_otpinvalid] : invalidate a key         
 |                                          
 |--> handle arguments: force, header_offset
 |                                          
 +--> [otp_invalid_key] invalidate a key    
```

```
 otp/otp.c                           
 [otp_invalid_key] : invalidate a key
 |                                   
 |--> [otp_read_data] read header    
 |                                   
 |--> [_otp_print_key] print key info
 |                                   
 |--> determine value to program     
 |                                   
 +--> for bit_offset in [14, 17]     
      -                              
      +--> [otp_prog_data_b] program 
```

### spd

```
 mem_utils/spd.c                                                     
 [main]                                                              
 |                                                                   
 |--> open /dev/mem                                                  
 |                                                                   
 |--> parse arguments: addr, size, ...                               
 |                                                                   
 |--> [mmap] map spi base (0x1e620000) to virtual addr               
 |                                                                   
 |--> [spim_init] given chip_select, init spi controller, map spi mem
 |                                                                   
 +--> [print_buffer] print spi data within specified range           
```

```
 mem_utils/spd.c                                         
 [print_buffer] : print spi data within specified range  
 -                                                       
 +--> while count                                        
      |                                                  
      |--> for line size                                 
      |    |                                             
      |    |--> [spim_read] read data thru spi controller
      |    |                                             
      |    +--> print out                                
      |                                                  
      +--> advance 'addr', update remaining 'count'      
```
