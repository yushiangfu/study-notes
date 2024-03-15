not an independent repo, but part of the openbmc

```
meta-phosphor/recipes-phosphor/initrdscripts/files/obmc-init.sh             
+------+                                                                     
| init | handle mounts and switch root
+-|----+                                                                     
  |                                                                          
  |--> mount dev, sys, proc, and run                                         
  |                                                                          
  |--> create ro and rw folders under /run/initramfs                         
  |                                                                          
  |--> copy important folders and scripts into /run/initramfs                
  |                                                                          
  |--> prepare default paths and settings                                    
  |                                                                          
  |--> handle 'init-options' files                                           
  |                                                                          
  |--> print e.g., "rofs = mtd4 squashfs rwfs = mtd5 jffs2"                  
  |                                                                          
  |--> let debug script take over if specified                               
  |                                                                          
  |--> if consider downloading files, (skip)                                 
  |                                                                          
  |--> if there's any /image-*, move to /run/                                
  |                                                                          
  |--> clean rwfs if it's specified in /run/initramfs/init-options           
  |                                                                          
  |--> perform factory reset if it's specified in /run/initramfs/init-options
  |                                                                          
  |    +--------+                                                            
  |--> | update | update image-* if there's any                              
  |    +--------+                                                            
  |                                                                          
  |--> mount rofs                                                            
  |                                                                          
  |--> do some fsck                                                          
  |                                                                          
  |--> mount rwfs                                                            
  |                                                                          
  +--> create upper and work folders, and mount overlay                      
  |                                                                          
  +--> move mounts and switch root                                           
```

```
meta-phosphor/recipes-phosphor/initrdscripts/files/obmc-shutdown.sh                               
+----------+                                                                                       
| shutdown | : flash mtd if there's any img, set watchdog timeout and wait for reset               
+-|--------+                                                                                       
  |                                                                                                
  |--> unmount all loop mounts                                                                     
  |                                                                                                
  |--> if there's any /run/initramfs/image-*                                                       
  |    -                                                                                           
  |    +--> if /run/initramfs/update exists and is executable                                      
  |         |                                                                                      
  |         |--> if /dev/watchdog exists and is char dev                                           
  |         |    -                                                                                 
  |         |    +--> start watchdog -t 1 -T 5 (reset every 1 sec, reboot after 5 sec if not reset)
  |         |                                                                                      
  |         |    +--------+                                                                        
  |         |--> | update | update /run/initramfs/image-* to their corresponding mtd               
  |         |    +--------+                                                                        
  |         |                                                                                      
  |         |--> print "Flash update completed."                                                   
  |         |                                                                                      
  |         +--> set a new timeout and kill watchdog (still have some time left before the reset)  
  |                                                                                                
  |--> show remaining mounts                                                                       
  |                                                                                                
  |--> drain tty msg to console                                                                    
  |                                                                                                
  |--> execute script argument if it's provided                                                    
  |                                                                                                
  |--> print "Execute ${1-reboot} -f if all unmounted ok, or exec /init"                           
  |                                                                                                
  +--> exec /bin/sh                                                                                
```

```
meta-phosphor/recipes-phosphor/initrdscripts/files/obmc-update.sh                   
+--------+                                                                           
| update | : update /run/initramfs/image-* to their corresponding mtd                
+-|------+                                                                           
  |                                                                                  
  +--> ensure /proc, /sys, and /dev are mounted                                      
  |                                                                                  
  |    +---------+                                                                   
  +--> | findmtd | find mtd of 'rwfs' under /sys/class/mtd, e.g., /sys/class/mtd/mtd5
  |    +---------+                                                                   
  |                                                                                  
  |--> prepare paths and settings                                                    
  |                                                                                  
  |--> handle script argument, e.g., --no-save-files                                 
  |                                                                                  
  |--> if do_save is set                                                             
  |    |                                                                             
  |    |--> ensure the rwfs is mounted                                               
  |    |                                                                             
  |    |--> while able to read line from whitelist                                   
  |    |    -                                                                        
  |    |    +--> copy something from /run/initramfs/rw/cow to /run/save/cow          
  |    |                                                                             
  |    +--> unmount rwfs                                                             
  |                                                                                  
  |--> for each img in /run/initramfs/image-*                                        
  |    |                                                                             
  |    |    +---------+                                                              
  |    +--> | findmtd |                                                              
  |         +---------+                                                              
  |                                                                                  
  |         check if there's any mounted child mtd (shouldn't)                       
  |                                                                                  
  |--> if do_flash is set                                                            
  |    -                                                                             
  |    +--> for each img in list                                                     
  |         -                                                                        
  |         +--> flash img to mtd, and remove img aftetwards                         
  |                                                                                  
  |--> if to_ram is set                                                              
  |    -                                                                             
  |    +--> (skip)                                                                   
  |                                                                                  
  |--> if do_restore is set                                                          
  |    -                                                                             
  |    +--> copy something from /run/save/cow to /run/initramfs/rw/cow               
  |                                                                                  
  +--> if do_clean is set                                                            
       -                                                                             
       +--> remove /run/save/cow                                                     
```
