> Study case: Linux version 5.15.0 on AST2500 emulation

## Index

- [Introduction](#introduction)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

```
+-------------------------+                                                                                 
| blk_schedule_flush_plug | : flush both cb and mq list in task plug                                        
+------|------------------+                                                                                 
       |                                                                                                    
       |--> if task has a plug                                                                              
       |                                                                                                    
       |        +---------------------+                                                                     
       +------> | blk_flush_plug_list |                                                                     
                +-----|---------------+                                                                     
                      |    +----------------------+                                                         
                      |--> | flush_plug_callbacks | excute each cb node in plug list                        
                      |    +----------------------+                                                         
                      |                                                                                     
                      |--> if plug->mq_list isn't empty                                                     
                      |                                                                                     
                      |        +------------------------+                                                   
                      +------> | blk_mq_flush_plug_list | deliver each rq in plug list to elevator or driver
                               +------------------------+                                                   
```

## <a name="reference"></a> Reference

(TBD)
