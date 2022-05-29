> Study case: Linux version 5.15.0 on OpenBMC

## Index

- [Introduction](#introduction)
- [Reference](#reference)

Here's the list of ARM processors that have their info compiled in the kernel.

```
...
80959e9c T __proc_info_begin
80959e9c t __v6_proc_info       <-- the one in our study case
80959ed0 t __v7_ca5mp_proc_info
80959f04 t __v7_ca9mp_proc_info
80959f38 t __v7_ca8_proc_info
80959f6c t __v7_cr7mp_proc_info
80959fa0 t __v7_cr8mp_proc_info
80959fd4 t __v7_ca7mp_proc_info
8095a008 t __v7_ca12mp_proc_info
8095a03c t __v7_ca15mp_proc_info
8095a070 t __v7_b15mp_proc_info
8095a0a4 t __v7_ca17mp_proc_info
8095a0d8 t __v7_ca73_proc_info
8095a10c t __v7_ca75_proc_info
8095a140 t __krait_proc_info
8095a174 t __v7_proc_info
8095a1a8 T __proc_info_end
...
```

```
                     +--------------------------------------------------------- cpu name   
                     |                           +----------------------------- main id reg
                     |                           |                         +--- cpu arch   
                     |                           |                         |               
                     |                           |                         |               
                     +-------------------------  +-------------------      +               
 [    0.000000] CPU: ARMv6-compatible processor [410fb767] revision 7 (ARMv7), cr=00c5387d 
 [    0.000000] CPU: VIPT aliasing data cache, unknown instruction cache                   
```

```
+-----------------+                                                                 
| setup_processor |                                                                 
+----|------------+                                                                 
     |    +---------------+                                                         
     |--> | read_cpuid_id | read main id register from co-processor (v6 in our case)
     |    +---------------+                                                         
     |    +------------------------+                                                
     |--> | __get_cpu_architecture | determine cpu arch (somehow it's v7 here)      
     |    +------------------------+                                                
     |                                                                              
     |--> print '"CPU: %s [%08x] revision %d (ARMv%s), cr=%08lx\n"'                 
     |                                                                              
     |    +-------------------+                                                     
     |--> | cpuid_init_hwcaps | determine elf hardware capability                   
     |    +-------------------+                                                     
     |    +--------------+                                                          
     |--> | cacheid_init |                                                          
     |    +---|----------+                                                          
     |        |                                                                     
     |        |--> determine cache id                                               
     |        |                                                                     
     |        +--> print "CPU: %s data cache, %s instruction cache\n"               
     |                                                                              
     |    +----------+                                                              
     +--> | cpu_init |                                                              
          +--|-------+                                                              
             |                                                                      
             +--> prepare stack space for irq/abt/und/fiq for one processor         
```

## <a name="introduction"></a> Introduction


## <a name="reference"></a> Reference




