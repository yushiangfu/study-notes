> Study case: Linux version 5.15.0 on OpenBMC

## Index

- [Introduction](#introduction)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

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

```
                                                          __v6_proc_info:                                                                             
                                                              .long   0x0007b000                                                                      
                                                              .long   0x0007f000                                                                      
                                                              ALT_SMP(.long \                                                                         
                                                                  PMD_TYPE_SECT | \                                                                   
                                                                  PMD_SECT_AP_WRITE | \                                                               
                                                                  PMD_SECT_AP_READ | \                                                                
 struct proc_info_list {                                          PMD_FLAGS_SMP)                                                                      
     unsigned int        cpu_val;                             ALT_UP(.long \                                                                          
     unsigned int        cpu_mask;                                PMD_TYPE_SECT | \                                                                   
     unsigned long       __cpu_mm_mmu_flags;                      PMD_SECT_AP_WRITE | \                                                               
     unsigned long       __cpu_io_mmu_flags;                      PMD_SECT_AP_READ | \                                                                
     unsigned long       __cpu_flush;                             PMD_FLAGS_UP)                                                                       
     const char      *arch_name;                              .long   PMD_TYPE_SECT | \                                                               
     const char      *elf_name;                                   PMD_SECT_XN | \                                                                     
     unsigned int        elf_hwcap;                               PMD_SECT_AP_WRITE | \                                                               
     const char      *cpu_name;                                   PMD_SECT_AP_READ                                                                    
     struct processor    *proc; -------------+                initfn  __v6_setup, __v6_proc_info                                                      
     struct cpu_tlb_fns  *tlb;               |                .long   cpu_arch_name                                                                   
     struct cpu_user_fns *user;              |                .long   cpu_elf_name                                                                    
     struct cpu_cache_fns    *cache;         |                /* See also feat_v6_fixup() for HWCAP_TLS */                                            
 };                                          |                .long   HWCAP_SWP|HWCAP_HALF|HWCAP_THUMB|HWCAP_FAST_MULT|HWCAP_EDSP|HWCAP_JAVA|HWCAP_TLS
                                             |                .long   cpu_v6_name                                                                     
                                             |-----------     .long   v6_processor_functions                                                          
                                             |                .long   v6wbi_tlb_fns                                                                   
                                             |                .long   v6_user_fns                                                                     
                                             |                .long   v6_cache_fns                                                                    
                                             |                                                                                                        
 struct processor {   -----------------------+                                                                                                        
     void (*_data_abort)(unsigned long pc);                                     v6_early_abort                                                        
     unsigned long (*_prefetch_abort)(unsigned long lr);                        v6_pabort                                                             
     void (*_proc_init)(void);                                                  cpu_v6_proc_init                                                      
     void (*check_bugs)(void);                                                  NULL                                                                  
     void (*_proc_fin)(void);                                                   cpu_v6_proc_fin                                                       
     void (*reset)(unsigned long addr, bool hvc);                               cpu_v6_reset                                                          
     int (*_do_idle)(void);                                          ------->   cpu_v6_do_idle                                                        
     void (*dcache_clean_area)(void *addr, int size);                           cpu_v6_dcache_clean_area                                              
     void (*switch_mm)(phys_addr_t pgd_phys, struct mm_struct *mm);             cpu_v6_switch_mm                                                      
     void (*set_pte_ext)(pte_t *ptep, pte_t pte, unsigned int ext);             cpu_v6_set_pte_ext                                                    
     unsigned int suspend_size;                                                 cpu_v6_suspend_size                                                   
     void (*do_suspend)(void *);                                                cpu_v6_do_suspend                                                     
     void (*do_resume)(void *);                                                 cpu_v6_do_resume                                                      
 };                                                                                                                                                   
```

## <a name="reference"></a> Reference

(TBD)
