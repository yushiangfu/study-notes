> Study case: Linux version 5.15.0 on OpenBMC

## Index

- [Introduction](#introduction)
- [Watchdog](#watchdog)
- [System Startup](#system-startup)
- [Cheat Sheet](#cheat-sheet)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

(TBD)


## <a name="system-startup"></a> System Startup

```
aspeed_wdt_init: register 'aspeed_watchdog_driver' to 'platform' bus
  aspeed_wdt_probe: 
```

```
drivers/watchdog/aspeed_wdt.c
+------------------+
| aspeed_wdt_probe | : map registers, install ops, register watchdog_dev
+-|----------------+
  |
  |--> alloc aspeed_wdt
  |
  |--> ioremap registers
  |
  |--> install 'aspeed_wdt_ops'
  |
  |    +-----------------------+
  |--> | watchdog_init_timeout | init timeout field
  |    +-----------------------+
  |
  |--> set wdt_ctrl based on properties
  |
  |--> if wdt is running (by checking hw reg)
  |    |
  |    |    +------------------+
  |    +--> | aspeed_wdt_start | ensure we're using the 1mhz clock source
  |         +------------------+
  |
  |--> if it's a 2500 or 2600 wdt
  |    -
  |    +--> set hw reg 'reset_width' based on dt properties
  |
  |    +-------------------------------+
  +--> | devm_watchdog_register_device | add watchdog_dev to list
       +-------------------------------+
```

```
drivers/watchdog/aspeed_wdt.c
+-------------------------------+
| devm_watchdog_register_device | : add watchdog_dev to list
+-|-----------------------------+
  |
  |--> alloc watchdog_device ptr as device resource
  |
  |    +--------------------------+
  |--> | watchdog_register_device | add watchdog_dev to list
  |    +--------------------------+
  |
  +--> have pointer point to the wdd
```

```                                                                                      
drivers/watchdog/watchdog_core.c                                                             
+--------------------------+                                                                  
| watchdog_register_device | : add watchdog_dev to list                                       
+-|------------------------+                                                                  
  |                                                                                           
  |--> if wtd_deferred_reg_done is set (not our case)                                         
  |    |                                                                                      
  |    |    +----------------------------+                                                    
  |    +--> | __watchdog_register_device | (skip)                                             
  |         +----------------------------+                                                    
  |                                                                                           
  +--> else                                                                                   
       |                                                                                      
       |    +------------------------------------+                                            
       +--> | watchdog_deferred_registration_add | add watchdog_dev to 'wtd_deferred_reg_list'
            +------------------------------------+                                            
```

## <a name="cheat-sheet"></a> Cheat Sheet


## <a name="reference"></a> Reference

