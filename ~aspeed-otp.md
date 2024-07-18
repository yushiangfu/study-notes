```
 drivers/soc/aspeed/aspeed-otp.c                                     
 [otp_ioctl]                                                         
 -                                                                   
 +--> switch cmd                                                     
      case read_data                                                 
      +--> [copy_from_user] for user specified range                 
      +--> [otp_read_data] perform soak, read data of specified range
      +--> [copy_to_user]                                            
                                                                     
      case read_conf                                                 
      +--> [copy_from_user] for user specified range                 
      +--> [otp_read_conf] perform soak, read conf of specified range
      +--> [copy_to_user]                                            
                                                                     
      case program_data                                              
      +--> [copy_from_user] for user specified offset and value      
      +--> [otp_prog_data] program data and verify                   
                                                                     
      case program_conf                                              
      +--> [copy_from_user] for user specified offset                
      +--> [otp_prog_conf] program conf and verify                   
                                                                     
      case ver                                                       
      +--> [copy_to_user]                                            
                                                                     
      case sw_rev_id                                                 
      +--> read sw rev id from reg                                   
      +--> [copy_to_user]                                            
                                                                     
      case key_num                                                   
      +--> read key num from reg                                     
      +--> [copy_to_user]                                            
```

```
 drivers/soc/aspeed/aspeed-otp.c                                   
 [otp_read_data] : perform soak, read data of specified range      
 |                                                                 
 |--> [otp_soak] perform soak                                      
 |                                                                 
 +--> for data in specified range                                  
      -                                                            
      +--> [otp_read_data_2dw] read data from reg, save in ctx data
```

```
 drivers/soc/aspeed/aspeed-otp.c               
 [otp_soak] : perform soak                     
 |                                             
 |--> if otp version is A2 or A3               
 |    -                                        
 |    +--> switch 'soak'                       
 |         case 0                              
 |         +--> [otp_write] write MRA          
 |         +--> [otp_write] write MRB          
 |         +--> [otp_write] write MR           
 |                                             
 |         case 1                              
 |         +--> [otp_write] write MRA          
 |         +--> [otp_write] write MRB          
 |         +--> [otp_write] write MR           
 |         +--> [aspeed_otp_write] write timing
 |                                             
 |         case 2                              
 |         +--> [otp_write] write MRA          
 |         +--> [otp_write] write MRB          
 |         +--> [otp_write] write MR           
 |         +--> [aspeed_otp_write] write timing
 |                                             
 |--> else (A0 or A1)                          
 |    -                                        
 |    +--> (skip)                              
 |                                             
 +--> [wait_complete]                          
```

```
 drivers/soc/aspeed/aspeed-otp.c                     
 [otp_write] : write arg 'otp_addr' and 'val' to regs
 |                                                   
 |--> [aspeed_otp_write] OTP_ADDR = otp_addr         
 |                                                   
 |--> [aspeed_otp_write] OTP_COMPARE_1 = val         
 |                                                   
 |--> [aspeed_otp_write] OTP_COMMAND = 0x23b1e362    
 |                                                   
 +--> [wait_complete]                                
```

```
 drivers/soc/aspeed/aspeed-otp.c             
 [otp_prog_data] : program and verify        
 [otp_prog_bit] : program and verify         
 |                                           
 |--> [otp_soak] perform soak                
 |                                           
 |--> [_otp_prog_bit] program data           
 |                                           
 +--> while we can retry                     
      |                                      
      |--> [verify_bit] read back and compare
      |                                      
      +--> if not the same, program again    
```
