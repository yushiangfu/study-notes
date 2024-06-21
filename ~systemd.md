```
main                                                                                                                                                      
|                                                                                                                                                          
|--> log_execution_mode          systemd[1]: Detected architecture arm.                                                                                    
|                                                                                                                                                          
|--> initialize_runtime                                                                                                                                    
|    |                                                                                                                                                     
|    |--> os_release_status      Welcome to Phosphor OpenBMC (Phosphor OpenBMC Project Reference Distro) 2.13.0-dev!                                       
|    |                                                                                                                                                     
|    +--> hostname_setup         systemd[1]: Hostname set to <ast2600-default>.                                                                            
|    |                                                                                                                                                     
|    +--> machine_id_setup       systemd[1]: Initializing machine ID from random generator.                                                                
|                                                                                                                                                          
|--> set_manager_settings                                                                                                                                  
|    -                                                                                                                                                     
|    +--> manager_set_watchdog                                                                                                                             
|         -                                                                                                                                                
|         +--> watchdog_setup    systemd[1]: Using hardware watchdog 'aspeed_wdt', version 0, device /dev/watchdog0                                        
|                                systemd[1]: Watchdog running with a timeout of 2min.                                                                      
|--> manager_startup                                                                                                                                       
|    -                                                                                                                                                     
|    +-->  manager_enumerate     /usr/lib/systemd/system/phosphor-ipmi-net@.socket:6: Invalid interface name, ignoring: sys-subsystem-net-devices-%i.device
|                                /usr/lib/systemd/system/phosphor-ipmi-net@.socket:6: Invalid interface name, ignoring: sys-subsystem-net-devices-%i.device
|                                                                                                                                                          
|--> do_queue_default_job        systemd[1]: Queued start job for default target Multi-User System.                                                        
|                                                                                                                                                          
+--> invoke_main_loop                                                                                                                                      
     -                                                                                                                                                     
     +--> manager_loop           [  OK  ] Created slice Slice /system/getty.                                                                               
                                 [  OK  ] Created slice Slice /system/hostlogger.                                                                          
                                 [  OK  ] Created slice Slice /system/mapper-wait.                                                                         
                                 ...                                                                                                                       
```

```
[  OK  ] Created slice Slice /system/getty.                         <-- system-getty.slice
[  OK  ] Created slice Slice /system/hostlogger.                    <-- system-hostlogger.slice
[  OK  ] Created slice Slice /system/mapper-wait.                   <-- system-mapper\x2dwait.slice
[  OK  ] Created slice Slice /system/modprobe.                      <-- system-modprobe.slice
[  OK  ] Created slice Slice /system/obmc-console.                  <-- system-obmc\x2dconsole.slice
[  OK  ] Created slice Slice /system/obmc-led-group-start.          <-- system-obmc\x2dled\x2dgroup\x2dstart.slice
[  OK  ] Created slice Slice /system/phosphor-certificate-manager.  <-- system-phosphor\x2dcertificate\x2dmanager.slice
[  OK  ] Created slice Slice /system/phosphor-ipmi-kcs.             <-- system-phosphor\x2dipmi\x2dkcs.slice
[  OK  ] Created slice Slice /system/phosphor-ipmi-net.             <-- system-phosphor\x2dipmi\x2dnet.slice
[  OK  ] Created slice Slice /system/serial-getty.                  <-- system-serial\x2dgetty.slice
[  OK  ] Created slice Slice /syste…project.Chassis.Control.Power.  <-- system-xyz.openbmc_project.Chassis.Control.Power.slice
[  OK  ] Created slice Slice /syste…c_project.State.Boot.PostCode.  <-- system-xyz.openbmc_project.State.Boot.PostCode.slice
[  OK  ] Started Dispatch Password …ts to Console Directory Watch.  <-- systemd-ask-password-console.path
[  OK  ] Started Forward Password R…uests to Wall Directory Watch.  <-- systemd-ask-password-wall.path
[  OK  ] Reached target Path Units.                                 <-- paths.target
[  OK  ] Reached target Remote File Systems.                        <-- remote-fs.target
[  OK  ] Reached target Slice Units.                                <-- slices.target
[  OK  ] Reached target Swaps.                                      <-- swap.target
[  OK  ] Listening on RPCbind Server Activation Socket.             <-- rpcbind.socket
[  OK  ] Reached target RPC Port Mapper.                            <-- rpcbind.target
[  OK  ] Listening on Syslog Socket.                                <-- syslog.socket
[  OK  ] Listening on Process Core Dump Socket.                     <-- systemd-coredump.socket
[  OK  ] Listening on Journal Socket (/dev/log).                    <-- systemd-journald-dev-log.socket
[  OK  ] Listening on Journal Socket.                               <-- systemd-journald.socket
[  OK  ] Listening on Network Service Netlink Socket.               <-- systemd-networkd.socket
[  OK  ] Listening on udev Control Socket.                          <-- systemd-udevd-control.socket
[  OK  ] Listening on udev Kernel Socket.                           <-- systemd-udevd-kernel.socket
         Mounting Kernel Debug File System...                       <-- sys-kernel-debug.mount
         Mounting Kernel Trace File System...                       <-- sys-kernel-tracing.mount
         Mounting Temporary Directory /tmp...                       <-- tmp.mount
         Starting Load Kernel Module configfs...                    <-- modprobe@configfs.service
         Starting Load Kernel Module efi_pstore...                  <-- modprobe@efi_pstore.service
         Starting Load Kernel Module fuse...                        <-- modprobe@fuse.service
         Starting RPC Bind...                                       <-- rpcbind.service
         Starting Journal Service...                                <-- systemd-journald.service
         Starting Generate network …ts from Kernel command line...  <-- systemd-network-generator.service
         Starting Remount Root and Kernel File Systems...           <-- systemd-remount-fs.service
         Starting Apply Kernel Variables...                         <-- systemd-sysctl.service
         Starting Coldplug All udev Devices...                      <-- systemd-udev-trigger.service
[  OK  ] Started RPC Bind.                                          <-- rpcbind.service
[  OK  ] Mounted Kernel Debug File System.                          <-- sys-kernel-debug.mount
[  OK  ] Mounted Kernel Trace File System.                          <-- sys-kernel-tracing.mount
[  OK  ] Mounted Temporary Directory /tmp.                          <-- tmp.mount
[  OK  ] Finished Load Kernel Module configfs.                      <-- modprobe@configfs.service
[  OK  ] Finished Load Kernel Module efi_pstore.                    <-- modprobe@efi_pstore.service
[  OK  ] Finished Load Kernel Module fuse.                          <-- modprobe@fuse.service
[  OK  ] Finished Generate network units from Kernel command line.  <-- systemd-network-generator.service
[  OK  ] Finished Remount Root and Kernel File Systems.             <-- systemd-remount-fs.service
[  OK  ] Finished Apply Kernel Variables.                           <-- systemd-sysctl.service
[  OK  ] Reached target Preparation for Network.                    <-- network-pre.target
         Mounting Kernel Configuration File System...               <-- sys-kernel-config.mount
         Starting Rebuild Hardware Database...                      <-- systemd-hwdb-update.service
         Starting Create System Users...                            <-- systemd-sysusers.service
[  OK  ] Mounted Kernel Configuration File System.                  <-- sys-kernel-config.mount
[  OK  ] Finished Rebuild Hardware Database.                        <-- systemd-hwdb-update.service
[  OK  ] Finished Create System Users.                              <-- systemd-sysusers.service
         Starting Create Static Device Nodes in /dev...             <-- systemd-tmpfiles-setup-dev.service
[  OK  ] Finished Create Static Device Nodes in /dev.               <-- systemd-tmpfiles-setup-dev.service
[  OK  ] Reached target Preparation for Local File Systems.         <-- local-fs-pre.target
         Mounting /var/volatile...                                  <-- var-volatile.mount
         Starting Rule-based Manage…for Device Events and Files...  <-- systemd-udevd.service
[  OK  ] Mounted /var/volatile.                                     <-- var-volatile.mount
         Starting Load/Save OS Random Seed...                       <-- systemd-random-seed.service
[  OK  ] Reached target Local File Systems.                         <-- local-fs.target
[  OK  ] Started Rule-based Manager for Device Events and Files.    <-- systemd-udevd.service
         Starting Network Configuration...                          <-- systemd-networkd.service
[  OK  ] Started Journal Service.                                   <-- systemd-journald.service
         Starting Flush Journal to Persistent Storage...            <-- systemd-journal-flush.service
[  OK  ] Finished Flush Journal to Persistent Storage.              <-- systemd-journal-flush.service
         Starting Create Volatile Files and Directories...          <-- systemd-tmpfiles-setup.service
[  OK  ] Started Network Configuration.                             <-- systemd-networkd.service
         Starting Wait for Network to be Configured...              <-- systemd-networkd-wait-online.service
[  OK  ] Finished Create Volatile Files and Directories.            <-- systemd-tmpfiles-setup.service
         Starting Run pending postinsts...                          <-- run-postinsts.service
         Starting Rebuild Journal Catalog...                        <-- systemd-journal-catalog-update.service
         Starting Network Name Resolution...                        <-- systemd-resolved.service
         Starting Network Time Synchronization...                   <-- systemd-timesyncd.service
[  OK  ] Finished Rebuild Journal Catalog.                          <-- systemd-journal-catalog-update.service
         Starting Update is Completed...                            <-- systemd-update-done.service
[  OK  ] Finished Update is Completed.                              <-- systemd-update-done.service
[  OK  ] Started Network Time Synchronization.                      <-- systemd-timesyncd.service
[  OK  ] Reached target System Time Set.                            <-- time-set.target
[  OK  ] Started Network Name Resolution.                           <-- systemd-resolved.service
[  OK  ] Reached target Network.                                    <-- network.target
[  OK  ] Reached target Host and Network Name Lookups.              <-- nss-lookup.target
[  OK  ] Started NFS status monitor for NFSv2/3 locking..           <-- nfs-statd.service
[  OK  ] Finished Load/Save OS Random Seed.                         <-- systemd-random-seed.service
[  OK  ] Finished Run pending postinsts.                            <-- run-postinsts.service
[  OK  ] Finished Coldplug All udev Devices.                        <-- systemd-udev-trigger.service
[  OK  ] Reached target System Initialization.                      <-- sysinit.target
[  OK  ] Started Daily rotation of log files.                       <-- logrotate.timer
[  OK  ] Started Daily Cleanup of Temporary Directories.            <-- systemd-tmpfiles-clean.timer
[  OK  ] Reached target Timer Units.                                <-- timers.target
[  OK  ] Listening on Avahi mDNS/DNS-SD Stack Activation Socket.    <-- avahi-daemon.socket
[  OK  ] Listening on BMC Webserver socket.                         <-- bmcweb.socket
[  OK  ] Listening on D-Bus System Message Bus Socket.              <-- dbus.socket
[  OK  ] Listening on dropbear.socket.                              <-- dropbear.socket
[  OK  ] Listening on MCTP demux Unix domain socket.                <-- mctp-demux.socket
[  OK  ] Listening on Phosphor Host…ole SSH Per-Connection socket.  <-- obmc-console-ssh.socket
[  OK  ] Listening on phosphor-ipmi-net@eth0.socket.                <-- phosphor-ipmi-net@eth0.socket
[  OK  ] Listening on phosphor-ipmi-net@eth1.socket.                <-- phosphor-ipmi-net@eth1.socket
[  OK  ] Reached target Socket Units.                               <-- sockets.target
         Starting D-Bus System Message Bus...                       <-- dbus-broker.service
[  OK  ] Started D-Bus System Message Bus.                          <-- dbus-broker.service
[  OK  ] Found device /dev/ttyS2.                                   <-- dev-ttyS2.device
[  OK  ] Found device /sys/subsystem/net/devices/eth0.              <-- sys-subsystem-net-devices-eth0.device
[  OK  ] Found device /sys/subsystem/net/devices/eth1.              <-- sys-subsystem-net-devices-eth1.device
[  *** ] A start job is running for Wait for… to be Configured (20s / no limit)
[  OK  ] Finished Wait for Network to be Configured.                <-- systemd-networkd-wait-online.service
[  OK  ] Created slice Slice /system/xyz.openbmc_project.Hwmon.     <-- system-xyz.openbmc_project.Hwmon.slice
[  OK  ] Created slice Slice /syste…penbmc_project.led.controller.  <-- system-xyz.openbmc_project.led.controller.slice
[  OK  ] Reached target Basic System.                               <-- basic.target
[  OK  ] Reached target Network is Online.                          <-- network-online.target
[  OK  ] Reached target Hardware activated USB gadget.              <-- usb-gadget.target
         Starting Avahi mDNS/DNS-SD Stack...                        <-- avahi-daemon.service
[  OK  ] Started Start bmcweb server.                               <-- bmcweb.service
         Starting Clear one time boot overrides...                  <-- clear-once.service
         Starting Convert PAM config files...                       <-- convert-pam-configs.service
         Starting SSH Key Generation...                             <-- dropbearkey.service
[  OK  ] Started LPC Snoop Daemon.                                  <-- lpcsnoop.service
         Starting Name Service Cache Daemon...                      <-- nscd.service
         Starting LDAP daemon...                                    <-- nslcd.service
         Starting Phosphor Console …tening on device /dev/ttyS2...  <-- obmc-console@ttyS2.service
[  OK  ] Started Phosphor Dump core monitor..                       <-- obmc-dump-monitor.service
         Starting Phosphor IPMI BT DBus Bridge...                   <-- org.openbmc.HostIpmi.service
[  OK  ] Started Phosphor certificate manager for authority.        <-- phosphor-certificate-manager@authority.service
[  OK  ] Started Phosphor certificate manager for bmcweb.           <-- phosphor-certificate-manager@bmcweb.service
[  OK  ] Started Phosphor certificate manager for nslcd.            <-- phosphor-certificate-manager@nslcd.service
         Starting BMC health monitoring...                          <-- phosphor-health-monitor.service
         Starting System Logging Service...                         <-- rsyslog.service
[  OK  ] Started Lightweight SLP Server.                            <-- slpd-lite.service
         Starting OpenBMC ipKVM daemon...                           <-- start-ipkvm.service
         Starting Hostname Service...                               <-- systemd-hostnamed.service
         Starting Permit User Sessions...                           <-- systemd-user-sessions.service
         Starting Enable Linux trace events in the boot loader...   <-- trace-enable.service
         Starting Intel Power Control for the Host 0...             <-- xyz.openbmc_project.Chassis.Control.Power@0.service
         Starting Phosphor Dump Manager...                          <-- xyz.openbmc_project.Dump.Manager.service
         Starting Entity Manager...                                 <-- xyz.openbmc_project.EntityManager.service
[  OK  ] Started Fru Device.                                        <-- xyz.openbmc_project.FruDevice.service
         Starting Phosphor Inventory Manager...                     <-- xyz.openbmc_project.Inventory.Manager.service
         Starting Phosphor LED Group Management Daemon...           <-- xyz.openbmc_project.LED.GroupManager.service
[  OK  ] Started IPMI SEL Logging Service.                          <-- xyz.openbmc_project.Logging.IPMI.service
         Starting Phosphor Log Manager...                           <-- xyz.openbmc_project.Logging.service
         Starting Phosphor Network Manager...                       <-- xyz.openbmc_project.Network.service
         Starting Phosphor DBus Service Discovery Manager...        <-- xyz.openbmc_project.ObjectMapper.service
         Starting Phosphor Settings Daemon...                       <-- xyz.openbmc_project.Settings.service
         Starting OpenBMC Software Update Manager...                <-- xyz.openbmc_project.Software.BMC.Updater.service
         Starting Phosphor Download Manager...                      <-- xyz.openbmc_project.Software.Download.service
         Starting Phosphor BMC State Manager...                     <-- xyz.openbmc_project.State.BMC.service
         Starting Post code manager...                              <-- xyz.openbmc_project.State.Boot.PostCode@0.service
         Starting Rsyslog config updater...                         <-- xyz.openbmc_project.Syslog.Config.service
         Starting Telemetry...                                      <-- xyz.openbmc_project.Telemetry.service
         Starting Phosphor User Manager...                          <-- xyz.openbmc_project.User.Manager.service
[  OK  ] Started Phosphor sysfs LED controller.                     <-- xyz.openbmc_project.led.controller@sys-class-leds-mmc0::.service
[  OK  ] Started Phosphor sysfs LED controller.                     <-- xyz.openbmc_project.led.controller@sys-class-leds-mmc1::.service
[  OK  ] Started Phosphor sysfs LED controller.                     <-- xyz.openbmc_project.led.controller@sys-class-leds-mmc2::.service
         Starting Rotate log files...                               <-- logrotate.service
[  OK  ] Started System Logging Service.                            <-- rsyslog.service
[  OK  ] Finished Convert PAM config files.                         <-- convert-pam-configs.service
[  OK  ] Started Name Service Cache Daemon.                         <-- nscd.service
[  OK  ] Started OpenBMC ipKVM daemon.                              <-- start-ipkvm.service
[  OK  ] Finished Permit User Sessions.                             <-- systemd-user-sessions.service
[  OK  ] Finished Enable Linux trace events in the boot loader.     <-- trace-enable.service
[FAILED] Failed to start Intel Power Control for the Host 0.        <-- xyz.openbmc_project.Chassis.Control.Power@0.service
See 'systemctl status xyz.openbmc_proje…s.Control.Power@0.service' for details.
[  OK  ] Finished Clear one time boot overrides.                    <-- clear-once.service
[  OK  ] Started Phosphor Console M…istening on device /dev/ttyS2.  <-- obmc-console@ttyS2.service
[  OK  ] Started Phosphor IPMI BT DBus Bridge.                      <-- org.openbmc.HostIpmi.service
[  OK  ] Started BMC health monitoring.                             <-- phosphor-health-monitor.service
[  OK  ] Started LDAP daemon.                                       <-- nslcd.service
[  OK  ] Started Phosphor Log Manager.                              <-- xyz.openbmc_project.Logging.service
[  OK  ] Started Phosphor Download Manager.                         <-- xyz.openbmc_project.Software.Download.service
[  OK  ] Started Phosphor Settings Daemon.                          <-- xyz.openbmc_project.Settings.service
[  OK  ] Started Phosphor LED Group Management Daemon.              <-- xyz.openbmc_project.LED.GroupManager.service
[  OK  ] Started Phosphor DBus Service Discovery Manager.           <-- xyz.openbmc_project.ObjectMapper.service
[  OK  ] Started Phosphor Inventory Manager.                        <-- xyz.openbmc_project.Inventory.Manager.service
[  OK  ] Started Rsyslog config updater.                            <-- xyz.openbmc_project.Syslog.Config.service
[  OK  ] Started Post code manager.                                 <-- xyz.openbmc_project.State.Boot.PostCode@0.service
[  OK  ] Started Telemetry.                                         <-- xyz.openbmc_project.Telemetry.service
[  OK  ] Started OpenBMC Software Update Manager.                   <-- xyz.openbmc_project.Software.BMC.Updater.service
[  OK  ] Started Hostname Service.                                  <-- systemd-hostnamed.service
[  OK  ] Started Phosphor Network Manager.                          <-- xyz.openbmc_project.Network.service
[  OK  ] Started Phosphor Dump Manager.                             <-- xyz.openbmc_project.Dump.Manager.service
[  OK  ] Started Entity Manager.                                    <-- xyz.openbmc_project.EntityManager.service
[  OK  ] Started Avahi mDNS/DNS-SD Stack.                           <-- avahi-daemon.service
[  OK  ] Started Getty on tty1.                                     <-- getty@tty1.service
[  OK  ] Started Host logger for ttyS2.                             <-- hostlogger@ttyS2.service
         Starting Wait for /xyz/ope…control/host0/boot/one_time...  <-- mapper-wait@-xyz-openbmc_project-control-host0-boot-one_time.service
         Starting Wait for /xyz/ope…_project/control/host0/boot...  <-- mapper-wait@-xyz-openbmc_project-control-host0-boot.service
         Starting Wait for /xyz/ope…/host0/power_restore_policy...  <-- mapper-wait@-xyz-openbmc_project-control-host0-power_restore_policy.service
         Starting Wait for /xyz/ope…trol/host0/restriction_mode...  <-- mapper-wait@-xyz-openbmc_project-control-host0-restriction_mode.service
         Starting Wait for /xyz/ope…oject/led/groups/bmc_booted...  <-- mapper-wait@-xyz-openbmc_project-led-groups-bmc_booted.service
         Starting Wait for /xyz/openbmc_project/led/groups...       <-- mapper-wait@-xyz-openbmc_project-led-groups.service
         Starting Wait for /xyz/ope…mc_project/time/sync_method...  <-- mapper-wait@-xyz-openbmc_project-time-sync_method.service
[  OK  ] Started phosphor systemd target monitor.                   <-- phosphor-systemd-target-monitor.service
[  OK  ] Started Serial Getty on ttyS4.                             <-- serial-getty@ttyS4.service
[  OK  ] Reached target Login Prompts.                              <-- getty.target
         Starting Phosphor Ldap config updater...                   <-- xyz.openbmc_project.Ldap.Config.service
         Starting Phosphor Version Software Manager...              <-- xyz.openbmc_project.Software.Version.service
[  OK  ] Started Adc Sensor.                                        <-- xyz.openbmc_project.adcsensor.service
[  OK  ] Started Exit Air Temp Sensor.                              <-- xyz.openbmc_project.exitairsensor.service
[  OK  ] Started External Sensor.                                   <-- xyz.openbmc_project.externalsensor.service
[  OK  ] Started Fan Sensor.                                        <-- xyz.openbmc_project.fansensor.service
[  OK  ] Started Phosphor-Pid-Contr…rgin-based Fan Control Daemon.  <-- phosphor-pid-control.service
[  OK  ] Started Hwmon Temp Sensor.                                 <-- xyz.openbmc_project.hwmontempsensor.service
[  OK  ] Started Intel CPU Sensor.                                  <-- xyz.openbmc_project.intelcpusensor.service
[  OK  ] Started Intrusion Sensor.                                  <-- xyz.openbmc_project.intrusionsensor.service
[  OK  ] Started IPMB Sensor.                                       <-- xyz.openbmc_project.ipmbsensor.service
[  OK  ] Started MCU Temp Sensor.                                   <-- xyz.openbmc_project.mcutempsensor.service
[  OK  ] Started PSU Sensor.                                        <-- xyz.openbmc_project.psusensor.service
[  OK  ] Finished Wait for /xyz/ope…t/control/host0/boot/one_time.  <-- mapper-wait@-xyz-openbmc_project-control-host0-boot-one_time.service
[  OK  ] Finished Wait for /xyz/ope…mc_project/control/host0/boot.  <-- mapper-wait@-xyz-openbmc_project-control-host0-boot.service
[  OK  ] Finished Wait for /xyz/ope…ol/host0/power_restore_policy.  <-- mapper-wait@-xyz-openbmc_project-control-host0-power_restore_policy.service
[  OK  ] Finished Wait for /xyz/ope…ontrol/host0/restriction_mode.  <-- mapper-wait@-xyz-openbmc_project-control-host0-restriction_mode.service
[  OK  ] Finished Wait for /xyz/ope…project/led/groups/bmc_booted.  <-- mapper-wait@-xyz-openbmc_project-led-groups-bmc_booted.service
[  OK  ] Finished Wait for /xyz/openbmc_project/led/groups.         <-- mapper-wait@-xyz-openbmc_project-led-groups.service
[  OK  ] Finished Wait for /xyz/openbmc_project/time/sync_method.   <-- mapper-wait@-xyz-openbmc_project-time-sync_method.service

Phosphor OpenBMC (Phosphor OpenBMC Project Reference Distro) nodistro.0 ast2600-default ttyS4

         Started Phosphor Version Software Manager.                 <-- xyz.openbmc_project.Software.Version.service
         Started Phosphor Ldap config updater.                      <-- xyz.openbmc_project.Ldap.Config.service
         Stopping LDAP daemon...                                    <-- nslcd.service
         Started FRU Fault monitor service.                         <-- obmc-fru-fault-monitor.service
         Starting Phosphor Inband IPMI...                           <-- phosphor-ipmi-host.service
         Stopped Intel Power Control for the Host 0.                <-- xyz.openbmc_project.Chassis.Control.Power@0.service
         Starting Intel Power Control for the Host 0...             <-- xyz.openbmc_project.Chassis.Control.Power@0.service
         Starting Phosphor Time Manager daemon...                   <-- xyz.openbmc_project.Time.Manager.service
         Stopped LDAP daemon.                                       <-- nslcd.service
         Started Phosphor BMC State Manager.                        <-- xyz.openbmc_project.State.BMC.service
         Started Phosphor User Manager.                             <-- xyz.openbmc_project.User.Manager.service
         Started Intel Power Control for the Host 0.                <-- xyz.openbmc_project.Chassis.Control.Power@0.service
         Starting Time & Date Service...                            <-- systemd-timedated.service
         Started Phosphor Inband IPMI.                              <-- phosphor-ipmi-host.service
         Started IPMB bridge.                                       <-- ipmb.service
         Starting Phosphor IPMI KCS DBus Bridge...                  <-- phosphor-ipmi-kcs@ipmi-kcs1.service
         Starting Phosphor IPMI KCS DBus Bridge...                  <-- phosphor-ipmi-kcs@ipmi-kcs2.service
         Starting Phosphor IPMI KCS DBus Bridge...                  <-- phosphor-ipmi-kcs@ipmi-kcs3.service
         Starting Phosphor IPMI KCS DBus Bridge...                  <-- phosphor-ipmi-kcs@ipmi-kcs4.service
         Starting Phosphor IPMI KCS DBus Bridge...                  <-- phosphor-ipmi-kcs@ipmi-kcs5.service
         Starting Phosphor IPMI KCS DBus Bridge...                  <-- phosphor-ipmi-kcs@ipmi-kcs6.service
         Starting Phosphor IPMI KCS DBus Bridge...                  <-- phosphor-ipmi-kcs@ipmi-kcs7.service
         Starting Phosphor IPMI KCS DBus Bridge...                  <-- phosphor-ipmi-kcs@ipmi-kcs8.service
         Finished SSH Key Generation.                               <-- dropbearkey.service
         Started Time & Date Service.                               <-- systemd-timedated.service
         Started Phosphor Time Manager daemon.                      <-- xyz.openbmc_project.Time.Manager.service
         Reached target Multi-User System.                          <-- multi-user.target
         Starting Assert bmc_booted LED...                          <-- obmc-led-group-start@bmc_booted.service
         Finished Assert bmc_booted LED.                            <-- obmc-led-group-start@bmc_booted.service
         Finished Rotate log files.                                 <-- logrotate.service
         Startup finished in 5.457s (kernel) + 49.357s (userspace) = 54.815s.
         Stopped Host logger for ttyS2.                             <-- hostlogger@ttyS2.service
```

```
call mount_enumerate_perpetual()
    push -.mount

call slice_enumerate_perpetual()
    push -.slice
    push system.slice

call scope_enumerate_perpetual()
    push init.scope

call mount_enumerate()
    push dev-mtdblock4.device
    push blockdev@dev-mtdblock4.target

pop push blockdev@dev-mtdblock4.target
    open /usr/lib/systemd/system/blockdev@.target
    push shutdown.target

pop shutdown.target
    open /usr/lib/systemd/system/shutdown.target

pop dev-mtdblock4.device
    open null

pop init.scope
    open null

pop system.slice
    open null

pop -.slice
    open null

pop -.mount
    open /run/systemd/generator/-.mount
    push local-fs.target
    push blockdev@dev-root.target
    push dev-root.mount
    push dev.mount

pop dev.mount
    open null

pop dev-root.mount
    open null

pop blockdev@dev-root.target
    open /usr/lib/systemd/system/blockdev@.target

pop local-fs.target
    open /usr/lib/systemd/system/local-fs.target
    push local-fs-pre.target
    push emergency.target
    push systemd-fsck-root.service
    push systemd-remount-fs.service
    push tmp.mount
    push var-volatile-cache.service
    push var-volatile-lib.service
    push var-volatile-spool.service
    push var-volatile-srv.service
    push var-volatile.mount

pop var-volatile.mount
    open /run/systemd/generator/var-volatile.mount
    push systemd-journald.socket
    push umount.target
    push swap.target
    push var.mount

pop var.mount
    open null

pop swap.target
    open /usr/lib/systemd/system/swap.target

pop umount.target
    open /usr/lib/systemd/system/umount.target

pop systemd-journald.socket
    open /usr/lib/systemd/system/systemd-journald.socket
    push sockets.target
    push systemd-journald.service
    push run-systemd-journal-socket.mount
    push run-systemd-journal.mount
    push run-systemd.mount
    push run.mount
    push run-systemd-journal-stdout.mount

pop run-systemd-journal-stdout.mount
    open null

pop run.mount
    open null

pop run-systemd.mount
    open null

pop run-systemd-journal.mount
    open null

pop run-systemd-journal-socket.mount
    open null

pop systemd-journald.service
    open /usr/lib/systemd/system/systemd-journald.service
    push systemd-journald-dev-log.socket
    push systemd-journald-audit.socket
    push syslog.socket
    push sysinit.target

pop sysinit.target
    open /usr/lib/systemd/system/sysinit.target
    push emergency.service
    push dev-hugepages.mount
    push dev-mqueue.mount
    push run-postinsts.service
    push sys-fs-fuse-connections.mount
    push sys-kernel-config.mount
    push sys-kernel-debug.mount
    push sys-kernel-tracing.mount
    push systemd-ask-password-console.path
    push systemd-hwdb-update.service
    push systemd-journal-catalog-update.service
    push systemd-journal-flush.service
    push systemd-machine-id-commit.service
    push systemd-network-generator.service
    push systemd-pstore.service
    push systemd-random-seed.service
    push systemd-resolved.service
    push systemd-sysctl.service
    push systemd-sysusers.service
    push systemd-timesyncd.service
    push systemd-tmpfiles-setup-dev.service
    push systemd-tmpfiles-setup.service
    push systemd-udev-trigger.service
    push systemd-udevd.service
    push systemd-update-done.service
    push xyz.openbmc_project.Chassis.Control.Power@0.service

pop xyz.openbmc_project.Chassis.Control.Power@0.service
    open /usr/lib/systemd/system/xyz.openbmc_project.Chassis.Control.Power@.service
    push system-xyz.openbmc_project.Chassis.Control.Power.slice
    push basic.target
    push dbus.socket

pop dbus.socket
    open /usr/lib/systemd/system/dbus.socket
    push dbus.service
    push run-dbus-system_bus_socket.mount
    push run-dbus.mount

pop run-dbus.mount
    open null

pop run-dbus-system_bus_socket.mount
    open null

pop dbus.service
    open /usr/lib/systemd/system/dbus-broker.service
    (u->id becomes dbus-broker.service?)
    push var-tmp.mount

pop var-tmp.mount
    open null

pop basic.target
    open /usr/lib/systemd/system/basic.target
    push timers.target
    push paths.target
    push slices.target
    push ipmb.service

pop ipmb.service
    open /usr/lib/systemd/system/ipmb.service
    add phosphor-ipmi-host.service

pop phosphor-ipmi-host.service
    open /usr/lib/systemd/system/phosphor-ipmi-host.service
    add clear-once.service
    add xyz.openbmc_project.Settings.service
    add org.openbmc.HostIpmi.service
    add mapper-wait@-xyz-openbmc_project-control-host0-boot.service
    add mapper-wait@-xyz-openbmc_project-control-host0-boot-one_time.service
    add mapper-wait@-xyz-openbmc_project-control-host0-power_restore_policy.service
    add mapper-wait@-xyz-openbmc_project-control-host0-restriction_mode.service
    add var-lib-ipmi.mount
    add var-lib.mount
    add run-ipmi.mount

pop run-ipmi.mount
    open null

pop var-lib.mount
    open null

pop var-lib-ipmi.mount
    open null

pop mapper-wait@-xyz-openbmc_project-control-host0-restriction_mode.service
    open /usr/lib/systemd/system/mapper-wait@.service
    push xyz.openbmc_project.ObjectMapper.service
    push system-mapper\x2dwait.slice

pop system-mapper\x2dwait.slice
    open null

pop xyz.openbmc_project.ObjectMapper.service
    open /usr/lib/systemd/system/xyz.openbmc_project.ObjectMapper.service

pop mapper-wait@-xyz-openbmc_project-control-host0-power_restore_policy.service
    open /usr/lib/systemd/system/mapper-wait@.service

pop mapper-wait@-xyz-openbmc_project-control-host0-boot-one_time.service
    open /usr/lib/systemd/system/mapper-wait@.service

pop mapper-wait@-xyz-openbmc_project-control-host0-boot.service
    open /usr/lib/systemd/system/mapper-wait@.service

pop org.openbmc.HostIpmi.service
    open /usr/lib/systemd/system/org.openbmc.HostIpmi.service

pop xyz.openbmc_project.Settings.service
    open /usr/lib/systemd/system/xyz.openbmc_project.Settings.service

pop clear-once.service
    open /usr/lib/systemd/system/clear-once.service
    push sbin.mount
    push etc.mount

pop etc.mount
    open null

pop sbin.mount
    open null

pop slices.target
    open /usr/lib/systemd/system/slices.target

pop paths.target
    open /usr/lib/systemd/system/paths.target

pop timers.target
    open /usr/lib/systemd/system/timers.target
    push logrotate.timer
    push systemd-tmpfiles-clean.timer

pop systemd-tmpfiles-clean.timer
    open /usr/lib/systemd/system/systemd-tmpfiles-clean.timer
    push systemd-tmpfiles-clean.service

pop systemd-tmpfiles-clean.service
    open /usr/lib/systemd/system/systemd-tmpfiles-clean.service
    push time-set.target
    push initrd-switch-root.target

pop initrd-switch-root.target
    open /usr/lib/systemd/system/initrd-switch-root.target

pop time-set.target
    open /usr/lib/systemd/system/time-set.target

pop logrotate.timer
    open /usr/lib/systemd/system/logrotate.timer
    push logrotate.service
    push time-sync.target
    push var-lib-systemd-timers.mount
    push var-lib-systemd.mount

pop var-lib-systemd.mount
    open null

pop var-lib-systemd-timers.mount
    open null

pop time-sync.target
    open /usr/lib/systemd/system/time-sync.target

pop logrotate.service
    open /usr/lib/systemd/system/logrotate.service
    push var-log.mount

pop var-log.mount
    open null

pop system-xyz.openbmc_project.Chassis.Control.Power.slice
    open null

pop systemd-update-done.service
    open /usr/lib/systemd/system/systemd-update-done.service

pop systemd-udevd.service
    open /usr/lib/systemd/system/systemd-udevd.service

pop systemd-udev-trigger.service
    open /usr/lib/systemd/system/systemd-udev-trigger.service

pop systemd-tmpfiles-setup.service
    open systemd-tmpfiles-setup.service

pop systemd-tmpfiles-setup-dev.service
    open /usr/lib/systemd/system/systemd-tmpfiles-setup-dev.service

pop systemd-timesyncd.service
    open /usr/lib/systemd/system/systemd-timesyncd.service
    push var-lib-systemd-timesync.mount
    push run-systemd-timesync.mount

pop run-systemd-timesync.mount
    open null

pop var-lib-systemd-timesync.mount
    open null

pop systemd-sysusers.service
    open /usr/lib/systemd/system/systemd-sysusers.service

pop systemd-sysctl.service
    open /usr/lib/systemd/system/systemd-sysctl.service
    push systemd-modules-load.service

pop systemd-modules-load.service
    open null

pop systemd-resolved.service
    open /usr/lib/systemd/system/systemd-resolved.service
    push network.target
    push nss-lookup.target
    push run-systemd-resolve.mount

pop run-systemd-resolve.mount
    open null

pop nss-lookup.target
    open /usr/lib/systemd/system/nss-lookup.target

pop network.target
    open /usr/lib/systemd/system/network.target
    push network-pre.target
    push bmcweb.service

pop bmcweb.service
    open /usr/lib/systemd/system/bmcweb.service
    push home-root.mount
    push home.mount

pop home.mount
    open null

pop home-root.mount
    open null

pop network-pre.target
    open /usr/lib/systemd/system/network-pre.target

pop systemd-random-seed.service
    open /usr/lib/systemd/system/systemd-random-seed.service
    push first-boot-complete.target
    push var-lib-systemd-random\x2dseed.mount

pop var-lib-systemd-random\x2dseed.mount
    open null

pop first-boot-complete.target
    open /usr/lib/systemd/system/first-boot-complete.target

pop systemd-pstore.service
    open /usr/lib/systemd/system/systemd-pstore.service
    push modprobe@efi_pstore.service
    push var-lib-systemd-pstore.mount

pop var-lib-systemd-pstore.mount
    open null

pop modprobe@efi_pstore.service
    open /usr/lib/systemd/system/modprobe@.service
    push system-modprobe.slice

pop system-modprobe.slice
    open null

pop systemd-network-generator.service
    open /usr/lib/systemd/system/systemd-network-generator.service

pop systemd-machine-id-commit.service
    open /usr/lib/systemd/system/systemd-machine-id-commit.service
    push systemd-machine-id-commit.service

pop systemd-journal-flush.service
    open /usr/lib/systemd/system/systemd-journal-flush.service
    push var-log-journal.mount

pop var-log-journal.mount
    open null

pop systemd-journal-catalog-update.service
    open /usr/lib/systemd/system/systemd-journal-catalog-update.service

pop systemd-hwdb-update.service
    open /usr/lib/systemd/system/systemd-hwdb-update.service

pop systemd-ask-password-console.path
    open /usr/lib/systemd/system/systemd-ask-password-console.path
    push cryptsetup.target
    push systemd-ask-password-console.service
    push run-systemd-ask\x2dpassword.mount

pop run-systemd-ask\x2dpassword.mount
    open null

pop systemd-ask-password-console.service
    open /usr/lib/systemd/system/systemd-ask-password-console.service
    push systemd-vconsole-setup.service

pop systemd-vconsole-setup.service
    open null

pop cryptsetup.target
    open null

pop sys-kernel-tracing.mount
    open /usr/lib/systemd/system/sys-kernel-tracing.mount
    push sys-kernel.mount
    push sys.mount

pop sys.mount
    open null

pop sys-kernel.mount
    open null

pop sys-kernel-debug.mount
    open /usr/lib/systemd/system/sys-kernel-debug.mount

pop sys-kernel-config.mount
    open /usr/lib/systemd/system/sys-kernel-config.mount
    push modprobe@configfs.service

pop modprobe@configfs.service
    open /usr/lib/systemd/system/modprobe@.service

pop sys-fs-fuse-connections.mount
    open /usr/lib/systemd/system/sys-fs-fuse-connections.mount
    push modprobe@fuse.service
    push sys-fs-fuse.mount
    push sys-fs.mount

pop sys-fs.mount
    open null

pop sys-fs-fuse.mount
    open null

pop modprobe@fuse.service
    open /usr/lib/systemd/system/modprobe@.service

pop run-postinsts.service
    open /usr/lib/systemd/system/run-postinsts.service
    push ldconfig.service

pop ldconfig.service
    open null

pop dev-mqueue.mount
    open /usr/lib/systemd/system/dev-mqueue.mount

pop dev-hugepages.mount
    open /usr/lib/systemd/system/dev-hugepages.mount

pop emergency.service
    open /usr/lib/systemd/system/emergency.service

pop syslog.socket
    open /usr/lib/systemd/system/syslog.socket
    push syslog.service
    push run-systemd-journal-syslog.mount

pop run-systemd-journal-syslog.mount
    open null

pop syslog.service
    open /usr/lib/systemd/system/rsyslog.service
    push network-online.target

pop network-online.target
    open /usr/lib/systemd/system/network-online.target
    push systemd-networkd-wait-online.service

pop systemd-networkd-wait-online.service
    open /usr/lib/systemd/system/systemd-networkd-wait-online.service
    push systemd-networkd.service

pop systemd-networkd.service
    open /usr/lib/systemd/system/systemd-networkd.service
    push systemd-networkd.socket
    push multi-user.target
    push run-systemd-netif.mount

pop run-systemd-netif.mount
    open null

pop multi-user.target
    open /usr/lib/systemd/system/multi-user.target
    push avahi-daemon.service
    push convert-pam-configs.service
    push dropbearkey.service
    push getty.target
    push hostlogger@ttyS2.service
    push lpcsnoop.service
    push nfs-statd.service
    push nscd.service
    push nslcd.service
    push obmc-console@ttyS2.service
    push obmc-dump-monitor.service
    push obmc-fru-fault-monitor.service
    push obmc-led-group-start@bmc_booted.service
    push phosphor-certificate-manager@authority.service
    push phosphor-certificate-manager@bmcweb.service
    push phosphor-certificate-manager@nslcd.service
    push phosphor-health-monitor.service
    push phosphor-ipmi-kcs@ipmi-kcs1.service
    push phosphor-ipmi-kcs@ipmi-kcs2.service
    push phosphor-ipmi-kcs@ipmi-kcs3.service
    push phosphor-ipmi-kcs@ipmi-kcs4.service
    push phosphor-ipmi-kcs@ipmi-kcs5.service
    push phosphor-ipmi-kcs@ipmi-kcs6.service
    push phosphor-ipmi-kcs@ipmi-kcs7.service
    push phosphor-ipmi-kcs@ipmi-kcs8.service
    push phosphor-ipmi-net@eth0.service
    push phosphor-ipmi-net@eth1.service
    push phosphor-pid-control.service
    push phosphor-systemd-target-monitor.service
    push ramoops-monitor.service
    push remote-fs.target
    push rpcbind.service
    push slpd-lite.service
    push ssifbridge.service
    push start-ipkvm.service
    push systemd-ask-password-wall.path
    push systemd-user-sessions.service
    push trace-enable.service
    push xyz.openbmc_project.Dump.Manager.service
    push xyz.openbmc_project.EntityManager.service
    push xyz.openbmc_project.FruDevice.service
    push xyz.openbmc_project.Inventory.Manager.service
    push xyz.openbmc_project.LED.GroupManager.service
    push xyz.openbmc_project.Ldap.Config.service
    push xyz.openbmc_project.Logging.IPMI.service
    push xyz.openbmc_project.Logging.service
    push xyz.openbmc_project.Network.service
    push xyz.openbmc_project.Software.BMC.Updater.service
    push xyz.openbmc_project.Software.Download.service
    push xyz.openbmc_project.Software.Version.service
    push xyz.openbmc_project.State.BMC.service
    push xyz.openbmc_project.State.Boot.PostCode
    push xyz.openbmc_project.Syslog.Config.service
    push xyz.openbmc_project.Telemetry.service
    push xyz.openbmc_project.Time.Manager.service
    push xyz.openbmc_project.User.Manager.service
    push xyz.openbmc_project.adcsensor.service
    push xyz.openbmc_project.exitairsensor.service
    push xyz.openbmc_project.externalsensor.service
    push xyz.openbmc_project.fansensor.service
    push xyz.openbmc_project.hwmontempsensor.service
    push xyz.openbmc_project.intelcpusensor.service
    push xyz.openbmc_project.intrusionsensor.service
    push xyz.openbmc_project.ipmbsensor.service
    push xyz.openbmc_project.mcutempsensor.service
    push xyz.openbmc_project.psusensor.service

pop xyz.openbmc_project.psusensor.service
    open /usr/lib/systemd/system/xyz.openbmc_project.psusensor.service

pop xyz.openbmc_project.mcutempsensor.service
    open /usr/lib/systemd/system/xyz.openbmc_project.mcutempsensor.service

pop xyz.openbmc_project.ipmbsensor.service
    open /usr/lib/systemd/system/xyz.openbmc_project.ipmbsensor.service

pop xyz.openbmc_project.intrusionsensor.service
    open /usr/lib/systemd/system/xyz.openbmc_project.intrusionsensor.service

pop xyz.openbmc_project.intelcpusensor.service
    open /usr/lib/systemd/system/xyz.openbmc_project.intelcpusensor.service

pop xyz.openbmc_project.hwmontempsensor.service
    open /usr/lib/systemd/system/xyz.openbmc_project.hwmontempsensor.service

pop xyz.openbmc_project.fansensor.service
    open /usr/lib/systemd/system/xyz.openbmc_project.fansensor.service

pop xyz.openbmc_project.externalsensor.service
    open /usr/lib/systemd/system/xyz.openbmc_project.externalsensor.service

pop xyz.openbmc_project.exitairsensor.service
    open /usr/lib/systemd/system/xyz.openbmc_project.exitairsensor.service

pop xyz.openbmc_project.adcsensor.service
    open /usr/lib/systemd/system/xyz.openbmc_project.adcsensor.service

pop xyz.openbmc_project.User.Manager.service
    open /usr/lib/systemd/system/xyz.openbmc_project.User.Manager.service

pop xyz.openbmc_project.Time.Manager.service
    open /usr/lib/systemd/system/xyz.openbmc_project.Time.Manager.service
    push mapper-wait@-xyz-openbmc_project-time-sync_method.service

pop mapper-wait@-xyz-openbmc_project-time-sync_method.service
    open /usr/lib/systemd/system/mapper-wait@.service

pop xyz.openbmc_project.Telemetry.service
    open /usr/lib/systemd/system/xyz.openbmc_project.Telemetry.service

pop xyz.openbmc_project.Syslog.Config.service
    open /usr/lib/systemd/system/xyz.openbmc_project.Syslog.Config.service

pop xyz.openbmc_project.State.Boot.PostCode@0.service
    open /usr/lib/systemd/system/xyz.openbmc_project.State.Boot.PostCode@.service
    push system-xyz.openbmc_project.State.Boot.PostCode.slice

pop system-xyz.openbmc_project.State.Boot.PostCode.slice
    open null

pop xyz.openbmc_project.State.BMC.service
    open /usr/lib/systemd/system/xyz.openbmc_project.State.BMC.service
    push mapper-wait@-xyz-openbmc_project-state-bmc.service
    push xyz.openbmc_project.State.Chassis@0.service

pop xyz.openbmc_project.State.Chassis@0.service
    open null

pop mapper-wait@-xyz-openbmc_project-state-bmc.service
    open /usr/lib/systemd/system/mapper-wait@.service

pop xyz.openbmc_project.Software.Version.service
    open /usr/lib/systemd/system/xyz.openbmc_project.Software.Version.service

pop xyz.openbmc_project.Software.Download.service
    open /usr/lib/systemd/system/xyz.openbmc_project.Software.Download.service

pop xyz.openbmc_project.Software.BMC.Updater.service
    open /usr/lib/systemd/system/xyz.openbmc_project.Software.BMC.Updater.service

pop xyz.openbmc_project.Network.service
    open /usr/lib/systemd/system/xyz.openbmc_project.Network.service
    push run-network.mount
    push var-lib-network.mount

pop var-lib-network.mount
    open null

pop run-network.mount
    open null

pop xyz.openbmc_project.Logging.service
    open /usr/lib/systemd/system/xyz.openbmc_project.Logging.service

pop xyz.openbmc_project.Logging.IPMI.service
    open /usr/lib/systemd/system/xyz.openbmc_project.Logging.IPMI.service

pop xyz.openbmc_project.Ldap.Config.service
    open /usr/lib/systemd/system/xyz.openbmc_project.Ldap.Config.service
    push xyz.openbmc_project.Software.Sync.service

pop xyz.openbmc_project.Software.Sync.service
    open null

pop xyz.openbmc_project.LED.GroupManager.service
    open /usr/lib/systemd/system/xyz.openbmc_project.LED.GroupManager.service
    push mapper-wait@-xyz-openbmc_project-led-groups.service

pop mapper-wait@-xyz-openbmc_project-led-groups.service
    open /usr/lib/systemd/system/mapper-wait@.service

pop xyz.openbmc_project.Inventory.Manager.service
    open /usr/lib/systemd/system/xyz.openbmc_project.Inventory.Manager.service
    push mapper-wait@-xyz-openbmc_project-inventory.service

pop mapper-wait@-xyz-openbmc_project-inventory.service
    open /usr/lib/systemd/system/mapper-wait@.service

pop xyz.openbmc_project.FruDevice.service
    open /usr/lib/systemd/system/xyz.openbmc_project.FruDevice.service

pop xyz.openbmc_project.EntityManager.service
    open /usr/lib/systemd/system/xyz.openbmc_project.EntityManager.service

pop xyz.openbmc_project.Dump.Manager.service
    open /usr/lib/systemd/system/xyz.openbmc_project.Dump.Manager.service

pop trace-enable.service
    open /usr/lib/systemd/system/trace-enable.service

pop systemd-user-sessions.service
    open /usr/lib/systemd/system/systemd-user-sessions.service
    push nss-user-lookup.target

pop nss-user-lookup.target
    open /usr/lib/systemd/system/nss-user-lookup.target

pop systemd-ask-password-wall.path
    open /usr/lib/systemd/system/systemd-ask-password-wall.path
    push systemd-ask-password-wall.service

pop systemd-ask-password-wall.service
    open /usr/lib/systemd/system/systemd-ask-password-wall.service

pop start-ipkvm.service
    open /usr/lib/systemd/system/start-ipkvm.service

pop ssifbridge.service
    open /usr/lib/systemd/system/ssifbridge.service

pop slpd-lite.service
    open /usr/lib/systemd/system/slpd-lite.service

pop rpcbind.service
    open /usr/lib/systemd/system/rpcbind.service
    push rpcbind.socket
    push rpcbind.target
    push var-run-rpcbind.mount
    push var-run.mount

pop var-run.mount
    open null

pop var-run-rpcbind.mount
    open null

pop rpcbind.target
    open /usr/lib/systemd/system/rpcbind.target

pop rpcbind.socket
    open /usr/lib/systemd/system/rpcbind.socket
    push run-rpcbind.sock.mount

pop run-rpcbind.sock.mount
    open null

pop remote-fs.target
    open /usr/lib/systemd/system/remote-fs.target
    push remote-fs-pre.target

pop remote-fs-pre.target
    open /usr/lib/systemd/system/remote-fs-pre.target

pop ramoops-monitor.service
    open /usr/lib/systemd/system/ramoops-monitor.service

pop phosphor-systemd-target-monitor.service
    open /usr/lib/systemd/system/phosphor-systemd-target-monitor.service

pop phosphor-pid-control.service
    open /usr/lib/systemd/system/phosphor-pid-control.service

pop phosphor-ipmi-net@eth1.service
    open /usr/lib/systemd/system/phosphor-ipmi-net@.service
    push sys-subsystem-net-devices-eth1.device
    push system-phosphor\x2dipmi\x2dnet.slice

pop system-phosphor\x2dipmi\x2dnet.slice
    open null

pop sys-subsystem-net-devices-eth1.device
    open null

pop phosphor-ipmi-net@eth0.service
    open /usr/lib/systemd/system/phosphor-ipmi-net@.service
    push sys-subsystem-net-devices-eth0.device

pop sys-subsystem-net-devices-eth0.device
    open null

pop phosphor-ipmi-kcs@ipmi-kcs8.service
    open /usr/lib/systemd/system/phosphor-ipmi-kcs@.service
    push system-phosphor\x2dipmi\x2dkcs.slice

pop system-phosphor\x2dipmi\x2dkcs.slice
    open null

pop phosphor-ipmi-kcs@ipmi-kcs7.service
    open /usr/lib/systemd/system/phosphor-ipmi-kcs@.service

pop phosphor-ipmi-kcs@ipmi-kcs6.service
    open /usr/lib/systemd/system/phosphor-ipmi-kcs@.service

pop phosphor-ipmi-kcs@ipmi-kcs5.service
    open /usr/lib/systemd/system/phosphor-ipmi-kcs@.service

pop phosphor-ipmi-kcs@ipmi-kcs4.service
    open /usr/lib/systemd/system/phosphor-ipmi-kcs@.service

pop phosphor-ipmi-kcs@ipmi-kcs3.service
    open /usr/lib/systemd/system/phosphor-ipmi-kcs@.service

pop phosphor-ipmi-kcs@ipmi-kcs2.service
    open /usr/lib/systemd/system/phosphor-ipmi-kcs@.service

pop phosphor-ipmi-kcs@ipmi-kcs1.service
    open /usr/lib/systemd/system/phosphor-ipmi-kcs@.service

pop phosphor-health-monitor.service
    open /usr/lib/systemd/system/phosphor-health-monitor.service

pop phosphor-certificate-manager@nslcd.service
    open /usr/lib/systemd/system/phosphor-certificate-manager@.service
    push system-phosphor\x2dcertificate\x2dmanager.slice

pop system-phosphor\x2dcertificate\x2dmanager.slice
    open null

pop phosphor-certificate-manager@bmcweb.service
    open /usr/lib/systemd/system/phosphor-certificate-manager@.service

pop phosphor-certificate-manager@authority.service
    open /usr/lib/systemd/system/phosphor-certificate-manager@.service

pop obmc-led-group-start@bmc_booted.service
    open /usr/lib/systemd/system/obmc-led-group-start@.service
    push mapper-wait@-xyz-openbmc_project-led-groups-bmc_booted.service
    push obmc-led-group-stop@bmc_booted.service
    push system-obmc\x2dled\x2dgroup\x2dstart.slice

pop system-obmc\x2dled\x2dgroup\x2dstart.slice
    open null

pop obmc-led-group-stop@bmc_booted.service
    open /usr/lib/systemd/system/obmc-led-group-stop@.service
    push system-obmc\x2dled\x2dgroup\x2dstop.slice

pop system-obmc\x2dled\x2dgroup\x2dstop.slice
    open null

pop mapper-wait@-xyz-openbmc_project-led-groups-bmc_booted.service
    open /usr/lib/systemd/system/mapper-wait@.service

pop obmc-fru-fault-monitor.service
    open /usr/lib/systemd/system/obmc-fru-fault-monitor.service

pop obmc-dump-monitor.service
    open /usr/lib/systemd/system/obmc-dump-monitor.service

pop obmc-console@ttyS2.service
    open /usr/lib/systemd/system/obmc-console@.service
    push dev-ttyS2.device
    push system-obmc\x2dconsole.slice

pop system-obmc\x2dconsole.slice
    open null

pop dev-ttyS2.device
    open null

pop nslcd.service
    open /usr/lib/systemd/system/nslcd.service
    push syslog.target

pop syslog.target
    open null

pop nscd.service
    open /usr/lib/systemd/system/nscd.service

pop nfs-statd.service
    open /usr/lib/systemd/system/nfs-statd.service

pop lpcsnoop.service
    open /usr/lib/systemd/system/lpcsnoop.service

pop hostlogger@ttyS2.service
    open /usr/lib/systemd/system/hostlogger@.service
    push system-hostlogger.slice

pop system-hostlogger.slice
    open null

pop getty.target
    open /usr/lib/systemd/system/getty.target
    push getty@tty1.service
    push serial-getty@ttyS4.service

pop serial-getty@ttyS4.service
    open /usr/lib/systemd/system/serial-getty@.service
    push dev-ttyS4.device
    push plymouth-quit-wait.service
    push getty-pre.target
    push rc-local.service
    push system-serial\x2dgetty.slice

pop system-serial\x2dgetty.slice
    open null

pop rc-local.service
    open null

pop getty-pre.target
    open /usr/lib/systemd/system/getty-pre.target

pop plymouth-quit-wait.service
    open null

pop dev-ttyS4.device
    open null

pop getty@tty1.service
    open /usr/lib/systemd/system/getty@.service
    push system-getty.slice

pop system-getty.slice
    open null

pop dropbearkey.service
    open /usr/lib/systemd/system/dropbearkey.service
    push dropbear-migrate-key-location.service

pop dropbear-migrate-key-location.service
    open /usr/lib/systemd/system/dropbear-migrate-key-location.service

pop convert-pam-configs.service
    open /usr/lib/systemd/system/convert-pam-configs.service

pop avahi-daemon.service
    open /usr/lib/systemd/system/avahi-daemon.service
    push avahi-daemon.socket
    push connman.service

pop connman.service
    open null

pop avahi-daemon.socket
    open /usr/lib/systemd/system/avahi-daemon.socket
    push run-avahi\x2ddaemon-socket.mount
    push run-avahi\x2ddaemon.mount

pop run-avahi\x2ddaemon.mount
    open null

pop run-avahi\x2ddaemon-socket.mount
    open null

pop systemd-networkd.socket
    open /usr/lib/systemd/system/systemd-networkd.socket

pop systemd-journald-audit.socket
    open /usr/lib/systemd/system/systemd-journald-audit.socket

pop systemd-journald-dev-log.socket
    open /usr/lib/systemd/system/systemd-journald-dev-log.socket
    push run-systemd-journal-dev\x2dlog.mount

pop run-systemd-journal-dev\x2dlog.mount
    open null

pop sockets.target
    open /usr/lib/systemd/system/sockets.target
    push bmcweb.socket
    push dropbear.socket
    push mctp-demux.socket
    push obmc-console-ssh.socket
    push phosphor-ipmi-net@eth0.socket
    push phosphor-ipmi-net@eth1.socket
    push systemd-coredump.socket

pop systemd-coredump.socket
    open /usr/lib/systemd/system/systemd-coredump.socket
    push run-systemd-coredump.mount

pop run-systemd-coredump.mount
    open null

pop phosphor-ipmi-net@eth1.socket
    open /usr/lib/systemd/system/phosphor-ipmi-net@.socket
    (Invalid interface name, ignoring: sys-subsystem-net-devices-%i.device)

pop phosphor-ipmi-net@eth0.socket
    open /usr/lib/systemd/system/phosphor-ipmi-net@.socket
    (Invalid interface name, ignoring: sys-subsystem-net-devices-%i.device)

pop obmc-console-ssh.socket
    open /usr/lib/systemd/system/obmc-console-ssh.socket

pop mctp-demux.socket
    open /usr/lib/systemd/system/mctp-demux.socket
    push mctp-demux.service

pop mctp-demux.service
    open /usr/lib/systemd/system/mctp-demux.service

pop dropbear.socket
    open /usr/lib/systemd/system/dropbear.socket
    push dropbear.service

pop dropbear.service
    open null

pop bmcweb.socket
    open /usr/lib/systemd/system/bmcweb.socket

pop var-volatile-srv.service
    open /usr/lib/systemd/system/var-volatile-srv.service

pop var-volatile-spool.service
    open /usr/lib/systemd/system/var-volatile-spool.service

pop var-volatile-lib.service
    open /usr/lib/systemd/system/var-volatile-lib.service

pop var-volatile-cache.service
    open /usr/lib/systemd/system/var-volatile-cache.service

pop tmp.mount
    open /usr/lib/systemd/system/tmp.mount

pop systemd-remount-fs.service
    open /usr/lib/systemd/system/systemd-remount-fs.service

pop systemd-fsck-root.service
    open /usr/lib/systemd/system/systemd-fsck-root.service

pop emergency.target
    open /usr/lib/systemd/system/emergency.target

pop local-fs-pre.target
    open /usr/lib/systemd/system/local-fs-pre.target
    push run-initramfs-ro.mount
    push dev-mtdblock5.device
    push blockdev@dev-mtdblock5.target

pop blockdev@dev-mtdblock5.target
    open /usr/lib/systemd/system/blockdev@.target

pop dev-mtdblock5.device
    open null

pop run-initramfs-ro.mount
    open null
    push run-initramfs.mount

pop run-initramfs.mount
    open null
    push run-initramfs-rw.mount

pop run-initramfs-rw.mount
    open null





"Listening on Syslog Socket."

???
    push systemd-coredump@internal.service

pop systemd-coredump@internal.service
    open /usr/lib/systemd/system/systemd-coredump@.service
    push system-systemd\x2dcoredump.slice
    push var-tmp.mount
    push var.mount
    push var-lib-systemd-coredump.mount
    push var-lib-systemd.mount
    push var-lib.mount

pop var-lib.mount
    open null

pop var-lib-systemd.mount
    open null

pop var-lib-systemd-coredump.mount
    open null

pop var.mount
    open null

pop var-tmp.mount
    open null

popo system-systemd\x2dcoredump.slice
    open null

???
    push run-credentials-systemd\x2dsysctl.service.mount

pop run-credentials-systemd\x2dsysctl.service.mount
    open null
    push run-credentials.mount
    push run.mount

pop run.mount
    open null

pop run-credentials.mount
    open null
```

```
src/libsystemd/sd-event/sd-event.c                                                   
+--------------+                                                                      
| sd_event_new | : prepare event and its priority queue, create epoll_fd, return event
+-|------------+                                                                      
  |                                                                                   
  |--> prepare 'sd_event'                                                             
  |                                                                                   
  |    +------------------------+                                                     
  |--> | prioq_ensure_allocated | alloc priority queue and install compare()          
  |    +------------------------+                                                     
  |    +---------------+                                                              
  |--> | epoll_create1 |                                                              
  |    +---------------+                                                              
  |    +---------------------+                                                        
  +--> | fd_move_above_stdio | ensure the epoll_fd isn't in range [0, 2]              
       +---------------------+                                                        
```

```
src/libsystemd/sd-event/sd-event.c                                                        
+------------------+                                                                       
| sd_event_default | : ensure we have a default event, return its address                  
+-|----------------+                                                                       
  |                                                                                        
  |--> if default_event (global) has value                                                 
  |    -                                                                                   
  |    +--> return its address                                                             
  |                                                                                        
  |    +--------------+                                                                    
  |--> | sd_event_new | prepare event and its priority queue, create epoll_fd, return event
  |    +--------------+                                                                    
  |                                                                                        
  +--> default_event = new event                                                           
```

```
src/libsystemd/sd-event/sd-event.c                                                        
+--------------+                                                                       
| sd_event_ref | : ref++
+-------------+                                                                       
```

```
src/libsystemd/sd-event/sd-event.c                                                        
+----------------+                                                                       
| sd_event_unref | : ref--, free event if no one uses
+----------------+                                                                       
```

```
src/libsystemd/sd-event/sd-event.c                                                   
+-----------------+                                                                   
| sd_event_add_io | : prepare 'source' and add to arg 'event, register the source's io
+-|---------------+                                                                   
  |                                                                                   
  |--> if arg callback isn't provided                                                 
  |    -                                                                              
  |    +--> use the default one                                                       
  |         +------------------+                                                      
  |         | io_exit_callback | set event's exit fields                              
  |         +------------------+                                                      
  |    +------------+                                                                 
  |--> | source_new | alloc and init 'event source', prepend to event's queue         
  |    +------------+                                                                 
  |                                                                                   
  |--> save fd/events/callback in that 'event source'                                 
  |                                                                                   
  |    +--------------------+                                                         
  +--> | source_io_register | register arg source's io to epoll                       
       +--------------------+                                                         
```

```
src/libsystemd/sd-event/sd-event.c                       
+--------------------+                                    
| source_io_register | : register arg source's io to epoll
+-|------------------+                                    
  |                                                       
  |--> set up 'epoll event'                               
  |                                                       
  |    +-----------+                                      
  +--> | epoll_ctl | register 'epoll event'               
       +-----------+                                      
```

```
src/libsystemd/sd-event/sd-event.c                                                                       
+-------------------+                                                                                     
| sd_event_add_time | : set up timer fields of event (fd/queues), prepare 'source' and add to those queues
+-|-----------------+                                                                                     
  |    +----------------------------+                                                                     
  |--> | clock_to_event_source_type | covnert clock mode to type                                          
  |    +----------------------------+                                                                     
  |                                                                                                       
  |--> if arg callback isn't provided                                                                     
  |    -                                                                                                  
  |    +--> use the default one                                                                           
  |         +--------------------+                                                                        
  |         | time_exit_callback | set event's exit fields                                                
  |         +--------------------+                                                                        
  |                                                                                                       
  |    +----------------------+                                                                           
  |--> | event_get_clock_data | given type, get target field in event accordingly                         
  |    +----------------------+                                                                           
  |    +------------------+                                                                               
  |--> | setup_clock_data | ensure the timer fd and two priority queues of 'clock data' are ready         
  |    +------------------+                                                                               
  |    +------------+                                                                                     
  |--> | source_new | prepare 'source' for event                                                          
  |    +------------+                                                                                     
  |                                                                                                       
  |--> set up 'source' (next, callback, enabled, ...)                                                     
  |                                                                                                       
  |    +-----------------------------+                                                                    
  +--> | event_source_time_prioq_put | place 'source' into clock_data's earliest and latest queues        
       +-----------------------------+                                                                    
```

```
src/libsystemd/sd-event/sd-event.c                                                         
+------------------+                                                                        
| setup_clock_data | : ensure the timer fd and two priority queues of 'clock data' are ready
+-|----------------+                                                                        
  |                                                                                         
  |--> if 'clock data' has no fd yet                                                        
  |    |                                                                                    
  |    |    +----------------------+                                                        
  |    +--> | event_setup_timer_fd | create timer fd, add to epoll, save in 'clock data'    
  |         +----------------------+                                                        
  |    +------------------------+                                                           
  |--> | prioq_ensure_allocated | ensure priority queue (clock_data->earliest) exists       
  |    +------------------------+                                                           
  |    +------------------------+                                                           
  +--> | prioq_ensure_allocated | ensure priority queue (clock_data->latest) exists         
       +------------------------+                                                           
```

```
src/libsystemd/sd-event/sd-event.c                                           
+----------------------+                                                      
| event_setup_timer_fd | : create timer fd, add to epoll, save in 'clock data'
+-|--------------------+                                                      
  |    +----------------+                                                     
  |--> | timerfd_create | create timer fd for later poll                      
  |    +----------------+                                                     
  |    +---------------------+                                                
  |--> | fd_move_above_stdio | ensure the fd isn't in range [0, 2]            
  |    +---------------------+                                                
  |    +-----------+                                                          
  |--> | epoll_ctl | add the timer fd to epoll                                
  |    +-----------+                                                          
  |                                                                           
  +--> save the timer fd in arg 'clock data'                                  
```

```
src/libsystemd/sd-event/sd-event.c                                                    
+--------------------+                                                                 
| sd_event_add_defer | : prepare 'source' for defer, set pending = true                
+-|------------------+                                                                 
  |                                                                                    
  |--> if arg callback isn't provided                                                  
  |    -                                                                               
  |    +--> use the default one                                                        
  |         +-----------------------+                                                  
  |         | generic_exit_callback | set event's exit fields                          
  |         +-----------------------+                                                  
  |    +------------+                                                                  
  |--> | source_new | prepare 'source' for defer                                       
  |    +------------+                                                                  
  |                                                                                    
  |--> set up source (install callback, set oneshot)                                   
  |                                                                                    
  |    +--------------------+                                                          
  +--> | source_set_pending | given target value and source type, handle pending action
       +--------------------+                                                          
```

```
src/libsystemd/sd-event/sd-event.c                                                                                 
+--------------------+                                                                                              
| source_set_pending | : given target value and source type, handle pending action                                  
+-|------------------+                                                                                              
  |                                                                                                                 
  |--> if 'pending' is the target value already, return                                                             
  |                                                                                                                 
  |--> if target value == true                                                                                      
  |    |                                                                                                            
  |    |    +-----------+                                                                                           
  |    +--> | prioq_put | place 'source' on event's pending queue                                                   
  |         +-----------+                                                                                           
  |                                                                                                                 
  |--> else, remove 'source' from pending queue                                                                     
  |                                                                                                                 
  |--> if the source is a timer source                                                                              
  |    |                                                                                                            
  |    |    +-----------------------------------+                                                                   
  |    +--> | event_source_time_prioq_reshuffle | reshuffle target source in event's both queues (earliest & latest)
  |         +-----------------------------------+                                                                   
  |                                                                                                                 
  |--> if source type is 'signal' and target value is 'false'                                                       
  |    |                                                                                                            
  |    |    +-------------+                                                                                         
  |    +--> | hashmap_get | get 'signal data' from event's hashmap                                                  
  |         +-------------+                                                                                         
  |                                                                                                                 
  +--> if souce type is 'inotify'                                                                                   
       -                                                                                                            
       +--> given target value, increment or decrement the pending# in 'inotify data' (embedded in 'source')        
```

```
src/libsystemd/sd-event/sd-event.c                                                                       
+-----------------------------------+                                                                     
| event_source_time_prioq_reshuffle | : reshuffle target source in event's both queues (earliest & latest)
+-|---------------------------------+                                                                     
  |                                                                                                       
  |--> determine and get 'clock data' from event                                                          
  |                                                                                                       
  |    +-----------------+                                                                                
  |--> | prioq_reshuffle | reshuffle target node in queue (earliest)                                      
  |    +-----------------+                                                                                
  |    +-----------------+                                                                                
  +--> | prioq_reshuffle | reshuffle target node in queue (latest)                                        
       +-----------------+                                                                                
```

```
src/basic/prioq.c                                                                                                     
+-----------------+                                                                                                    
| prioq_reshuffle | : reshuffle target node in queue                                                                   
+-|---------------+                                                                                                    
  |    +-----------+                                                                                                   
  |--> | find_item | given idx, find target item in priority queue                                                     
  |    +-----------+                                                                                                   
  |                                                                                                                    
  |--> if it's not even there, return                                                                                  
  |                                                                                                                    
  |    +--------------+                                                                                                
  |--> | shuffle_down | apply binary tree search idea on queue, adjust target node downward (do nothing if unnecessary)
  |    +--------------+                                                                                                
  |    +------------+                                                                                                  
  +--> | shuffle_up | anjudst replaced node upward (could be the original one)                                         
       +------------+                                                                                                  
```

```
src/libsystemd/sd-event/sd-event.c                                            
+-------------------+                                                          
| sd_event_add_post | : prepare 'source' for post and add to event's hash table
+-|-----------------+                                                          
  |    +------------+                                                          
  |--> | source_new | create 'source' for post                                 
  |    +------------+                                                          
  |                                                                            
  |--> set up source (install callback, set event_on)                          
  |                                                                            
  +--> +----------------+                                                      
       | set_ensure_put | insert 'source' into event's hash table of post      
       +----------------+                                                      
```

```
src/libsystemd/sd-event/sd-event.c                                            
+-------------------+                                                          
| sd_event_add_exit | : prepare 'source' for exit and add to even'ts exit queue
+-|-----------------+                                                          
  |    +------------------------+                                              
  |--> | prioq_ensure_allocated | ensure priority queue is allocated           
  |    +------------------------+                                              
  |    +------------+                                                          
  |--> | source_new | prepare 'source' for exit                                
  |    +------------+                                                          
  |                                                                            
  |--> set up 'source' (install callback, set oneshot)                         
  |                                                                            
  |    +-----------+                                                           
  +--> | prioq_put | insert 'source' into event's exit queue                   
       +-----------+                                                           
```

```
src/libsystemd/sd-event/sd-event.c                                                                   
+------------------+                                                                                  
| sd_event_prepare | : register event's sources to epoll, arm all kinds of timers, set state = armed  
+-|----------------+                                                                                  
  |                                                                                                   
  |--> set event state = prepared                                                                     
  |                                                                                                   
  |    +---------------+                                                                              
  |--> | event_prepare | for each source in event: register bus of source to epoll                    
  |    +---------------+                                                                              
  |                                                                                                   
  |--> set event state = initial                                                                      
  |                                                                                                   
  |    +----------------------------------+                                                           
  |--> | event_memory_pressure_write_list | for each source in event queue: write data and free buffer
  |    +----------------------------------+                                                           
  |    +-----------------+                                                                            
  |--> | event_arm_timer | determine sleep time and arm timer                                         
  |    +-----------------+                                                                            
  |    +-----------------+                                                                            
  |--> | event_arm_timer | (the above is for real time, this is for boot time)                        
  |    +-----------------+                                                                            
  |    +-----------------+                                                                            
  |--> | event_arm_timer | (the is for monotonic)                                                     
  |    +-----------------+                                                                            
  |    +-----------------+                                                                            
  |--> | event_arm_timer | (the is for realtime alarm)                                                
  |    +-----------------+                                                                            
  |    +-----------------+                                                                            
  |--> | event_arm_timer | (the is for boottime alarm)                                                
  |    +-----------------+                                                                            
  |    +----------------------------+                                                                 
  |--> | event_close_inode_data_fds | for each fd in event's close list: close fd and remove from list
  |    +----------------------------+                                                                 
  |                                                                                                   
  +--> set state = armed                                                                              
```

```
src/libsystemd/sd-event/sd-event.c                                                               
+---------------+                                                                                 
| event_prepare | : for each source in event: register bus of source to epoll                     
+-|-------------+                                                                                 
  |                                                                                               
  |--> endless loop                                                                               
  |                                                                                               
  |    +------------+                                                                             
  |--> | prioq_peek | get the first 'source' from event's prepare queue                           
  |    +------------+                                                                             
  |                                                                                               
  |--> source iteration = event iteration                                                         
  |                                                                                               
  |    +-----------------+                                                                        
  |--> | prioq_reshuffle | adjust 'source' position in prepare queue                              
  |    +-----------------+                                                                        
  |                                                                                               
  |--> call ->prepare(), e.g.,                                                                    
  |    +------------------+                                                                       
  |    | prepare_callback | register bus source to epoll, determine timeout, set enabled = oneshot
  |    +------------------+                                                                       
  |                                                                                               
  +--> if source ref == 0                                                                         
       |                                                                                          
       |     +-------------+                                                                      
       +-->  | source_free | given source type, free itself and related resource (fd, buffer)     
             +-------------+                                                                      
```

```
src/libsystemd/sd-bus/sd-bus.c                                                                       
+------------------+                                                                                  
| prepare_callback | : register bus source to epoll, determine timeout, set enabled = oneshot         
+-|----------------+                                                                                  
  |    +-------------------+                                                                          
  |--> | sd_bus_get_events | given bus state, determine flags (poll in and/or poll out)               
  |    +-------------------+                                                                          
  |                                                                                                   
  |--> if bus has different fd for input and output                                                   
  |    |                                                                                              
  |    |    +-------------------------------+                                                         
  |    |--> | sd_event_source_set_io_events | register (source, event) to epoll, save events in source
  |    |    +-------------------------------+                                                         
  |    |    +-------------------------------+                                                         
  |    +--> | sd_event_source_set_io_events | (the above one is for input, this one is for output)    
  |         +-------------------------------+                                                         
  |                                                                                                   
  |--> else                                                                                           
  |    |                                                                                              
  |    |    +-------------------------------+                                                         
  |    +--> | sd_event_source_set_io_events | (this one is for both input and output)                 
  |         +-------------------------------+                                                         
  |    +--------------------+                                                                         
  |--> | sd_bus_get_timeout | given bus state, determine timeout                                      
  |    +--------------------+                                                                         
  |                                                                                                   
  |--> if timeout is meaningful (not forever)                                                         
  |    |                                                                                              
  |    |    +--------------------------+                                                              
  |    +--> | sd_event_source_set_time | unset pending, save timeout in source, reshuffle the prio-q  
  |         +--------------------------+                                                              
  |    +-----------------------------+                                                                
  +--> | sd_event_source_set_enabled | given target value, register or unregister source accordingly  
       +-----------------------------+                                                                
```

```
src/libsystemd/sd-event/sd-event.c                                                         
+-------------------------------+                                                           
| sd_event_source_set_io_events | : register (source, event) to epoll, save events in source
+-|-----------------------------+                                                           
  |    +--------------------+                                                               
  |--> | source_set_pending | set source pending = false                                    
  |    +--------------------+                                                               
  |                                                                                         
  |--> if source is online (enabled or oneshot)                                             
  |    |                                                                                    
  |    |    +--------------------+                                                          
  |    +--> | source_io_register | register (source, event) to epoll                        
  |         +--------------------+                                                          
  |                                                                                         
  +--> save arg events (poll_in and/or poll_out) in source                                  
```

```
src/libsystemd/sd-event/sd-event.c                                                                       
+-----------------------------+                                                                           
| sd_event_source_set_enabled | : given target value, register or unregister source accordingly           
+-|---------------------------+                                                                           
  |                                                                                                       
  |--> if ->enabled == target value already, return                                                       
  |                                                                                                       
  |--> if target value is off                                                                             
  |    |                                                                                                  
  |    |    +----------------------+                                                                      
  |    +--> | event_source_offline | given source type, unregister it accordingly                         
  |         +----------------------+                                                                      
  |                                                                                                       
  |--> else                                                                                               
  |    |                                                                                                  
  |    |--> if source is already on                                                                       
  |    |    -                                                                                             
  |    |    +--> source->enabled = target value, and return                                               
  |    |                                                                                                  
  |    |    +---------------------+                                                                       
  |    +--> | event_source_online | given source type, register source to epoll accordingly               
  |         +---------------------+                                                                       
  |    +---------------------------------+                                                                
  +--> | event_source_pp_prioq_reshuffle | reshuffle source's 'pending' and 'prepare' queues if they exist
       +---------------------------------+                                                                
```

```
src/libsystemd/sd-event/sd-event.c                                                      
+----------------------+                                                                 
| event_source_offline | : given source type, unregister it accordingly                  
+-|--------------------+                                                                 
  |                                                                                      
  |--> if event will be disabled                                                         
  |    |                                                                                 
  |    |    +--------------------+                                                       
  |    +--> | source_set_pending | unset pending flag                                    
  |         +--------------------+                                                       
  |                                                                                      
  |--> switch source type                                                                
  |    caee io                                                                           
  |    -    +----------------------+                                                     
  |    +--> | source_io_unregister | unregister io from epoll                            
  |         +----------------------+                                                     
  |    case signal                                                                       
  |    -    +----------------------+                                                     
  |    +--> | event_gc_signal_data | unset signal bit from all three possible masks      
  |         +----------------------+                                                     
  |    case child                                                                        
  |    |--> if it's a pid event source                                                   
  |    |    -    +-------------------------------+                                       
  |    |    +--> | source_child_pidfd_unregister | unregister source from epoll          
  |    |         +-------------------------------+                                       
  |    +--> else                                                                         
  |         -    +----------------------+                                                
  |         +--> | event_gc_signal_data | unset signal bit from all three possible masks 
  |              +----------------------+                                                
  |    case exit                                                                         
  |         -    +-----------------+                                                     
  |         +--> | prioq_reshuffle | adjust its position in queue                        
  |          +-->+-----------------+                                                     
  |    case memory-pressure                                                              
  |              +-----------------------------------+                                   
  |              | source_memory_pressure_unregister | unregister source from epoll      
  |              +-----------------------------------+                                   
  |    +-----------------------------------+                                             
  +--> | event_source_time_prioq_reshuffle | reshuffle source in two queues of clock data
       +-----------------------------------+                                             
```

```
src/libsystemd/sd-event/sd-event.c                                      
+----------------------+                                                 
| event_gc_signal_data | : unset signal bit from all three possible masks
+-|--------------------+                                                 
  |                                                                      
  |--> if event still has interests in signal, return                    
  |                                                                      
  |--> if 'priority' is provided && in event hash map                    
  |    |                                                                 
  |    |    +--------------------------+                                 
  |    +--> | event_unmask_signal_data | unset the bit in mask           
  |         +--------------------------+                                 
  |                                                                      
  |--> if arg signal is in signal source                                 
  |    |                                                                 
  |    |    +--------------------------+                                 
  |    +--> | event_unmask_signal_data | unset the bit in mask           
  |         +--------------------------+                                 
  |                                                                      
  +--> if there's zero priority in event hash map                        
       |                                                                 
       |    +--------------------------+                                 
       +--> | event_unmask_signal_data | unset the bit in mask           
            +--------------------------+                                 
```

```
src/libsystemd/sd-event/sd-event.c                                                   
+---------------------+                                                               
| event_source_online | : given source type, register source to epoll accordingly     
+-|-------------------+                                                               
  |                                                                                   
  |--> if source will be turned on                                                    
  |    |                                                                              
  |    |    +--------------------+                                                    
  |    +--> | source_set_pending | set pending = false                                
  |         +--------------------+                                                    
  |                                                                                   
  |--> switch source type                                                             
  |    case io                                                                        
  |    -    +--------------------+                                                    
  |    +--> | source_io_register | register source to epoll                           
  |         +--------------------+                                                    
  |    case signal                                                                    
  |    -    +------------------------+                                                
  |    +--> | event_make_signal_data | ensure target signal is monitored by epoll     
  |         +------------------------+                                                
  |    case child                                                                     
  |    ---> if source is a pid event source                                           
  |         -    +-----------------------------+                                      
  |         +--> | source_child_pidfd_register | register source to epoll             
  |              +-----------------------------+                                      
  |         else                                                                      
  |         -    +------------------------+                                           
  |         +--> | event_make_signal_data | ensure target signal is monitored by epoll
  |              +------------------------+                                           
  |    case memory_pressure                                                           
  |    -    +---------------------------------+                                       
  |    +--> | source_memory_pressure_register | register source to epoll              
  |         +---------------------------------+                                       
  |                                                                                   
  |--> save arg 'enabled' and ratelimited' to source                                  
  |                                                                                   
  +--> reshuffle related prio-q if necessary                                          
```

```
src/libsystemd/sd-event/sd-event.c                                        
+------------------------+                                                 
| event_make_signal_data | : ensure target signal is monitored by epoll    
+-|----------------------+                                                 
  |                                                                        
  |--> given priority, get signal_data from event hash table               
  |                                                                        
  |--> if arg signal is already saved in signal data, return               
  |                                                                        
  |--> else                                                                
  |    |                                                                   
  |    |--> prepare signal data                                            
  |    |                                                                   
  |    |    +--------------------+                                         
  |    +--> | hashmap_ensure_put | insert 'signal data' to event hash table
  |         +--------------------+                                         
  |                                                                        
  |--> add signal to mask                                                  
  |                                                                        
  |    +----------+                                                        
  |--> | signalfd | register mask to signal fd (which might not exist yet) 
  |    +----------+                                                        
  |                                                                        
  |--> if it exists already, return                                        
  |                                                                        
  |    +-----------+                                                       
  +--> | epoll_ctl | registe signal fd to epoll                            
       +-----------+                                                       
```

```
src/libsystemd/sd-event/sd-event.c                                               
+-------------+                                                                   
| source_free | : given source type, free itself and related resource (fd, buffer)
+-|-----------+                                                                   
  |                                                                               
  |--> if source type is io, close fd                                             
  |                                                                               
  |--> if source type is child                                                    
  |    |                                                                          
  |    |--> if child is process-owned (?)                                         
  |    |    -                                                                     
  |    |    +--> if it doesn't exit yet                                           
  |    |         |                                                                
  |    |         |--> if it has pid fd                                            
  |    |         |    |                                                           
  |    |         |    |    +-------------------+                                  
  |    |         |    +--> | pidfd_send_signal | send kill to that child thru fd  
  |    |         |         +-------------------+                                  
  |    |         |                                                                
  |    |         +--> else                                                        
  |    |              |                                                           
  |    |              |    +------+                                               
  |    |              +--> | kill | send kill to that child thru pid              
  |    |                   +------+                                               
  |    |                                                                          
  |    |--> if child isn't waited (nobody wants it?)                              
  |    |    |                                                                     
  |    |    |    +--------+                                                       
  |    |    +--> | waitid | try to adopt it                                       
  |    |         +--------+                                                       
  |    |                                                                          
  |    +--> if child is pid-owned (?), close fd                                   
  |                                                                               
  |--> if source type is 'memory pressure'                                        
  |    -                                                                          
  |    +--> close fd and free buffer                                              
  |                                                                               
  |--> if source has destroy_callback()                                           
  |    -                                                                          
  |    +--> call it                                                               
  |                                                                               
  +--> free source related mem                                                    
```

```
src/libsystemd/sd-event/sd-event.c                                                              
+----------------------------------+                                                             
| event_memory_pressure_write_list | : for each source in event queue: write data and free buffer
+-|--------------------------------+                                                             
  |                                                                                              
  +--> endless loop                                                                              
       |                                                                                         
       |--> pop first node (source) from event's queue                                           
       |                                                                                         
       |--> if nothing left, break                                                               
       |                                                                                         
       |    +------------------------------+                                                     
       +--> | source_memory_pressure_write | write data and free buffer                          
            +------------------------------+                                                     
```

```
src/libsystemd/sd-event/sd-event.c                          
+------------------------------+                             
| source_memory_pressure_write | : write data and free buffer
+-|----------------------------+                             
  |                                                          
  |--> set locked = true (for later writing)                 
  |                                                          
  |    +-------+                                             
  |--> | write |                                             
  |    +-------+                                             
  |                                                          
  +--> free write buffer                                     
```

```
src/libsystemd/sd-event/sd-event.c                                                                 
+-----------------+                                                                                 
| event_arm_timer | : determine sleep time and arm timer                                            
+-|---------------+                                                                                 
  |    +------------+                                                                               
  |--> | prioq_peek | get the earliest source from clock data                                       
  |    +------------+                                                                               
  |                                                                                                 
  |--> if it's not on                                                                               
  |    -                                                                                            
  |    +--> disarm the timer and return                                                             
  |                                                                                                 
  |    +------------+                                                                               
  |--> | prioq_peek | get the latest source from clock data                                         
  |    +------------+                                                                               
  |    +---------------+                                                                            
  |--> | sleep_between | given the earliest and latest, calculate the sleep time till the next timer
  |    +---------------+                                                                            
  |    +----------------+                                                                           
  |--> | timespec_store | save the sleep time in struct                                             
  |    +----------------+                                                                           
  |    +-----------------+                                                                          
  |--> | timerfd_settime | arm timer                                                                
  |    +-----------------+                                                                          
  |                                                                                                 
  +--> save 'sleep time' value in clock data                                                        
```

```
src/libsystemd/sd-event/sd-event.c                                                     
+---------------+                                                                       
| sd_event_wait | : wait for each child finishes                                        
+-|-------------+                                                                       
  |                                                                                     
  |--> for each threshold (from max to min)                                             
  |    |                                                                                
  |    |    +---------------+                                                           
  |    |--> | process_epoll | wait for data and read it, set source pending = true      
  |    |    +---------------+                                                           
  |    |    +---------------+                                                           
  |    |--> | process_child | wait till child finishes, set pending = true              
  |    |    +---------------+                                                           
  |    |                                                                                
  |    +--> if no new child event, break early                                          
  |                                                                                     
  |    +------------------+                                                             
  |--> | process_watchdog | arm a watchdog                                              
  |    +------------------+                                                             
  |    +-----------------+                                                              
  |--> | process_inotify | for each source has interest in the event, set pending = true
  |    +-----------------+                                                              
  |    +---------------+                                                                
  |--> | process_timer | find the earliest timer, set pending = true                    
  |    +---------------+ (for realtime)                                                 
  |    +---------------+                                                                
  |--> | process_timer | for boottime                                                   
  |    +---------------+                                                                
  |    +---------------+                                                                
  |--> | process_timer | for realtime_alarm                                             
  |    +---------------+                                                                
  |    +---------------+                                                                
  |--> | process_timer | for boottime_alarm                                             
  |    +---------------+                                                                
  |    +---------------+                                                                
  |--> | process_timer | for monotonic                                                  
  |    +---------------+                                                                
  |                                                                                     
  |--> if there's still source in event's pending queue                                 
  |    |                                                                                
  |    |--> set event state = pending                                                   
  |    +--> return                                                                      
  |                                                                                     
  +--> set event state = initial                                                        
```

```
src/libsystemd/sd-event/sd-event.c                                                                                   
+---------------+                                                                                                     
| process_epoll | : wait for data and read it, set source pending = true                                              
+---------------+                                                                                                     
  |--> adjust event's queue size                                                                                      
  |--> endless loop                                                                                                   
  |    |    +-----------------+                                                                                       
  |    |--> | epoll_wait_usec |                                                                                       
  |    |    +-----------------+                                                                                       
  |    +--> if condition met, break                                                                                   
  |--> if it's called first time                                                                                      
  |    -    +----------------------+                                                                                  
  |    +--> | triple_timestamp_get | get current time of three bases                                                  
  |         +----------------------+                                                                                  
  +--> for each node in event queue                                                                                   
       |--> if source type is 'watchdog'                                                                              
       |    -    +-------------+                                                                                      
       |    +--> | flush_timer | read data from fd (unused), set next = infinity                                      
       |         +-------------+                                                                                      
       +--> else                                                                                                      
            +--> switch wakeup type                                                                                   
                 case event_source                                                                                    
                 +--> switch source_type                                                                              
                      case io                                                                                         
                      -    +------------+                                                                             
                      +--> | process_io | save event in source, set pending = true                                    
                           +------------+                                                                             
                      case pidfd                                                                                      
                      -    +---------------+                                                                          
                      +--> | process_pidfd | wait till child exits                                                    
                           +---------------+                                                                          
                      case memory pressure                                                                            
                      -    +-------------------------+                                                                
                      +--> | process_memory_pressure | save event in source, set pending = true                       
                           +-------------------------+                                                                
                 case clock data                                                                                      
                 -    +-------------+                                                                                 
                 +--> | flush_timer | read data from fd (unused), set next = infinity                                 
                      +-------------+                                                                                 
                 case signal data                                                                                     
                 -    +----------------+                                                                              
                 +--> | process_signal | read signal to get target source, save it in signal_data, set pending = true 
                      +----------------+                                                                              
                 case inotify data                                                                                    
                 -    +-------------------------+                                                                     
                 +--> | event_inotify_data_read | given inotify_data, read data, prepend inotify_data to event's queue
                      +-------------------------+                                                                     
```

```
src/libsystemd/sd-event/sd-event.c                                                              
+----------------+                                                                               
| process_signal | : read signal to get target source, save it in signal_data, set pending = true
+-|--------------+                                                                               
  |                                                                                              
  |--> is signal_data->current exists (got somethign to do), return                              
  |                                                                                              
  |--> read signal info                                                                          
  |                                                                                              
  |--> given signal, get source                                                                  
  |                                                                                              
  |--> signal_data->current = source                                                             
  |                                                                                              
  |--> set pending = true                                                                        
  |                                                                                              
  +--> if the priority is smaller, update to min_priority                                        
```

```
src/libsystemd/sd-event/sd-event.c                             
+---------------+                                               
| process_child | : wait till child finishes, set pending = true
+-|-------------+                                               
  |                                                             
  +--> for each child source                                    
       |                                                        
       |    +--------+                                          
       |--> | waitid | wait till child finishes                 
       |    +--------+                                          
       |                                                        
       +--> if we got something                                 
            |                                                   
            |--> set pending = true                             
            |                                                   
            +--> update min_priority                            
```

```
src/libsystemd/sd-event/sd-event.c                                  
+------------------+                                                 
| process_watchdog | : arm a watchdog                                
+-|----------------+                                                 
  |    +-----------+                                                 
  |--> | sd_notify | get addr from env 'NOTIFY_SOCKET', create socket and connect, send msg
  |    +-----------+                                                 
  |    +--------------+                                              
  +--> | arm_watchdog | calculate wait interval, set thru watchdog fd
       +--------------+                                              
```

```
src/libsystemd/sd-daemon/sd-daemon.c                                                                         
+-----------+                                                                                                 
| sd_notify | : get addr from env 'NOTIFY_SOCKET', create socket and connect, send msg                        
+------------------------+                                                                                    
| sd_pid_notify_with_fds | : get addr from env 'NOTIFY_SOCKET', create socket and connect, send msg           
+-|----------------------+                                                                                    
  |    +------------------------------+                                                                       
  |--> | pid_notify_with_fds_internal | get addr from env 'NOTIFY_SOCKET', create socket and connect, send msg
  |    +------------------------------+                                                                       
  |                                                                                                           
  +--> if arg unset_environment is set                                                                        
       -                                                                                                      
       +--> unset 'NOTIFY_SOCKET'                                                                             
```

```
src/libsystemd/sd-daemon/sd-daemon.c                                                                    
+------------------------------+                                                                         
| pid_notify_with_fds_internal | : get addr from env 'NOTIFY_SOCKET', create socket and connect, send msg
+-|----------------------------+                                                                         
  |                                                                                                      
  |--> get env 'NOTIFY_SOCKET'                                                                           
  |                                                                                                      
  |--> parse env string into address                                                                     
  |                                                                                                      
  |    +--------+                                                                                        
  |--> | socket | create socket                                                                          
  |    +--------+                                                                                        
  |    +---------+                                                                                       
  |--> | connect |                                                                                       
  |    +---------+                                                                                       
  |    +---------------+                                                                                 
  |--> | fd_inc_sndbuf | config send buffer                                                              
  |    +---------------+                                                                                 
  |                                                                                                      
  |--> alloc msg                                                                                         
  |                                                                                                      
  |    +---------+                                                                                       
  +--> | sendmsg |                                                                                       
       +---------+                                                                                       
```

```
src/libsystemd/sd-event/sd-event.c                                                                                       
+-----------------+                                                                                                       
| process_inotify | : for each source has interest in the event, set pending = true                                       
+-|---------------+                                                                                                       
  |                                                                                                                       
  +--> for each inotify_data in event                                                                                     
       |                                                                                                                  
       |    +----------------------------+                                                                                
       +--> | event_inotify_data_process | for each source has interest in the inotify_data: set pending = true (trigger?)
            +----------------------------+                                                                                
```

```
src/libsystemd/sd-event/sd-event.c                                          
+-------------------+                                                        
| sd_event_dispatch | : dispatch 1st source from event's pending queue       
+-|-----------------+                                                        
  |                                                                          
  |--> is exit_requested is set                                              
  |    |                                                                     
  |    |    +---------------+                                                
  |    +--> | dispatch_exit | get 1st source from event, dispatch that source
  |         +---------------+                                                
  |    +--------------------+                                                
  |--> | event_next_pending | get 1st source from event's pending queue      
  |    +--------------------+                                                
  |                                                                          
  |--> set event state = running                                             
  |                                                                          
  |    +-----------------+                                                   
  |--> | source_dispatch | given source type, call its callback accordingly  
  |    +-----------------+                                                   
  |                                                                          
  +--> set event state = initial                                             
```

```
src/libsystemd/sd-event/sd-event.c                                        
+---------------+                                                          
| dispatch_exit | : get 1st source from event, dispatch that source        
+-|-------------+                                                          
  |    +------------+                                                      
  |--> | prioq_peek | get first source from event's exit queue             
  |    +------------+                                                      
  |                                                                        
  |--> event iteration++                                                   
  |                                                                        
  |--> set event state = exiting                                           
  |                                                                        
  |    +-----------------+                                                 
  |--> | source_dispatch | given source type, call its callback accordingly
  |    +-----------------+                                                 
  |                                                                        
  +--> set event state = initial                                           
```

```
src/libsystemd/sd-event/sd-event.c                                                    
+-----------------+                                                                    
| source_dispatch | : given source type, call its callback accordingly                 
+-|---------------+                                                                    
  |                                                                                    
  +--> switch source type                                                              
       case io:                                                                        
       +--> call ->callback(), e.g.,                                                   
            +------------------+                                                       
            | io_exit_callback | set event's exit fields                               
            +------------------+                                                       
       case time_xxx                                                                   
       +--> call ->callback(), e.g.,                                                   
            +--------------------+                                                     
            | time_exit_callback | set event's exit fields                             
            +--------------------+                                                     
       case signal                                                                     
       +--> call ->callback(), e.g.,                                                   
            +----------------------+                                                   
            | signal_exit_callback | set event's exit fields                           
            +----------------------+                                                   
       case child                                                                      
       +--> call ->callback(), e.g.,                                                   
            +---------------------+                                                    
            | child_exit_callback | set event's exit fields                            
            +---------------------+                                                    
            if child is zombie, reap it                                                
       case defer                                                                      
       +--> call ->callback(), e.g.,                                                   
            +-----------------------+                                                  
            | generic_exit_callback | set event's exit fields                          
            +-----------------------+                                                  
       case post                                                                       
       +--> call ->callback(), e.g.,                                                   
            +-----------------------+                                                  
            | generic_exit_callback | set event's exit fields                          
            +-----------------------+                                                  
       case exit                                                                       
       +--> call ->callback(), e.g.,                                                   
            +---------------+                                                          
            | quit_callback | if 'close on exit' is set, flush bus wqueue and close bus
            +---------------+                                                          
       case inotify                                                                    
       +--> call ->callback(), e.g.,                                                   
            +-----------------------+                                                  
            | inotify_exit_callback | set event's exit fields                          
            +-----------------------+                                                  
       case memory_pressure                                                            
       +--> call ->callback(), e.g.,                                                   
            +--------------------------+                                               
            | memory_pressure_callback | free unused hash maps, return unused memory to kernel
            +--------------------------+                                               
```

```
src/libsystemd/sd-bus/sd-bus.c                                              
+---------------+                                                            
| quit_callback | : if 'close on exit' is set, flush bus wqueue and close bus
+-|-------------+                                                            
  |                                                                          
  +--> if bus has set 'close on exit'                                        
       |                                                                     
       |    +--------------+                                                 
       |--> | sd_bus_flush | ensure bus is running, flush bus wqueue         
       |    +--------------+                                                 
       |    +--------------+                                                 
       +--> | sd_bus_close | release sources/event/fd of bus
            +--------------+                                                 
```

```
src/libsystemd/sd-bus/sd-bus.c                                                      
+--------------+                                                                     
| sd_bus_flush | : ensure bus is running, flush bus wqueue                           
+-|------------+                                                                     
  |    +--------------------+                                                        
  |--> | bus_ensure_running | ensure bus is running                                  
  |    +--------------------+                                                        
  |                                                                                  
  +--> endless loop                                                                  
       |                                                                             
       |    +-----------------+                                                      
       |--> | dispatch_wqueue | for each msg in write queue: write to bus's output fd
       |    +-----------------+                                                      
       |                                                                             
       |--> if nothing left in wqueue, return                                        
       |                                                                             
       |    +----------+                                                             
       +--> | bus_poll | determine flags and timeout, then poll
            +----------+                                                             
```

```
src/libsystemd/sd-bus/sd-bus.c                                      
+--------------------+                                               
| bus_ensure_running | : ensure bus is running                       
+-|------------------+                                               
  |                                                                  
  |--> if it's already running, return                               
  |                                                                  
  +--> endless loop                                                  
       |                                                             
       |    +----------------+                                       
       |--> | sd_bus_process | given bus state, handle it accordingly
       |    +----------------+                                       
       |                                                             
       |--> if bus state == running, return                          
       |                                                             
       |--> if not running, but everything goes well, continue       
       |                                                             
       |    +-------------+                                          
       +--> | sd_bus_wait | determine flags and timeout, then poll
            +-------------+                                          
```

```
src/libsystemd/sd-bus/sd-bus.c                                                                                                          
+----------------+                                                                                                                       
| sd_bus_process | : given bus state, handle it accordingly                                                                              
+----------------------+                                                                                                                 
| bus_process_internal | : given bus state, handle it accordingly                                                                        
+-|--------------------+                                                                                                                 
  |                                                                                                                                      
  +--> switch bus state                                                                                                                  
       case watch_bind                                                                                                                   
       -    +-------------------------------+                                                                                            
       +--> | bus_socket_process_watch_bind | create socket for input/output, connect, prepare io/inotify sources, register them to epoll
            +-------------------------------+                                                                                            
       case opening                                                                                                                      
       -    +----------------------------+                                                                                               
       +--> | bus_socket_process_opening | given bus address, prepare socket & connect, register sources to epoll                        
            +----------------------------+                                                                                               
       case authenticating                                                                                                               
       -    +-----------------------------------+                                                                                        
       +--> | bus_socket_process_authenticating | (skip)                                                                                 
            +-----------------------------------+                                                                                        
       case running/hello                                                                                                                
       -    +-----------------+                                                                                                          
       +--> | process_running | handle timeout, dispatch write queue, receive msg and process it                                         
            +-----------------+                                                                                                          
       case closing                                                                                                                      
       -    +-----------------+                                                                                                          
       +--> | process_closing | prepare msg of 'disconnected', close bus and exit                                                        
            +-----------------+                                                                                                          
```

```
src/libsystemd/sd-bus/bus-socket.c                                                                                                      
+-------------------------------+                                                                                                        
| bus_socket_process_watch_bind | : create socket for input/output, connect, prepare io source and inotify source, register them to epoll
+-|-----------------------------+                                                                                                        
  |    +----------+                                                                                                                      
  |--> | flush_fd | wait and read data to empty the queue                                                                                
  |    +----------+                                                                                                                      
  |    +--------------------+                                                                                                            
  |--> | bus_socket_connect | create one socket for both input/output, connect, close inotify                                            
  |    +--------------------+                                                                                                            
  |    +----------------------+                                                                                                          
  |--> | bus_attach_io_events | for input/output fd: prepare source and register to epoll & set source's priority/description            
  |    +----------------------+                                                                                                          
  |    +--------------------------+                                                                                                      
  +--> | bus_attach_inotify_event | prepare source and register to epoll & set source's priority/description                             
       +--------------------------+                                                                                                      
```

```
src/libsystemd/sd-bus/bus-socket.c                                                     
+--------------------+                                                                  
| bus_socket_connect | : create one socket for both input/output, connect, close inotify
+-|------------------+                                                                  
  |                                                                                     
  |--> endless loop                                                                     
  |    |                                                                                
  |    |--> create socket as input fd for bus                                           
  |    |                                                                                
  |    |    +------------------+                                                        
  |    |--> | bind_description | prepare bind name and bind to sock fd                  
  |    |    +------------------+                                                        
  |    |                                                                                
  |    |--> output fd = input fd                                                        
  |    |                                                                                
  |    |    +------------------+                                                        
  |    |--> | bus_socket_setup | set input/output buffer size to 8mb separately         
  |    |    +------------------+                                                        
  |    |    +---------+                                                                 
  |    |--> | connect |                                                                 
  |    |    +---------+                                                                 
  |    |                                                                                
  |    +--> if everything goes well, break                                              
  |                                                                                     
  |    +----------------------+                                                         
  +--> | bus_close_inotify_fd | we have socket now, don't need inotify anymore          
       +----------------------+                                                         
```

```
src/libsystemd/sd-bus/bus-socket.c                            
+------------------+                                           
| bind_description | : prepare bind name and bind to sock fd   
+-|----------------+                                           
  |    +------------------------+                              
  |--> | sd_bus_get_description | get bus description          
  |    +------------------------+                              
  |    +------------------+                                    
  |--> | get_process_comm | get process name                   
  |    +------------------+                                    
  |                                                            
  |--> assemble bind_name = /bus/$process_name/$bus_description
  |                                                            
  |    +----------------------+                                
  |--> | sockaddr_un_set_path | save bind_name in 'sock addr'  
  |    +----------------------+                                
  |    +------+                                                
  +--> | bind | bind 'sock addr' to fd                         
       +------+                                                
```

```
src/libsystemd/sd-bus/sd-bus.c                                                                                         
+----------------------+                                                                                                
| bus_attach_io_events | : for input/output fd: prepare source and register to epoll & set source's priority/description
+-|--------------------+                                                                                                
  |                                                                                                                     
  |--> if bus has no event source of input                                                                              
  |    |                                                                                                                
  |    |    +-----------------+                                                                                         
  |    +--> | sd_event_add_io | prepare 'source' and add to arg 'event, register the source's io                        
  |    |    +-----------------+                                                                                         
  |    |    +-----------------------------+                                                                             
  |    |--> | sd_event_source_set_prepare | update s->callback, insert to or remove from queue accordingly              
  |    |    +-----------------------------+                                                                             
  |    |    +------------------------------+                                                                            
  |    |--> | sd_event_source_set_priority | s->priority = priority, reposition source in data structure                
  |    |    +------------------------------+                                                                            
  |    |    +---------------------------------+                                                                         
  |    +--> | sd_event_source_set_description | duplicate description into source                                       
  |         +---------------------------------+                                                                         
  |                                                                                                                     
  |--> else                                                                                                             
  |    |                                                                                                                
  |    |    +---------------------------+                                                                               
  |    +--> | sd_event_source_set_io_fd | add new fd to epoll, remove the old one                                       
  |         +---------------------------+                                                                               
  |                                                                                                                     
  +--> if input fd != output fd                                                                                         
       |                                                                                                                
       |--> if bus has no event source of output                                                                        
       |    |                                                                                                           
       |    |    +-----------------+                                                                                    
       |    +--> | sd_event_add_io | prepare 'source' and add to arg 'event, register the source's io                   
       |    |    +-----------------+                                                                                    
       |    |    +------------------------------+                                                                       
       |    |--> | sd_event_source_set_priority | s->priority = priority, reposition source in data structure           
       |    |    +------------------------------+                                                                       
       |    |    +---------------------------------+                                                                    
       |    +--> | sd_event_source_set_description | duplicate description into source                                  
       |         +---------------------------------+                                                                    
       |                                                                                                                
       +--> else                                                                                                        
            |                                                                                                           
            |    +---------------------------+                                                                          
            +--> | sd_event_source_set_io_fd | add new fd to epoll, remove the old one                                  
                 +---------------------------+                                                                          
```

```
src/libsystemd/sd-event/sd-event.c                                                             
+-----------------------------+                                                                 
| sd_event_source_set_prepare | : update s->callback, insert to or remove from queue accordingly
+-|---------------------------+                                                                 
  |    +------------------------+                                                               
  |--> | prioq_ensure_allocated | ensure 'prepare' queue exists                                 
  |    +------------------------+                                                               
  |                                                                                             
  |--> install arg callback to source                                                           
  |                                                                                             
  |--> if callback is provided                                                                  
  |    |                                                                                        
  |    |    +-----------+                                                                       
  |    +--> | prioq_put | insert source to 'prepare' queue                                      
  |         +-----------+                                                                       
  |                                                                                             
  +--> else                                                                                     
       |                                                                                        
       |    +--------------+                                                                    
       +--> | prioq_remove | remove source from 'prepare' queue                                 
            +--------------+                                                                    
```

```
src/libsystemd/sd-event/sd-event.c                                                                       
+------------------------------+                                                                          
| sd_event_source_set_priority | : s->priority = priority, reposition source in data structure            
+-|----------------------------+                                                                          
  |                                                                                                       
  |--> if source type == inotify                                                                          
  |    |                                                                                                  
  |    |    +-------------------------+                                                                   
  |    |--> | event_make_inotify_data | ensure inotify_data exists and registered to epoll                
  |    |    +-------------------------+                                                                   
  |    |    +-----------------------+                                                                     
  |    |--> | event_make_inode_data | ensure inode_data in inotify_data's hashmap                         
  |    |    +-----------------------+                                                                     
  |    |                                                                                                  
  |    |--> move source from old inode_data to new inode_data                                             
  |    |                                                                                                  
  |    |    +--------------------------+                                                                  
  |    |--> | inode_data_realize_watch | add arg inode_data to inotify                                    
  |    |    +--------------------------+                                                                  
  |    |                                                                                                  
  |    |--> s->priority = arg priority                                                                    
  |    |                                                                                                  
  |    |    +---------------------+                                                                       
  |    +--> | event_gc_inode_data | garbage collect inode_data & inotify_data                             
  |         +---------------------+                                                                       
  |                                                                                                       
  |--> else if source type == signal                                                                      
  |    |                                                                                                  
  |    |--> s->priority = arg priority                                                                    
  |    |                                                                                                  
  |    |    +------------------------+                                                                    
  |    |--> | event_make_signal_data | ensure target signal is monitored by epoll                         
  |    |    +------------------------+                                                                    
  |    |    +--------------------------+                                                                  
  |    +--> | event_unmask_signal_data | unset the bit in mask                                            
  |         +--------------------------+                                                                  
  |                                                                                                       
  |--> else                                                                                               
  |    -                                                                                                  
  |    +--> s->priority = arg priority                                                                    
  |                                                                                                       
  |    +---------------------------------+                                                                
  |--> | event_source_pp_prioq_reshuffle | reshuffle source's 'pending' and 'prepare' queues if they exist
  |    +---------------------------------+                                                                
  |                                                                                                       
  +--> if source type == exit                                                                             
       |                                                                                                  
       |    +-----------------+                                                                           
       +--> | prioq_reshuffle | reshuffle target node in queue (exit)                                     
            +-----------------+                                                                           
```

```
src/libsystemd/sd-event/sd-event.c                                             
+-------------------------+                                                     
| event_make_inotify_data | : ensure inotify_data exists and registered to epoll
+-|-----------------------+                                                     
  |    +-------------+                                                          
  |--> | hashmap_get | given priority, get inotify_data from event hashmap      
  |    +-------------+                                                          
  |                                                                             
  |--> if got, return                                                           
  |                                                                             
  |    +---------------+                                                        
  |--> | inotify_init1 | get inotify fd                                         
  |    +---------------+                                                        
  |                                                                             
  |--> alloc inotify_data & set up                                              
  |                                                                             
  |    +--------------------+                                                   
  |--> | hashmap_ensure_put | insert inotify_data to event hashmap              
  |    +--------------------+                                                   
  |                                                                             
  |--> set up epoll_eventz                                                      
  |                                                                             
  |    +-----------+                                                            
  +--> | epoll_ctl | register fd to epoll                                       
       +-----------+                                                            
```

```
src/libsystemd/sd-event/sd-event.c                                         
+-----------------------+                                                   
| event_make_inode_data | : ensure inode_data in inotify_data's hashmap     
+-|---------------------+                                                   
  |                                                                         
  |--> set up key                                                           
  |                                                                         
  |    +-------------+                                                      
  |--> | hashmap_get | given key, get inode_data from inotify_data's hashmap
  |    +-------------+                                                      
  |                                                                         
  |--> if found, return                                                     
  |                                                                         
  |    +--------------------------+                                         
  |--> | hashmap_ensure_allocated | ensure inotify_data's hashmap exists    
  |    +--------------------------+                                         
  |                                                                         
  |--> alloc inode_data & set up                                            
  |                                                                         
  |    +-------------+                                                      
  +--> | hashmap_put | insert inode_data to inotify_data's hashmap          
       +-------------+                                                      
```

```
src/libsystemd/sd-event/sd-event.c                                     
+--------------------------+                                            
| inode_data_realize_watch | : add arg inode_data to inotify            
+-|------------------------+                                            
  |    +--------------------------+                                     
  |--> | hashmap_ensure_allocated | ensure inotify_data's hashmap exists
  |    +--------------------------+                                     
  |    +----------------------+                                         
  +--> | inotify_add_watch_fd | add arg inode_data to inotify           
       +----------------------+                                         
```

```
src/libsystemd/sd-event/sd-event.c                                               
+---------------------+                                                           
| event_gc_inode_data | : garbage collect inode_data & inotify_data               
+-|-------------------+                                                           
  |    +-----------------------+                                                  
  |--> | event_free_inode_data | remove inode_data from event/inotify, and free it
  |    +-----------------------+                                                  
  |    +-----------------------+                                                  
  +--> | event_gc_inotify_data | remove inotify_data from event, and free it      
       +-----------------------+                                                  
```

```
src/libsystemd/sd-bus/sd-bus.c                                                                         
+--------------------------+                                                                            
| bus_attach_inotify_event | : prepare source and register to epoll & set source's priority/description 
+-|------------------------+                                                                            
  |                                                                                                     
  |--> if bus has no inotify event source                                                               
  |    |                                                                                                
  |    |    +-----------------+                                                                         
  |    |--> | sd_event_add_io | prepare 'source' and add to arg 'event, register the source's io        
  |    |    +-----------------+                                                                         
  |    |    +------------------------------+                                                            
  |    |--> | sd_event_source_set_priority | s->priority = priority, reposition source in data structure
  |    |    +------------------------------+                                                            
  |    |    +---------------------------------+                                                         
  |    +--> | sd_event_source_set_description | duplicate description into source                       
  |         +---------------------------------+                                                         
  |                                                                                                     
  +--> else                                                                                             
       |                                                                                                
       |    +---------------------------+                                                               
       +--> | sd_event_source_set_io_fd | add new fd to epoll, remove the old one                       
            +---------------------------+                                                               
```

```
src/libsystemd/sd-bus/bus-socket.c                                                                                        
+----------------------------+                                                                                             
| bus_socket_process_opening | : given bus address, prepare socket & connect, register sources to epoll                    
+-|--------------------------+                                                                                             
  |    +-------------------+                                                                                               
  |--> | fd_wait_for_event | poll (wait for event)                                                                         
  |    +-------------------+                                                                                               
  |    +-----------------------+                                                                                           
  |--> | bus_socket_start_auth | (skip)                                                                                    
  |    +-----------------------+                                                                                           
  |    +------------------+                                                                                                
  +--> | bus_next_address | given bus address, spawn child if required, prepare socket & connect, register sources to epoll
       +------------------+                                                                                                
```

```
src/libsystemd/sd-bus/sd-bus.c                                                                                             
+------------------+                                                                                                        
| bus_next_address | : given bus address, spawn child if required, prepare socket & connect, register sources to epoll      
+-|----------------+                                                                                                        
  |    +--------------------------+                                                                                         
  |--> | bus_reset_parsed_address | reset bus structure                                                                     
  |    +--------------------------+                                                                                         
  |    +-------------------+                                                                                                
  +--> | bus_start_address | given bus address, spawn child if required, prepare socket & connect, register sources to epoll
       +-------------------+                                                                                                
```

```
src/libsystemd/sd-bus/sd-bus.c                                                                                                   
+-------------------+                                                                                                             
| bus_start_address | : given bus address, spawn child if required, prepare socket & connect, register sources to epoll           
+-|-----------------+                                                                                                             
  |                                                                                                                               
  +--> endless loop                                                                                                               
       |                                                                                                                          
       |    +------------------+                                                                                                  
       |--> | bus_close_io_fds | detach event (source->enabled = off, ref--), close input/output fd                               
       |    +------------------+                                                                                                  
       |    +----------------------+                                                                                              
       |--> | bus_close_inotify_fd | release inotify source, close fd                                                             
       |    +----------------------+                                                                                              
       |    +---------------+                                                                                                     
       |--> | bus_kill_exec | send 'terminate' signal to target task, wait till it exits                                          
       |    +---------------+                                                                                                     
       |                                                                                                                          
       |--> if exec path is provided                                                                                              
       |    |                                                                                                                     
       |    |    +-----------------+                                                                                              
       |    +--> | bus_socket_exec | fork and exec child, set up input/output buffer size (8m)                                    
       |         +-----------------+                                                                                              
       |                                                                                                                          
       |--> elif target is in another namespace (???)                                                                             
       |    |                                                                                                                     
       |    |    +------------------------------+                                                                                 
       |    +--> | bus_container_connect_socket | fork a task into target namespace, have it connect to bus's input fd
       |         +------------------------------+                                                                                 
       |                                                                                                                          
       |--> elif socket addr is specified                                                                                         
       |    |                                                                                                                     
       |    |    +--------------------+                                                                                           
       |    +--> | bus_socket_connect | create one socket for both input/output, connect, close inotify                           
       |         +--------------------+                                                                                           
       |                                                                                                                          
       |--> else, go to 'next'                                                                                                    
       |                                                                                                                          
       |    +----------------------+                                                                                              
       |--> | bus_attach_io_events | for input/output fd: prepare source and register to epoll & set source's priority/description
       |    +----------------------+                                                                                              
       |    +--------------------------+                                                                                          
       |--> | bus_attach_inotify_event | prepare source and register to epoll & set source's priority/description                 
       |    +--------------------------+                                                                                          
       |                                                                                                                          
       |--> return                                                                                                                
       |next:                                                                                                                     
       |    +------------------------+                                                                                            
       +--> | bus_parse_next_address | parse address, save to sock_addr in bus                                                    
            +------------------------+                                                                                            
```

```
src/libsystemd/sd-bus/bus-socket.c                                            
+-----------------+                                                            
| bus_socket_exec | : fork and exec child, set up input/output buffer size (8m)
+-|---------------+                                                            
  |    +------------+                                                          
  |--> | socketpair | create socket pair that connect to each other            
  |    +------------+                                                          
  |    +----------------+                                                      
  |--> | safe_fork_full | fork                                                 
  |    +----------------+                                                      
  |                                                                            
  |--> if I'm the child                                                        
  |    |                                                                       
  |    |    +--------+    +--------+                                           
  |    +--> | execvp | or | execvp |                                           
  |         +--------+    +--------+                                           
  |                                                                            
  |--> if I'm the parent, close socket[1]                                      
  |                                                                            
  |    +------------------+                                                    
  +--> | bus_socket_setup | set bus's input/output buffer to 8mb each          
       +------------------+                                                    
```

```
src/libsystemd/sd-bus/bus-container.c                                                                        
+------------------------------+                                                                              
| bus_container_connect_socket | : fork a task into target namespace, have it connect to bus's input fd       
+-|----------------------------+                                                                              
  |    +----------------+                                                                                     
  |--> | namespace_open |                                                                                     
  |    +----------------+                                                                                     
  |                                                                                                           
  |--> create socket for input fd                                                                             
  |                                                                                                           
  |--> output fd = input fd                                                                                   
  |                                                                                                           
  |    +------------------+                                                                                   
  |--> | bus_socket_setup | config send/receive buffer size                                                   
  |    +------------------+                                                                                   
  |    +------------+                                                                                         
  |--> | socketpair |                                                                                         
  |    +------------+                                                                                         
  |    +----------------+                                                                                     
  |--> | namespace_fork | parent forks child, child enters target ns & forks grandchild & waits for its finish
  |    +----------------+                                                                                     
  |                                                                                                           
  |--> if we are the grandchild                                                                               
  |    |                                                                                                      
  |    |--> close socker pair[0]                                                                              
  |    |                                                                                                      
  |    +--> connect bus's input fd and exit (why?)                                                            
  |                                                                                                           
  |    (parent reaches here)                                                                                  
  |                                                                                                           
  |--> close socket pair[1]                                                                                   
  |                                                                                                           
  |    +------------------------------+                                                                       
  |--> | wait_for_terminate_and_check | wait till grandchild finishes                                         
  |    +------------------------------+                                                                       
  |    +-----------------------+                                                                              
  +--> | bus_socket_start_auth | (skip)                                                                       
       +-----------------------+                                                                              
```

```
src/libsystemd/sd-bus/bus-container.c                                                                   
+----------------+                                                                                       
| namespace_fork | : parent forks child, child enters target ns & forks grandchild & waits for its finish
+-|--------------+                                                                                       
  |    +----------------+                                                                                
  |--> | safe_fork_full | fork, parent returns immediatly, child handles flags properly                  
  |    +----------------+                                                                                
  |                                                                                                      
  +--> if we are the forked child                                                                        
       |                                                                                                 
       |    +----------------+                                                                           
       +--> | safe_fork_full | fork, parent returns immediatly, child handles flags properly             
       |    +----------------+                                                                           
       |                                                                                                 
       |--> if we are the forked grandchild                                                              
       |    -                                                                                            
       |    +--> return pid thru arg, return                                                             
       |                                                                                                 
       |    (child reaches here)                                                                         
       |                                                                                                 
       |    +------------------------------+                                                             
       +--> | wait_for_terminate_and_check | grandchild exits                                            
       |    +------------------------------+                                                             
       |    +-------+                                                                                    
       +--> | _exit | child exits                                                                        
            +-------+                                                                                    
```

```
src/basic/process-util.c                                                         
+----------------+                                                                
| safe_fork_full | : fork, parent returns immediatly, child handles flags properly
+-|--------------+                                                                
  |                                                                               
  |--> given flags, determine signal mask for block                               
  |                                                                               
  |    +-------------+                                                            
  |--> | sigprocmask | block                                                      
  |    +-------------+                                                            
  |                                                                               
  |--> if flag has 'detach'                                                       
  |    |                                                                          
  |    |    +-------------------+                                                 
  |    |--> | is_reaper_process | check if we are reaper task (pid = 1)           
  |    |    +-------------------+                                                 
  |    |                                                                          
  |    +--> if we are not the reaper                                              
  |         |                                                                     
  |         |    +------+                                                         
  |         |--> | fork |                                                         
  |         |    +------+                                                         
  |         |                                                                     
  |         +--> if we are the parent return                                      
  |                                                                               
  |    +------+                                                                   
  |--> | fork |                                                                   
  |    +------+                                                                   
  |                                                                               
  |--> if we are child && the middle task                                         
  |    -                                                                          
  |    +--> exit to let the grandchild be reaped                                  
  |                                                                               
  |--> if we are the parent                                                       
  |    -                                                                          
  |    +--> if flag has 'wait'                                                    
  |    |    |                                                                     
  |    |    |    +------------------------------+                                 
  |    |    +--> | wait_for_terminate_and_check | wait for child to terminate     
  |    |         +------------------------------+                                 
  |    |                                                                          
  |    +--> return                                                                
  |                                                                               
  |--> (only child task reaches here)                                             
  |                                                                               
  +--> handle flags accordingly                                                   
```

```
src/libsystemd/sd-bus/sd-bus.c                                     
+------------------------+                                          
| bus_parse_next_address | : parse address, save to sock_addr in bus
+-|----------------------+                                          
  |    +--------------------------+                                 
  |--> | bus_reset_parsed_address | reset bus address               
  |    +--------------------------+                                 
  |                                                                 
  |--> get first address                                            
  |                                                                 
  +--> while address is still valid                                 
       |                                                            
       |--> if it starts with 'unix:'                               
       |    |                                                       
       |    |    +--------------------+                             
       |    |--> | parse_unix_address |                             
       |    |    +--------------------+                             
       |    +--> break                                              
       |                                                            
       |--> elif it starts with 'tcp:'                              
       |    |                                                       
       |    |    +-------------------+                              
       |    |--> | parse_tcp_address |                              
       |    |    +-------------------+                              
       |    +--> break                                              
       |                                                            
       |--> elif it starts with 'unixexec:'                         
       |    |                                                       
       |    |    +--------------------+                             
       |    |--> | parse_exec_address |                             
       |    |    +--------------------+                             
       |    +--> break                                              
       |                                                            
       |--> elif it starts with 'x-machine-unix:'                   
       |    |                                                       
       |    |    +------------------------------+                   
       |    |--> | parse_container_unix_address |                   
       |    |    +------------------------------+                   
       |    +--> break                                              
       |                                                            
       +--> advance to next address                                 
```

```
src/libsystemd/sd-bus/sd-bus.c                                                                             
+-----------------+                                                                                         
| process_running | : handle timeout, dispatch write queue, receive msg and process it                      
+-|---------------+                                                                                         
  |    +-----------------+                                                                                  
  |--> | process_timeout | remove 1st reply_callback from bus, prepare msg and set sender, call ->callback()
  |    +-----------------+                                                                                  
  |    +-----------------+                                                                                  
  |--> | dispatch_wqueue | for each msg in write queue: write to bus's output fd                            
  |    +-----------------+                                                                                  
  |    +----------------+                                                                                   
  |--> | dispatch_track | (skip)                                                                            
  |    +----------------+                                                                                   
  |    +-----------------+                                                                                  
  |--> | dispatch_rqueue | ensure rqueue has msg, get 1st msg and remove it from rqueue                     
  |    +-----------------+                                                                                  
  |    +-----------------+                                                                                  
  +--> | process_message | process callbacks of reply/filter/match, process method                          
       +-----------------+                                                                                  
```

```
src/libsystemd/sd-bus/sd-bus.c                                                                        
+-----------------+                                                                                    
| process_timeout | : remove 1st reply_callback from bus, prepare msg and set sender, call ->callback()
+-|---------------+                                                                                    
  |    +------------+                                                                                  
  |--> | prioq_peek | get first callback from bus prio queue                                           
  |    +------------+                                                                                  
  |    +---------------------------------+                                                             
  |--> | bus_message_new_synthetic_error | prepare msg, append information, set msg sender             
  |    +---------------------------------+                                                             
  |    +----------------------------+                                                                  
  |--> | bus_seal_synthetic_message | finish the msg build                                             
  |    +----------------------------+                                                                  
  |    +------------------------+                                                                      
  |--> | ordered_hashmap_remove | remove callback from bus                                             
  |    +------------------------+                                                                      
  |                                                                                                    
  |--> set up bus fields for callback                                                                  
  |                                                                                                    
  |--> call ->callback(), e.g.,                                                                        
  |    +----------------+                                                                              
  |    | hello_callback | read data from reply, change bus state from 'hello' to 'running'             
  |    +--------------------+                                                                          
  |    | add_match_callback | call ->install_callback()                                                
  |    +--------------------+                                                                          
  |                                                                                                    
  +--> reset those bus fields                                                                          
```

```
src/libsystemd/sd-bus/bus-message.c                                                 
+---------------------------------+                                                  
| bus_message_new_synthetic_error | : prepare msg, append information, set msg sender
+-|-------------------------------+                                                  
  |    +--------------------+                                                        
  |--> | sd_bus_message_new | prepare msg                                            
  |    +--------------------+                                                        
  |    +-----------------------------+                                               
  |--> | message_append_reply_cookie | append 'reply cookie' to msg                  
  |    +-----------------------------+                                               
  |                                                                                  
  |--> if bus has unique name                                                        
  |    |                                                                             
  |    |    +-----------------------------+                                          
  |    +--> | message_append_field_string | append 'unique name' to msg              
  |         +-----------------------------+                                          
  |    +-----------------------------+                                               
  |--> | message_append_field_string | append 'error name' to msg                    
  |    +-----------------------------+                                               
  |                                                                                  
  |--> if error has msg                                                              
  |    |                                                                             
  |    |    +-----------------------------+                                          
  |    +--> | message_append_field_string | append 'error msg' to msg                
  |         +-----------------------------+                                          
  |    +-------------------------------+                                             
  +--> | bus_message_set_sender_driver | set msg sender = "org.freedesktop.DBus"     
       +-------------------------------+                                             
```

```
src/libsystemd/sd-bus/sd-bus.c                                                                
+----------------+                                                                             
| hello_callback | : read data from reply, change bus state from 'hello' to 'running'          
+-|--------------+                                                                             
  |    +---------------------+                                                                 
  |--> | sd_bus_message_read | read data (type = string) from reply                            
  |    +---------------------+                                                                 
  |                                                                                            
  +--> if bus state == hello                                                                   
       |                                                                                       
       |    +---------------+                                                                  
       |--> | bus_set_state | set state = arg running                                          
       |    +---------------+                                                                  
       |    +-----------------------------+                                                    
       +--> | synthesize_connected_signal | prepare msg of signal type, insert to bus rqueue[0]
            +-----------------------------+                                                    
```

```
src/libsystemd/sd-bus/sd-bus.c                                                                   
+-----------------------------+                                                                   
| synthesize_connected_signal | : prepare msg of signal type, insert to bus rqueue[0]             
+-|---------------------------+                                                                   
  |    +---------------------------+                                                              
  |--> | sd_bus_message_new_signal | prepare msg of signal type, append path/interface/member info
  |    +---------------------------+                                                              
  |    +------------------------------+                                                           
  |--> | bus_message_set_sender_local | set msg sender = org.freedesktop.DBus.Local               
  |    +------------------------------+                                                           
  |    +----------------------------+                                                             
  |--> | bus_seal_synthetic_message | finish msg build                                            
  |    +----------------------------+                                                             
  |    +----------------------+                                                                   
  |--> | bus_rqueue_make_room | realloc bus rqueue (msg array)                                    
  |    +----------------------+                                                                   
  |                                                                                               
  +--> insert msg to rqueue[0]                                                                    
```

```
src/libsystemd/sd-bus/bus-message.c                                                            
+---------------------------+                                                                   
| sd_bus_message_new_signal | : prepare msg of signal type, append path/interface/member info   
+------------------------------+                                                                
| sd_bus_message_new_signal_to | : prepare msg of signal type, append path/interface/member info
+-|----------------------------+                                                                
  |    +--------------------+                                                                   
  |--> | sd_bus_message_new | prepare msg of signal type                                        
  |    +--------------------+                                                                   
  |    +-----------------------------+                                                          
  |--> | message_append_field_string | append 'path' to msg                                     
  |    +-----------------------------+                                                          
  |    +-----------------------------+                                                          
  |--> | message_append_field_string | append 'interface' to msg                                
  |    +-----------------------------+                                                          
  |    +-----------------------------+                                                          
  |--> | message_append_field_string | append 'member' to msg                                   
  |    +-----------------------------+                                                          
  |                                                                                             
  +--> if arg destination is provided                                                           
       |                                                                                        
       |    +-----------------------------+                                                     
       +--> | message_append_field_string | append 'destination' to msg                         
            +-----------------------------+                                                     
```

```
src/libsystemd/sd-bus/sd-bus.c                                                 
+-----------------+                                                             
| dispatch_wqueue | : for each msg in write queue: write to bus's output fd     
+-|---------------+                                                             
  |                                                                             
  +--> while wqueue_size > 0                                                    
       |                                                                        
       |    +-------------------+                                               
       |--> | bus_write_message | convert msg to iovec, write to bus's output fd
       |    +-------------------+                                               
       |                                                                        
       +--> remove the msg rom bus write queue                                  
```

```
src/libsystemd/sd-bus/sd-bus.c                                              
+-------------------+                                                        
| bus_write_message | : convert msg to iovec, write to bus's output fd       
+--------------------------+                                                 
| bus_socket_write_message | : convert msg to iovec, write to bus's output fd
+-|------------------------+                                                 
  |    +-------------------------+                                           
  |--> | bus_message_setup_iovec | convert msg to iovec                      
  |    +-------------------------+                                           
  |                                                                          
  +--> either writev() or sendmsg() to bus output fd                         
```

```
src/libsystemd/sd-bus/bus-socket.c               
+-------------------------+                       
| bus_message_setup_iovec | : convert msg to iovec
+-|-----------------------+                       
  |                                               
  |--> determine iovec buffer                     
  |                                               
  |    +--------------+                           
  |--> | append_iovec | append one vector         
  |    +--------------+                           
  |                                               
  +--> for each part in msg                       
       |                                          
       |    +-------------------+                 
       |--> | bus_body_part_map | perform mmap    
       |    +-------------------+                 
       |    +--------------+                      
       +--> | append_iovec | append one vector    
            +--------------+                      
```

```
src/libsystemd/sd-bus/sd-bus.c                                                                                     
+-----------------+                                                                                                 
| dispatch_rqueue | : ensure rqueue has msg, get 1st msg and remove it from rqueue                                  
+-|---------------+                                                                                                 
  |                                                                                                                 
  +--> endless loop                                                                                                 
       |                                                                                                            
       |--> if there's already something in rqueue                                                                  
       |    |                                                                                                       
       |    |--> get rqueue[0]                                                                                      
       |    |                                                                                                       
       |    |    +-----------------+                                                                                
       |    |--> | rqueue_drop_one | remove target msg from rqueue                                                  
       |    |    +-----------------+                                                                                
       |    +--> return                                                                                             
       |                                                                                                            
       |    +------------------+                                                                                    
       +--> | bus_read_message | prepare rbuffer, read data from bus's input fd, convert to msg and append to rqueue
            +------------------+                                                                                    
```

```
src/libsystemd/sd-bus/sd-bus.c                                                                                  
+------------------+                                                                                             
| bus_read_message | : prepare rbuffer, read data from bus's input fd, convert to msg and append to rqueue       
+-------------------------+                                                                                      
| bus_socket_read_message | : prepare rbuffer, read data from bus's input fd, convert to msg and append to rqueue
+-|-----------------------+                                                                                      
  |    +------------------------------+                                                                          
  |--> | bus_socket_read_message_need | determine how much size is needed                                        
  |    +------------------------------+                                                                          
  |                                                                                                              
  |--> if bus has enough size for msg                                                                            
  |    |                                                                                                         
  |    |    +-------------------------+                                                                          
  |    |--> | bus_socket_make_message | alloc msg & parse buffer, append to bus rqueue                           
  |    |    +-------------------------+                                                                          
  |    +--> return                                                                                               
  |                                                                                                              
  |    +---------+                                                                                               
  |--> | realloc | realloc bus rbuffer                                                                           
  |    +---------+                                                                                               
  |                                                                                                              
  |--> either readv() or recvmsg_safe()                                                                          
  |                                                                                                              
  |    +------------------------------+                                                                          
  |--> | bus_socket_read_message_need | determine how much size is needed                                        
  |    +------------------------------+                                                                          
  |                                                                                                              
  +--> if bus has enough size for msg                                                                            
       |                                                                                                         
       |    +-------------------------+                                                                          
       |--> | bus_socket_make_message | alloc msg & parse buffer, append to bus rqueue                           
       |    +-------------------------+                                                                          
       +--> return                                                                                               
```

```
src/libsystemd/sd-bus/bus-socket.c                                         
+-------------------------+                                                 
| bus_socket_make_message | : alloc msg & parse buffer, append to bus rqueue
+-|-----------------------+                                                 
  |    +----------------------+                                             
  |--> | bus_rqueue_make_room | realloc bus's rqueue                        
  |    +----------------------+                                             
  |    +-------------------------+                                          
  |--> | bus_message_from_malloc | alloc msg, and parse buffer              
  |    +-------------------------+                                          
  |                                                                         
  +--> add msg to the last position of bus rqueue                           
```

```
src/libsystemd/sd-bus/sd-bus.c                                                                         
+-----------------+                                                                                     
| process_message | : process callbacks of reply/filter/match, process method                           
+-|---------------+                                                                                     
  |    +---------------+                                                                                
  |--> | process_hello | simple check                                                                   
  |    +---------------+                                                                                
  |    +---------------+                                                                                
  |--> | process_reply | given msg cookie, remove target 'reply callback' from bus and call ->callback()
  |    +---------------+                                                                                
  |    +------------------+                                                                             
  |--> | process_fd_check | check fd                                                                    
  |    +------------------+                                                                             
  |    +----------------+                                                                               
  |--> | process_filter | for each filter callback in bus: call ->callback()                            
  |    +----------------+                                                                               
  |    +---------------+                                                                                
  |--> | process_match | recursively traverse children/sibling, if leaf: run ->callback()               
  |    +---------------+                                                                                
  |    +-----------------+                                                                              
  |--> | process_builtin | if necessary, prepare msg and reply                                          
  |    +-----------------+                                                                              
  |    +--------------------+                                                                           
  +--> | bus_process_object | handle method get/set/get_all/introspect/get_managed_objects              
       +--------------------+                                                                           
```

```
src/libsystemd/sd-bus/sd-bus.c                                                                    
+---------------+                                                                                  
| process_reply | : given msg cookie, remove target 'reply callback' from bus and call ->callback()
+-|-------------+                                                                                  
  |                                                                                                
  |--> given msg cookie (key), remove 'reply callback' from bus hashmap                            
  |                                                                                                
  |--> given 'reply callback', get outer 'slot'                                                    
  |                                                                                                
  |    +-----------------------+                                                                   
  |--> | sd_bus_message_rewind | (???)                                                             
  |    +-----------------------+                                                                   
  |                                                                                                
  |--> call ->callback()                                                                           
  |                                                                                                
  +--> slot ref--                                                                                  
```

```
src/libsystemd/sd-bus/sd-bus.c                                        
+----------------+                                                     
| process_filter | : for each filter callback in bus: call ->callback()
+-|--------------+                                                     
  |                                                                    
  +--> for each filter callback in bus                                 
       |                                                               
       |    +-----------------------+                                  
       |--> | sd_bus_message_rewind |                                  
       |    +-----------------------+                                  
       |                                                               
       +--> call ->callback()                                          
```

```
src/libsystemd/sd-bus/sd-bus.c                                                     
+---------------+                                                                   
| process_match | : recursively traverse children/sibling, if leaf: run ->callback()
+---------------+                                                                   
| bus_match_run | : recursively traverse children/sibling, if leaf: run ->callback()
+-|-------------+                                                                   
  |                                                                                 
  |--> switch node type                                                             
  |--> case root/value                                                              
  |    -    +---------------+                                                       
  |    +--> | bus_match_run | recursively apply to child                            
  |         +---------------+                                                       
  |--> case leaf                                                                    
  |    |--> run ->callback()                                                        
  |    |    +---------------+                                                       
  |    +--> | bus_match_run | recursively apply to sibling (next)                   
  |         +---------------+                                                       
  |--> case type/sender/destination/...                                             
  |    +--> get info from header                                                    
  |--> case args_xxx                                                                
  |    -    +---------------------+                                                 
  |    +--> | bus_message_get_arg |                                                 
  |         +---------------------+                                                 
  |                                                                                 
  |--> if node type is in hash table                                                
  |    |                                                                            
  |    |    +-------------+                                                         
  |    |--> | hashmap_get |                                                         
  |    |    +-------------+                                                         
  |    |    +---------------+                                                       
  |    +--> | bus_match_run | recursively apply to value found from hash table      
  |         +---------------+                                                       
  |                                                                                 
  |--> for each child of current node                                               
  |    |                                                                            
  |    |    +---------------+                                                       
  |    +--> | bus_match_run | recursively apply to current child                    
  |         +---------------+                                                       
  |    +---------------+                                                            
  +--> | bus_match_run | recursively apply to sibling (next)                        
       +---------------+                                                            
```

```
src/libsystemd/sd-bus/bus-objects.c                                                            
+--------------------+                                                                          
| bus_process_object | ： handle method get/set/get_all/introspect/get_managed_objects           
+-|------------------+                                                                          
  |                                                                                             
  |--> if it's broadcast msg, return (ignore)                                                   
  |                                                                                             
  |--> alloc buffer for prefix                                                                  
  |                                                                                             
  |    +---------------------+                                                                  
  |--> | object_find_and_run | handle method get/set/get_all/introspect/get_managed_objects     
  |    +---------------------+                                                                  
  |                                                                                             
  +--> for each prefix in msg path                                                              
       |                                                                                        
       |    +---------------------+                                                             
       +--> | object_find_and_run | handle method get/set/get_all/introspect/get_managed_objects
            +---------------------+                                                             
```

```
src/libsystemd/sd-bus/bus-objects.c                                                        
+---------------------+                                                                     
| object_find_and_run | : handle method get/set/get_all/introspect/get_managed_objects      
+-|-------------------+                                                                     
  |                                                                                         
  |--> given arg key, get node from bus's hashmap                                           
  |                                                                                         
  |    +--------------------+                                                               
  |--> | node_callbacks_run | for each callback in node, call ->callback()                  
  |    +--------------------+                                                               
  |                                                                                         
  |--> set up vtable key, get method from bus's another hashmap                             
  |                                                                                         
  |    +----------------------+                                                             
  |--> | method_callbacks_run | get user data and signature, call method.handler()          
  |    +----------------------+                                                             
  |                                                                                         
  |--> if interface == org.freedesktop.DBus.Properties                                      
  |    |                                                                                    
  |    |--> if member is 'get' or 'set'                                                     
  |    |    |                                                                               
  |    |    |    +---------------------+                                                    
  |    |    |--> | sd_bus_message_read | read interface/member from msg to set up vtable key
  |    |    |    +---------------------+                                                    
  |    |    |                                                                               
  |    |    |--> given key, get properties from bus's hashmap                               
  |    |    |                                                                               
  |    |    |    +--------------------------------+                                         
  |    |    +--> | property_get_set_callbacks_run | prepare msg of 'method return' (method is get or set), send msg out
  |    |         +--------------------------------+                                         
  |    |                                                                                    
  |    +--> elif member is 'get all'                                                        
  |         |                                                                               
  |         |    +---------------------+                                                    
  |         |--> | sd_bus_message_read | read interface from msg                            
  |         |    +---------------------+                                                    
  |         |    +--------------------------------+                                         
  |         +--> | property_get_all_callbacks_run | prepare msg of type 'method return' (method is get_all), append all properties, send msg out
  |              +--------------------------------+                                         
  |                                                                                         
  |--> elif method == 'introspect'                                                          
  |    |                                                                                    
  |    |    +--------------------+                                                          
  |    +--> | process_introspect | prepare msg of 'method return', append introspect info to msg, send out
  |         +--------------------+                                                          
  |                                                                                         
  +--> elif method == 'get managed objects'                                                 
       |                                                                                    
       |    +-----------------------------+                                                 
       +--> | process_get_managed_objects | prepare msg of 'method return', add all vtables of path/prefix, send msg out
            +-----------------------------+                                                 
```

```
src/libsystemd/sd-bus/bus-objects.c                                              
+----------------------+                                                          
| method_callbacks_run | : get user data and signature, call method.handler()     
+-|--------------------+                                                          
  |    +--------------------------+                                               
  |--> | node_vtable_get_userdata | call ->find() to get user data                
  |    +--------------------------+                                               
  |    +--------------------------------+                                         
  |--> | vtable_method_convert_userdata | += offset                               
  |    +--------------------------------+                                         
  |    +------------------------------+                                           
  |--> | sd_bus_message_get_signature | determine container, get signature from it
  |    +------------------------------+                                           
  |                                                                               
  +--> call method.handler()                                                      
```

```
src/libsystemd/sd-bus/bus-objects.c                                                                              
+--------------------------------+                                                                                
| property_get_set_callbacks_run | : prepare msg of 'method return' (method is get or set), send msg out          
+-|------------------------------+                                                                                
  |    +------------------------------+                                                                           
  |--> | vtable_property_get_userdata | call ->find() to get user data                                            
  |    +------------------------------+                                                                           
  |                                                                                                               
  |--> given vtable parent, get outer slot                                                                        
  |                                                                                                               
  |    +----------------------------------+                                                                       
  |--> | sd_bus_message_new_method_return | prepare msg of 'method return'                                        
  |    +----------------------------------+                                                                       
  |                                                                                                               
  |--> if arg is_get == 1                                                                                         
  |    |                                                                                                          
  |    |    +-------------------------------+                                                                     
  |    |--> | sd_bus_message_open_container | ensure contents in signature, set up container for it and add to msg
  |    |    +-------------------------------+                                                                     
  |    |    +---------------------+                                                                               
  |    |--> | invoke_property_get | get property and append to msg                                                
  |    |    +---------------------+                                                                               
  |    |    +--------------------------------+                                                                    
  |    +--> | sd_bus_message_close_container | close container and free signature                                 
  |         +--------------------------------+                                                                    
  |                                                                                                               
  |--> else                                                                                                       
  |    |                                                                                                          
  |    |    +--------------------------+                                                                          
  |    |--> | sd_bus_message_peek_type | peek type (and signature)                                                
  |    |    +--------------------------+                                                                          
  |    |    +--------------------------------+                                                                    
  |    |--> | sd_bus_message_enter_container | enter container, set up another container in msg                   
  |    |    +--------------------------------+                                                                    
  |    |    +---------------------+                                                                               
  |    |--> | invoke_property_set | read msg and return user data (where's the set action???)                     
  |    |    +---------------------+                                                                               
  |    |    +-------------------------------+                                                                     
  |    +--> | sd_bus_message_exit_container | free last container in msg                                          
  |         +-------------------------------+                                                                     
  |    +-------------+                                                                                            
  +--> | sd_bus_send | seal, send msg out or append to wqueue                                                     
       +-------------+                                                                                            
```

```
src/libsystemd/sd-bus/bus-objects.c                                       
+----------------------------------+                                       
| sd_bus_message_new_method_return | : prepare msg of 'method return'      
+-------------------+--------------+                                       
| message_new_reply | : prepare msg of arg type                            
+-|-----------------+                                                      
  |    +--------------------+                                              
  |--> | sd_bus_message_new | prepare msg of arg type (e.g., method_return)
  |    +--------------------+                                              
  |    +-----------------------------+                                     
  |--> | message_append_reply_cookie | append 'reply cookie' to msg        
  |    +-----------------------------+                                     
  |                                                                        
  +--> if call sender is set                                               
       |                                                                   
       |    +-----------------------------+                                
       +--> | message_append_field_string | append 'destination' to msg    
            +-----------------------------+                                
```

```
src/libsystemd/sd-bus/bus-message.c                                                                    
+-------------------------------+                                                                       
| sd_bus_message_open_container | : ensure contents in signature, set up container for it and add to msg
+-|-----------------------------+                                                                       
  |                                                                                                     
  |--> increase container array size                                                                    
  |                                                                                                     
  |    +----------------------------+                                                                   
  |--> | message_get_last_container | get last container from msg                                       
  |    +----------------------------+                                                                   
  |                                                                                                     
  |--> if type == array                                                                                 
  |    |                                                                                                
  |    |    +------------------------+                                                                  
  |    +--> | bus_message_open_array | ensure signature contains contents                               
  |         +------------------------+                                                                  
  |                                                                                                     
  |--> elif itype == variant                                                                            
  |    |                                                                                                
  |    |    +--------------------------+                                                                
  |    +--> | bus_message_open_variant | extend msg body and copy contents to it                        
  |         +--------------------------+                                                                
  |                                                                                                     
  |--> elif itype == struct                                                                             
  |    |                                                                                                
  |    |    +-------------------------+                                                                 
  |    +--> | bus_message_open_struct | ensure signature contains contents                              
  |         +-------------------------+                                                                 
  |                                                                                                     
  |--> elif itype == dict entry                                                                         
  |    |                                                                                                
  |    |    +-----------------------------+                                                             
  |    +--> | bus_message_open_dict_entry | (expect that signature contains contents already)           
  |         +-----------------------------+                                                             
  |                                                                                                     
  +--> prepare a container of the contents and add to msg (end of container array)                      
```

```
src/libsystemd/sd-bus/bus-message.c                              
+------------------------+                                        
| bus_message_open_array | : ensure signature contains contents   
+-|----------------------+                                        
  |                                                               
  |--> if signature contains contents already, update end index   
  |                                                               
  |                                                               
  +--> else                                                       
  |    |                                                          
  |    |    +-----------+                                         
  |    +--> | strextend | extend signature and copy contents to it
  |         +-----------+                                         
  |    +---------------------+                                    
  |    | message_extend_body | extend msg body                    
  |    +---------------------+                                    
  |                                                               
  +--> if contaoiner enclosing != array                           
       -                                                          
       +--> save end index in container                           
```

```
src/libsystemd/sd-bus/bus-message.c                                  
+--------------------------+                                          
| bus_message_open_variant | : extend msg body and copy contents to it
+-|------------------------+                                          
  |    +--------------------+                                         
  |--> | message_extend_body|                                         
  |    +--------------------+                                         
  |    +--------+                                                     
  +--> | memcpy | copy contents to msg body                           
       +--------+                                                     
```

```
src/libsystemd/sd-bus/bus-message.c                             
+-------------------------+                                      
| bus_message_open_struct | : ensure signature contains contents 
+-|-----------------------+                                      
  |                                                              
  |--> if signature already contains contents, update end index  
  |                                                              
  |--> else                                                      
  |    |                                                         
  |    |    +-----------+                                        
  |    +--> | strextend | extend signatue and copy contents to it
  |         +-----------+                                        
  |                                                              
  +--> if container enclosing != array                           
       -                                                         
       +--> save end index in container                          
```

```
src/libsystemd/sd-bus/bus-message.c                            
+--------------------------+                                    
| sd_bus_message_peek_type | : peek type (and signature)        
+-|------------------------+                                    
  |    +----------------------------+                           
  +--> | message_get_last_container | get msg's last container  
  |    +----------------------------+                           
  |                                                             
  |--> peek signature, if it's basic type, return type to caller
  |                                                             
  |--> peek signature, if it's array                            
  |    -                                                        
  |    +--> peek further, return type/content to caller         
  |                                                             
  |--> peek signature, if it's part of struct or dict           
  |    -                                                        
  |    +--> peek further, return type/content to caller         
  |                                                             
  +--> peek signature, if it's variant                          
       -                                                        
       +--> peek msg body, return type/content to caller        
```

```
src/libsystemd/sd-bus/bus-objects.c                                                 
+--------------------------------+                                                   
| sd_bus_message_enter_container | : enter container, set up another container in msg
+-|------------------------------+                                                   
  |    +----------------------------+                                                
  |--> | message_get_last_container |                                                
  |    +----------------------------+                                                
  |                                                                                  
  |--> if type == array                                                              
  |    |                                                                             
  |    |    +-------------------------+                                              
  |    +--> | bus_message_enter_array | peek msg body to get rindex and array size   
  |         +-------------------------+                                              
  |                                                                                  
  |--> elif type == variant                                                          
  |    |                                                                             
  |    |    +---------------------------+                                            
  |    +--> | bus_message_enter_variant | peek msg body to get rindex                
  |         +---------------------------+                                            
  |                                                                                  
  |--> elif type == struct                                                           
  |    |                                                                             
  |    |    +--------------------------+                                             
  |    +--> | bus_message_enter_struct | peek msg body to get rindex                 
  |         +--------------------------+                                             
  |                                                                                  
  |--> elif type == dict entry                                                       
  |    |                                                                             
  |    |    +------------------------------+                                         
  |    +--> | bus_message_enter_dict_entry | peek msg body to get rindex             
  |         +------------------------------+                                         
  |                                                                                  
  +--> set up container in msg (end of container array)                              
```

```
src/libsystemd/sd-bus/sd-bus.c                                                 
+-------------+                                                                 
| sd_bus_send | : seal, send msg out or append to wqueue                        
+-|-----------+                                                                 
  |    +------------------+                                                     
  |--> | bus_seal_message |                                                     
  |    +------------------+                                                     
  |    +-----------------------+                                                
  |--> | bus_remarshal_message | if necessary, remarshal msg                    
  |    +-----------------------+                                                
  |                                                                             
  |--> if no reply is required, return                                          
  |                                                                             
  |--> if bus state is hello or running && wqueue is empty                      
  |    |                                                                        
  |    |    +-------------------+                                               
  |    +--> | bus_write_message | convert msg to iovec, write to bus's output fd
  |         +-------------------+                                               
  |                                                                             
  +--> else                                                                     
       -                                                                        
       +--> append msg to wqueue                                                
```

```
src/libsystemd/sd-bus/sd-bus.c
+-----------------------+                                                  
| bus_remarshal_message | : if necessary, remarshal msg                    
+-|---------------------+                                                  
  |                                                                        
  |--> check if wee need remarshal                                         
  |                                                                        
  +--> if we do                                                            
       |                                                                   
       |    +-----------------------+                                      
       +--> | bus_message_remarshal | alloc new msg, copy data from old one
            +-----------------------+                                      
```

```
src/libsystemd/sd-bus/bus-message.c                                                                                         
+-----------------------+                                                                                                    
| bus_message_remarshal | : alloc new msg, copy data from old one                                                            
+-|---------------------+                                                                                                    
  |                                                                                                                          
  |--> switch header type                                                                                                    
  |--> case signal                                                                                                           
  |    -    +---------------------------+                                                                                    
  |    +--> | sd_bus_message_new_signal | prepare msg of signal type, append path/interface/member info                      
  |         +---------------------------+                                                                                    
  |--> case method_call                                                                                                      
  |    -    +--------------------------------+                                                                               
  |    +--> | sd_bus_message_new_method_call | prepare msg of method_call type, append path/member/interface/destination info
  |         +--------------------------------+                                                                               
  |--> case method_return/method_error                                                                                       
  |    |    +--------------------+                                                                                           
  |    |--> | sd_bus_message_new | prepare msg                                                                               
  |    |    +--------------------+                                                                                           
  |    |    +-----------------------------+                                                                                  
  |    |--> | message_append_reply_cookie | append reply_cookie to msg                                                       
  |    |    +-----------------------------+                                                                                  
  |    |                                                                                                                     
  |    +--> if it's method_error                                                                                             
  |         |                                                                                                                
  |         |    +-----------------------------+                                                                             
  |         +--> | message_append_field_string | append error name to msg                                                    
  |              +-----------------------------+                                                                             
  |                                                                                                                          
  |--> if orig msg has destination                                                                                           
  |    |                                                                                                                     
  |    |    +-----------------------------+                                                                                  
  |    +--> | message_append_field_string | append destination to msg                                                        
  |         +-----------------------------+                                                                                  
  |                                                                                                                          
  |--> if orig msg has sender                                                                                                
  |    |                                                                                                                     
  |    |    +-----------------------------+                                                                                  
  |    +--> | message_append_field_string |                                                                                  
  |         +-----------------------------+                                                                                  
  |    +---------------------+                                                                                               
  |--> | sd_bus_message_copy | copy data from old msg to new one                                                             
  |    +---------------------+                                                                                               
  |    +---------------------+                                                                                               
  +--> | sd_bus_message_seal |                                                                                               
       +---------------------+                                                                                               
```

```
src/libsystemd/sd-bus/bus-objects.c                                                                                            
+--------------------------------+                                                                                              
| property_get_all_callbacks_run | ：prepare msg of type 'method return' (method is get_all), append all properties, send msg out
+-|------------------------------+                                                                                              
  |    +----------------------------------+                                                                                     
  |--> | sd_bus_message_new_method_return | prepare msg (type is 'method return')                                               
  |    +----------------------------------+                                                                                     
  |    +-------------------------------+                                                                                        
  |--> | sd_bus_message_open_container | given arg type, interpret container, extend msg and append container                   
  |    +-------------------------------+                                                                                        
  |                                                                                                                             
  |--> for each vtable                                                                                                          
  |    |                                                                                                                        
  |    |    +--------------------------+                                                                                        
  |    |--> | node_vtable_get_userdata | call ->find() to get user data                                                         
  |    |    +--------------------------+                                                                                        
  |    |                                                                                                                        
  |    |--> if found no obj, continue                                                                                           
  |    |                                                                                                                        
  |    |--> found_obj = true                                                                                                    
  |    |                                                                                                                        
  |    |--> if found no iface, continue                                                                                         
  |    |                                                                                                                        
  |    |--> found_iface = true                                                                                                  
  |    |                                                                                                                        
  |    |    +------------------------------+                                                                                    
  |    +--> | vtable_append_all_properties | given node, append all properties (name/value) to msg                              
  |         +------------------------------+                                                                                    
  |                                                                                                                             
  |--> if found_obj == false, return                                                                                            
  |                                                                                                                             
  |--> if found_iface == false, return error                                                                                    
  |                                                                                                                             
  |    +--------------------------------+                                                                                       
  |--> | sd_bus_message_close_container |                                                                                       
  |    +--------------------------------+                                                                                       
  |    +-------------+                                                                                                          
  +--> | sd_bus_send | seal, send msg out or append to wqueue                                                                   
       +-------------+                                                                                                          
```

```
src/libsystemd/sd-bus/bus-objects.c                                                    
+------------------------------+                                                        
| vtable_append_all_properties | : given node, append all properties (name/value) to msg
+-|----------------------------+                                                        
  |                                                                                     
  +--> for each vtable in node                                                          
       |                                                                                
       |    +----------------------------+                                              
       +--> | vtable_append_one_property | append property name/value to msg            
            +----------------------------+                                              
```

```
src/libsystemd/sd-bus/bus-objects.c                                                                                      
+----------------------------+                                                                                            
| vtable_append_one_property | : append property name/value to msg                                                        
+-|--------------------------+                                                                                            
  |    +-------------------------------+                                                                                  
  |--> | sd_bus_message_open_container | given arg type (dict entry), interpret container, extend msg and append container
  |    +-------------------------------+                                                                                  
  |    +-----------------------+                                                                                          
  +--> | sd_bus_message_append | append property (type = string) to msg                                                   
  |    +-----------------------+                                                                                          
  |    +-------------------------------+                                                                                  
  |--> | sd_bus_message_open_container | given arg type (variant), interpret container, extend msg and append container   
  |    +-------------------------------+                                                                                  
  |                                                                                                                       
  |--> given node, get outer slot                                                                                         
  |                                                                                                                       
  |    +---------------------+                                                                                            
  |--> | invoke_property_get | append property to msg                                                                     
  |    +---------------------+                                                                                            
  |    +--------------------------------+                                                                                 
  |--> | sd_bus_message_close_container | close container and free signature                                              
  |    +--------------------------------+                                                                                 
  |    +--------------------------------+                                                                                 
  +--> | sd_bus_message_close_container | (bc we opened the container twice)                                              
       +--------------------------------+                                                                                 
```

```
src/libsystemd/sd-bus/bus-objects.c                          
+---------------------+                                       
| invoke_property_get | : append property to msg              
+-|-------------------+                                       
  |                                                           
  |--> if property.get exists                                 
  |    |                                                      
  |    |--> call property.get()                               
  |    +--> return                                            
  |                                                           
  |--> get user data                                          
  |                                                           
  |    +-----------------------------+                        
  +--> | sd_bus_message_append_basic | append user data to msg
       +-----------------------------+                        
```

```
src/libsystemd/sd-bus/bus-objects.c                                                            
+--------------------+                                                                          
| process_introspect | : prepare msg of 'method return', append introspect info to msg, send out
+-|------------------+                                                                          
  |    +-----------------+                                                                      
  |--> | introspect_path | write interfacess (and method/property/signal), write each child node
  |    +-----------------+                                                                      
  |    +----------------------------------+                                                     
  |--> | sd_bus_message_new_method_return | prepare msg of 'method return'                      
  |    +----------------------------------+                                                     
  |    +-----------------------+                                                                
  |--> | sd_bus_message_append | append introspect info to msg                                  
  |    +-----------------------+                                                                
  |    +-------------+                                                                          
  +--> | sd_bus_send | seal, send msg out or append to wqueue                                   
       +-------------+                                                                          
```

```
src/libsystemd/sd-bus/bus-objects.c                                                                          
+-----------------+                                                                                           
| introspect_path | : write interfacess (and method/property/signal), write each child node                   
+-|---------------+                                                                                           
  |                                                                                                           
  |--> if arg node isn't provided, get one ourselves                                                          
  |                                                                                                           
  |    +-----------------+                                                                                    
  |--> | get_child_nodes | create a set, add subtree to it                                                    
  |    +-----------------+                                                                                    
  |    +------------------+                                                                                   
  |--> | introspect_begin | init mem stream, write doctype to fd                                              
  |    +------------------+                                                                                   
  |    +-------------------------------------+                                                                
  |--> | introspect_write_default_interfaces | write default interfaces (peer/introspectable/properties) to fd
  |    +-------------------------------------+                                                                
  |                                                                                                           
  |--> for each vtable in node                                                                                
  |    |                                                                                                      
  |    |    +--------------------------+                                                                      
  |    |--> | node_vtable_get_userdata | get user data                                                        
  |    |    +--------------------------+                                                                      
  |    |    +----------------------------+                                                                    
  |    +--> | introspect_write_interface | write interface and its method/property/signal to fd               
  |         +----------------------------+                                                                    
  |    +------------------------------+                                                                       
  |--> | introspect_write_child_nodes | write child nodes to fd                                               
  |    +------------------------------+                                                                       
  |    +-------------------+                                                                                  
  +--> | introspect_finish | write </node> to fd, fininalize mem stream                                       
       +-------------------+                                                                                  
```

```
src/libsystemd/sd-bus/bus-objects.c                      
+-----------------+                                       
| get_child_nodes | : create a set, add subtree to it     
+-|---------------+                                       
  |    +-----------------+                                
  |--> | ordered_set_new | ceate a hashmap                
  |    +-----------------+                                
  |    +--------------------+                             
  +--> | add_subtree_to_set | add the whole subtree to set
       +--------------------+                             
```

```
src/libsystemd/sd-bus/bus-objects.c                 
+--------------------+                               
| add_subtree_to_set | : add the whole subtree to set
+-|------------------+                               
  |    +-----------------------+                     
  |--> | add_enumerated_to_set | add node to set     
  |    +-----------------------+                     
  |                                                  
  +--> for each child of node                        
       |                                             
       |    +---------------------+                  
       +--> | ordered_set_consume | add to set       
            +---------------------+                  
```

```
src/libsystemd/sd-bus/bus-introspect.c                                              
+----------------------------+                                                       
| introspect_write_interface | : write interface and its method/property/signal to fd
+-|--------------------------+                                                       
  |    +--------------------+                                                        
  |--> | set_interface_name | write interface name to fd                             
  |    +--------------------+                                                        
  |                                                                                  
  +--> for each vtable                                                               
       -                                                                             
       +--> switch type                                                              
            case start                                                               
            +--> write annotation to fd                                              
            case method                                                              
            +--> write method to fd                                                  
            case property                                                            
            +--> write property to fd                                                
            case signal                                                              
            +--> write signal to fd                                                  
```

```
src/libsystemd/sd-bus/bus-introspect.c                           
+------------------------------+                                  
| introspect_write_child_nodes | : write child nodes to fd        
+-|----------------------------+                                  
  |                                                               
  +--> while set isn't empty                                      
       |                                                          
       |    +-------------------------+                           
       |--> | ordered_set_steal_first | remove first node from set
       |    +-------------------------+                           
       |    +------------------------+                            
       |--> | object_path_startswith | skip prefix and get data   
       |    +------------------------+                            
       |                                                          
       |--> if got                                                
       |    -                                                     
       |    +--> write node info to fd                            
       |                                                          
       +--> free node                                             
```

```
src/libsystemd/sd-bus/bus-objects.c                                                                              
+-----------------------------+                                                                                   
| process_get_managed_objects | : prepare msg of 'method return', add all vtables of path/prefix, send msg out    
+-|---------------------------+                                                                                   
  |    +-----------------+                                                                                        
  |--> | get_child_nodes | create a set, add subtree to it                                                        
  |    +-----------------+                                                                                        
  |    +----------------------------------+                                                                       
  |--> | sd_bus_message_new_method_return | prepare msg of 'method return'                                        
  |    +----------------------------------+                                                                       
  |    +-------------------------------+                                                                          
  |--> | sd_bus_message_open_container | ensure contents in signature, set up container for it and add to msg     
  |    +-------------------------------+                                                                          
  |                                                                                                               
  |--> for each item in set (?)                                                                                   
  |    |                                                                                                          
  |    |    +---------------------------------------------+                                                       
  |    +--> | object_manager_serialize_path_and_fallbacks | add all vtables registered for the path and its prefix
  |         +---------------------------------------------+                                                       
  |    +--------------------------------+                                                                         
  |--> | sd_bus_message_close_container | close container and free signature                                      
  |    +--------------------------------+                                                                         
  |    +-------------+                                                                                            
  +--> | sd_bus_send | seal, send msg out or append to wqueue                                                     
       +-------------+                                                                                            
```

```
src/libsystemd/sd-bus/bus-objects.c                                                                                           
+---------------------------------------------+                                                                                
| object_manager_serialize_path_and_fallbacks | : add all vtables registered for the path and its prefix                       
+-|-------------------------------------------+                                                                                
  |    +-------------------------------+                                                                                       
  |--> | object_manager_serialize_path | get node of prefix, for each vtable in node: append interfaces and properties to reply
  |    +-------------------------------+                                                                                       
  |                                                                                                                            
  |--> alloc buffer for prefix iterator                                                                                        
  |                                                                                                                            
  +--> for each possible prefix                                                                                                
       |                                                                                                                       
       |    +-------------------------------+                                                                                  
       +--> | object_manager_serialize_path | this time, add fallback vtables for any of the prefix                            
            +-------------------------------+                                                                                  
```

```
src/libsystemd/sd-bus/bus-objects.c                                                                                      
+-------------------------------+                                                                                         
| object_manager_serialize_path | : get node of prefix, for each vtable in node: append interfaces and properties to reply
+-|-----------------------------+                                                                                         
  |    +-------------+                                                                                                    
  |--> | hashmap_get | given key (prefix), get node from hashmap                                                          
  |    +-------------+                                                                                                    
  |                                                                                                                       
  +--> for each vtable in node                                                                                            
       |                                                                                                                  
       |    +--------------------------+                                                                                  
       |--> | node_vtable_get_userdata | get user data                                                                    
       |    +--------------------------+                                                                                  
       |                                                                                                                  
       |--> if we haven't found something                                                                                 
       |    |                                                                                                             
       |    |    +-------------------------------+                                                                        
       |    |--> | sd_bus_message_open_container | ensure contents in signature, set up container for it and add to msg   
       |    |    +-------------------------------+                                                                        
       |    |    +-----------------------+                                                                                
       |    |--> | sd_bus_message_append | append path to reply                                                           
       |    |    +-----------------------+                                                                                
       |    |    +-------------------------------+                                                                        
       |    |--> | sd_bus_message_open_container |                                                                        
       |    |    +-------------------------------+                                                                        
       |    |    +-----------------------+                                                                                
       |    |--> | sd_bus_message_append | append org.freedesktop.DBus.Peer to reply                                      
       |    |    +-----------------------+                                                                                
       |    |    +-----------------------+                                                                                
       |    |--> | sd_bus_message_append | append org.freedesktop.DBus.Introspectable to reply                            
       |    |    +-----------------------+                                                                                
       |    |    +-----------------------+                                                                                
       |    |--> | sd_bus_message_append | append org.freedesktop.DBus.Properties to reply                                
       |    |    +-----------------------+                                                                                
       |    |                                                                                                             
       |    +--> found_something = true                                                                                   
       |                                                                                                                  
       |    +-----------------------+                                                                                     
       |--> | sd_bus_message_append | append interface to reply                                                           
       |    +-----------------------+                                                                                     
       |    +------------------------------+                                                                              
       +--> | vtable_append_all_properties | given node, append all properties (name/value) to msg                        
            +------------------------------+                                                                              
```

```
src/libsystemd/sd-bus/sd-bus.c                                                          
+-----------------+                                                                      
| process_closing | : prepare msg of 'disconnected', close bus and exit                  
+-|---------------+                                                                      
  |                                                                                      
  |--> prepare msg of 'disconnected'                                                     
  |                                                                                      
  |    +------------------------------+                                                  
  |--> | bus_message_set_sender_local | set sender = org.freedesktop.DBus.Local          
  |    +------------------------------+                                                  
  |    +----------------------------+                                                    
  |--> | bus_seal_synthetic_message | set timestamp and seal msg                         
  |    +----------------------------+                                                    
  |    +--------------+                                                                  
  |--> | sd_bus_close | release sources/event/fd of bus
  |    +--------------+                                                                  
  |    +----------------+                                                                
  |--> | process_filter | for each filter callback in bus: call ->callback()             
  |    +----------------+                                                                
  |    +---------------+                                                                 
  |--> | process_match | recursively traverse children/sibling, if leaf: run ->callback()
  |    +---------------+                                                                 
  |    +--------------+                                                                  
  +--> | bus_exit_now | bus exits
       +--------------+                                                                  
```

```
src/libsystemd/sd-bus/sd-bus.c                                                    
+--------------+                                                                   
| sd_bus_close | : release sources/event/fd of bus                                 
+-|------------+                                                                   
  |    +---------------+                                                           
  |--> | bus_kill_exec | kill task of bus                                          
  |    +---------------+                                                           
  |    +---------------+                                                           
  |--> | bus_set_state | set bus state = closed                                    
  |    +---------------+                                                           
  |    +---------------------+                                                     
  |--> | sd_bus_detach_event | unref sources, unref event                          
  |    +---------------------+                                                     
  |    +------------------+                                                        
  |--> | bus_reset_queues | for bus rqueue/wqueue: empty messages and release queue
  |    +------------------+                                                        
  |    +------------------+                                                        
  |--> | bus_close_io_fds | close bus input/output fd                              
  |    +------------------+                                                        
  |    +----------------------+                                                    
  +--> | bus_close_inotify_fd | close inotify fd                                   
       +----------------------+                                                    
```

```
src/libsystemd/sd-bus/sd-bus.c                                                       
+---------------------+                                                               
| sd_bus_detach_event | : unref sources, unref event                                  
+-|-------------------+                                                               
  |    +----------------------+                                                       
  |--> | bus_detach_io_events | for io input/output sources: set enabled = off & ref--
  |    +----------------------+                                                       
  |                                                                                   
  +--> for inotify/time/quite event source: set enabled = off & ref--                 
  |                                                                                   
  |    +----------------+                                                             
  +--> | sd_event_unref | event ref--, release if nobody needs it                     
       +----------------+                                                             
```

```
src/libsystemd/sd-event/sd-event.c                                                                
+----------------+                                                                                 
| sd_event_unref | : event ref--, release if nobody needs it                                       
+-|--------------+                                                                                 
  |                                                                                                
  |--> event ref--                                                                                 
  |                                                                                                
  +--> if ref > 0, return null                                                                     
       |                                                                                           
       |    +------------+                                                                         
       +--> | event_free | disconnect all sources from event, free other resources and event itself
            +------------+                                                                         
```

```
src/libsystemd/sd-event/sd-event.c                                                                  
+------------+                                                                                       
| event_free | : disconnect all sources from event, free other resources and event itself            
+-|----------+                                                                                       
  |                                                                                                  
  |--> for each source in event                                                                      
  |    |                                                                                             
  |    |    +-------------------+                                                                    
  |    +--> | source_disconnect | given source type, disconnect source properly, remove it from event
  |         +-------------------+                                                                    
  |                                                                                                  
  |--> close epoll/watchdog fd                                                                       
  |                                                                                                  
  |--> free clock data of realtime/boottime/monotonic/realtime_alarm/boottime_alarm                  
  |                                                                                                  
  |--> free event's pending/prepare/exit queue                                                       
  |                                                                                                  
  |--> free signal_sources and other hash maps                                                       
  |                                                                                                  
  +--> blabla, eventually free event itself                                                          
```

```
src/libsystemd/sd-event/sd-event.c                                                         
+-------------------+                                                                       
| source_disconnect | : given source type, disconnect source properly, remove it from event 
+-|-----------------+                                                                       
  |                                                                                         
  |--> switch source type                                                                   
  |--> case io                                                                              
  |    -    +----------------------+                                                        
  |    +--> | source_io_unregister | unregister from epoll                                  
  |         +----------------------+                                                        
  |--> case realtime/boottime/monotonic/realtime_alarm/boottime_alarm                       
  |    -    +--------------------------------+                                              
  |    +--> | event_source_time_prioq_remove | remove source from earliest/latest prio queue
  |         +--------------------------------+                                              
  |--> case signal                                                                          
  |    -    +----------------------+                                                        
  |    +--> | event_gc_signal_data | unset signal bit from all three possible masks         
  |         +----------------------+                                                        
  |--> case child                                                                           
  |    |    +----------------+                                                              
  |    |--> | hashmap_remove | remove source from event                                     
  |    |    +----------------+                                                              
  |    |    +-------------------------------+                                               
  |    +--> | source_child_pidfd_unregister | unregister source from epoll                  
  |         +-------------------------------+                                               
  |--> case post                                                                            
  |    -    +------------+                                                                  
  |    +--> | set_remove | remove source from event                                         
  |         +------------+                                                                  
  |--> case exit                                                                            
  |    -    +--------------+                                                                
  |    +--> | prioq_remove | remove source from event                                       
  |         +--------------+                                                                
  |--> case inotify                                                                         
  |    |    +-------------+                                                                 
  |    |--> | LIST_REMOVE | remove souroce from inode                                       
  |    |    +-------------+                                                                 
  |    |    +---------------------+                                                         
  |    +--> | event_gc_inode_data | garbage collect inode_data & inotify_data               
  |         +---------------------+                                                         
  |--> case memory_pressure                                                                 
  |    +--> remove source from event, from epoll                                            
  |                                                                                         
  |--> remove source from event pending/prepare prio queue                                  
  |                                                                                         
  |--> remove source from signal data's earliest/latest queue                               
  |                                                                                         
  +--> remove source from event, event source num--                                         
```

```
src/libsystemd/sd-bus/sd-bus.c              
+--------------+                             
| bus_exit_now | : bus exits                 
+-|------------+                             
  |                                          
  |--> set bus exited = true                 
  |                                          
  |--> if bus->event exists                  
  |    |                                     
  |    |    +---------------+                
  |    +--> | sd_event_exit | set exit fields
  |         +---------------+                
  |                                          
  +--> else                                  
       -                                     
       +--> exit task                        
```

```
src/libsystemd/sd-bus/sd-bus.c
+----------+                                                               
| bus_poll | : determine flags and timeout, then poll                      
+-|--------+                                                               
  |                                                                        
  |--> if bus state == watch_bind                                          
  |    -                                                                   
  |    +--> set up pollfd from inotify fd                                  
  |                                                                        
  +--> else                                                                
       |                                                                   
       |    +-------------------+                                          
       |--> | sd_bus_get_events | determine flags (e.g., poll in, poll out)
       |    +-------------------+                                          
       |    +--------------------+                                         
       |--> | sd_bus_get_timeout | given bus state, determine timeout      
       |    +--------------------+                                         
       |                                                                   
       |--> set up pollfd                                                  
       |                                                                   
       |    +------------+                                                 
       +--> | ppoll_usec |                                                 
            +------------+                                                 
```

```
src/libsystemd/sd-event/sd-event.c                                                 
+--------------------------+                                                        
| memory_pressure_callback | : free unused hash maps, return unused memory to kernel
+----------------------+---+                                                        
| sd_event_trim_memory | : free unused hash maps, return unused memory to kernel    
+-|--------------------+                                                            
  |    +--------------------+                                                       
  |--> | hashmap_trim_pools | free unused hash maps                                 
  |    +--------------------+                                                       
  |    +-------------+                                                              
  +--> | malloc_trim | return unused memory to kernel                               
       +-------------+                                                              
```

```
src/libsystemd/sd-event/sd-event.c                                                                      
+--------------+                                                                                         
| sd_event_run | : register event (sources) to epoll, wait till there's pending source, dispatch it      
+-|------------+                                                                                         
  |    +------------------+                                                                              
  |--> | sd_event_prepare | register event's sources to epoll, arm all kinds of timers, set state = armed
  |    +------------------+                                                                              
  |                                                                                                      
  |--> if nothing happened                                                                               
  |    |                                                                                                 
  |    |    +---------------+                                                                            
  |    +--> | sd_event_wait | wait for each child finishes                                               
  |         +---------------+                                                                            
  |                                                                                                      
  +--> if something is pending                                                                           
       |                                                                                                 
       |    +-------------------+                                                                        
       +--> | sd_event_dispatch | dispatch 1st source from event's pending queue                         
            +-------------------+                                                                        
```

```
src/libsystemd/sd-event/sd-event.c                                                                          
+---------------+                                                                                            
| sd_event_loop | loop: register event to epoll & wait & dispatch it                                         
+-|-------------+                                                                                            
  |                                                                                                          
  |--> when event state!= finished                                                                           
  |    |                                                                                                     
  |    |    +--------------+                                                                                 
  |    +--> | sd_event_run | register event (sources) to epoll, wait till there's pending source, dispatch it
  |         +--------------+                                                                                 
  |                                                                                                          
  +--> return event's exit_code                                                                              
```

```
src/libsystemd/sd-event/sd-event.c                                                             
+-----------------------+                                                                       
| sd_event_set_watchdog | : given arg b, arm or unarm watchdog of event                         
+-|---------------------+                                                                       
  |                                                                                             
  |--> if arg b is provided                                                                     
  |    |                                                                                        
  |    |    +---------------------+                                                             
  |    |--> | sd_watchdog_enabled | if WATCHDOG_PID shows our pid, return 1                     
  |    |    +---------------------+                                                             
  |    |    +-----------+                                                                       
  |    |--> | sd_notify | get addr from env 'NOTIFY_SOCKET', create socket and connect, send msg
  |    |    +-----------+ (WATCHDOG=1)                                                          
  |    |    +----------------+                                                                  
  |    |--> | timerfd_create | create timer fd, save to watchdog                                
  |    |    +----------------+                                                                  
  |    |    +--------------+                                                                    
  |    |--> | arm_watchdog | calculate wait interval, set thru watchdog fd                      
  |    |    +--------------+                                                                    
  |    |                                                                                        
  |    |--> set up epoll event                                                                  
  |    |                                                                                        
  |    |    +-----------+                                                                       
  |    +--> | epoll_ctl | register to epoll                                                     
  |         +-----------+                                                                       
  |                                                                                             
  +--> else                                                                                     
       |                                                                                        
       |    +-----------+                                                                       
       |--> | epoll_ctl | unregister from epoll                                                 
       |    +-----------+                                                                       
       |    +------------+                                                                      
       +--> | safe_close | close the timer fd saved in watchdog                                 
            +------------+                                                                      
```

```
src/libsystemd/sd-daemon/sd-daemon.c                            
+---------------------+                                          
| sd_watchdog_enabled | : if WATCHDOG_PID shows our pid, return 1
+-|-------------------+                                          
  |                                                              
  |--> get env 'WATCHDOG_USEC'                                   
  |                                                              
  |    +-------------+                                           
  +--> | safe_atou64 | convert                                   
  |    +-------------+                                           
  |                                                              
  |--> get env 'WATCHDOG_PID'                                    
  |                                                              
  |    +-----------+                                             
  |--> | parse_pid | convert                                     
  |    +-----------+                                             
  |                                                              
  +--> if pid isn't for us, go to finish                         
  |                                                              
  |--> r = 1                                                     
  |                                                              
  |    +----------+                                              
  |--> | unsetenv | 'WATCHDOG_USEC'                              
  |    +----------+                                              
  |    +----------+                                              
  +--> | unsetenv | 'WATCHDOG_PID'                               
       +----------+                                              
```

```
src/libsystemd/sd-event/sd-event.c                                                                            
+-----------------------------------+                                                                          
| sd_event_source_set_time_accuracy | : set source's time accuracy                                             
+-|---------------------------------+                                                                          
  |    +--------------------+                                                                                  
  |--> | source_set_pending | set pending = false                                                              
  |    +--------------------+                                                                                  
  |                                                                                                            
  |--> save arg accuracy in source                                                                             
  |                                                                                                            
  |    +-----------------------------------+                                                                   
  +--> | event_source_time_prioq_reshuffle | reshuffle target source in event's both queues (earliest & latest)
       +-----------------------------------+                                                                   
```

```
src/libsystemd/sd-bus/bus-objects.c                                                      
+---------------------------+                                                             
| sd_bus_add_object_manager | : prepare node of path, prepare slot and add to node        
+-|-------------------------+                                                             
  |    +-------------------+                                                              
  |--> | bus_node_allocate | ensure path and all ancestors have their nodes in bus hashmap
  |    +-------------------+                                                              
  |    +-------------------+                                                              
  |--> | bus_slot_allocate |  prepapre slot and prepend to bus->slots                     
  |    +-------------------+                                                              
  |                                                                                       
  |--> prepent slot to node->object_managers                                              
  |                                                                                       
  +--> bus->nodes_modified = true                                                         
```

```
src/libsystemd/sd-bus/bus-objects.c                                                         
+-------------------+                                                                        
| bus_node_allocate |    : ensure path and all ancestors have their nodes in bus hashmap     
+-|-----------------+                                                                        
  |    +-------------+                                                                       
  |--> | hashmap_get | given path, get target node from hashmap                              
  |    +-------------+                                                                       
  |                                                                                          
  |--> if got, return node                                                                   
  |                                                                                          
  |    +--------------------------+                                                          
  +--> | hashmap_ensure_allocated | ensure bus hashmap exists                                
  |    +--------------------------+                                                          
  |                                                                                          
  |    if path is /, parent = null                                                           
  |                                                                                          
  |--> else                                                                                  
  |    |                                                                                     
  |    |    +--------------+                                                                 
  |    +--> |strndupa_safe | partially duplicate path for parent (parent path = path - child)
  |    |    +--------------+                                                                 
  |    |    +-------------------+                                                            
  |    +--> | bus_node_allocate | recursively allocate node till '/'                         
  |         +-------------------+                                                            
  |                                                                                          
  |--> alloc node and set up (parent, path)                                                  
  |                                                                                          
  |    +-------------+                                                                       
  |--> | hashmap_put | add node to bus hashmap                                               
  |    +-------------+                                                                       
  |                                                                                          
  +--> if parent node exists                                                                 
       -                                                                                     
       +--> prepend node to parent->child                                                    
```

```
src/libsystemd/sd-bus/bus-objects.c                                                                               
+--------------------------+                                                                                       
| sd_bus_add_object_vtable | : prepare node of path, add each vtable to bus hashmap, prepare slot and add to node  
+----------------------------+                                                                                     
| add_object_vtable_internal | : prepare node of path, add each vtable to bus hashmap, prepare slot and add to node
+-|--------------------------+                                                                                     
  |    +--------------------------+                                                                                
  +--> | hashmap_ensure_allocated | ensure hashmap (bus->vtable_methods) exists                                    
  |    +--------------------------+                                                                                
  |    +--------------------------+                                                                                
  |--> | hashmap_ensure_allocated | ensure hashmap (bus->vtable_properties) exists                                 
  |    +--------------------------+                                                                                
  |    +-------------------+                                                                                       
  |--> | bus_node_allocate | ensure path and all ancestors have their nodes in bus hashmap                         
  |    +-------------------+                                                                                       
  |                                                                                                                
  |--> for each vtable in node                                                                                     
  |    -                                                                                                           
  |    +--> check if there's existing interface(s)                                                                 
  |                                                                                                                
  |    +-------------------+                                                                                       
  |--> | bus_slot_allocate | prepapre slot and prepend to bus->slots                                               
  |    +-------------------+                                                                                       
  |                                                                                                                
  |--> set up slot (fallback/vtable/find/interface)                                                                
  |                                                                                                                
  |--> for each vtable in array                                                                                    
  |    |                                                                                                           
  |    |--> switch vtable type                                                                                     
  |    |--> case method                                                                                            
  |    |    +--> prepare member and add to bus->vtable_methods                                                     
  |    |--> case writable_property/property                                                                        
  |    |    +--> prepare member and add to bus->vtable_properties                                                  
  |    +--> case signal                                                                                            
  |         +--> check if valid                                                                                    
  |                                                                                                                
  |--> insert slot to node                                                                                         
  |                                                                                                                
  +--> bus->nodes_modified = true                                                                                  
```

```
src/libsystemd/sd-bus/sd-bus.c                                                                                      
+------------------+                                                                                                 
| sd_bus_add_match | : send msg ('add match' method) to bus clients, add match rule to bus                           
+--------------------+                                                                                               
| bus_add_match_full | : send msg ('add match' method) to bus clients, add match rule to bus                         
+-|------------------+                                                                                               
  |    +-----------------+                                                                                           
  |--> | bus_match_parse | parse match fule                                                                          
  |    +-----------------+                                                                                           
  |    +-------------------+                                                                                         
  |--> | bus_slot_allocate | prepare slot                                                                            
  |    +-------------------+                                                                                         
  |                                                                                                                  
  |--> install arg callbacks to slot                                                                                 
  |                                                                                                                  
  |--> if bus has client                                                                                             
  |    |                                                                                                             
  |    |    +---------------------+                                                                                  
  |    |--> | bus_match_get_scope | traverse components to determine scope (local, driver, or generic)               
  |    |    +---------------------+                                                                                  
  |    |                                                                                                             
  |    +--> if scope != local                                                                                        
  |         |                                                                                                        
  |         |--> if arg async is true                                                                                
  |         |    |                                                                                                   
  |         |    |    +------------------------------+                                                               
  |         |    |--> | bus_add_match_internal_async | prepare msg to call 'add match', prepare slot and send msg out
  |         |    |    +------------------------------+                                                               
  |         |    |    +--------------------------+                                                                   
  |         |    +--> | sd_bus_slot_set_floating | set slot floating = true                                          
  |         |         +--------------------------+                                                                   
  |         |                                                                                                        
  |         +--> else                                                                                                
  |              |                                                                                                   
  |              |    +------------------------+                                                                     
  |              +--> | bus_add_match_internal | prepare msg to call 'add match', send msg out, wait for reply       
  |                   +------------------------+                                                                     
  |    +---------------+                                                                                             
  +--> | bus_match_add | add match rule (leaf + several values)                                                      
       +---------------+                                                                                             
```

```
src/libsystemd/sd-bus/bus-match.c                                                          
+---------------------+                                                                     
| bus_match_get_scope | : traverse components to determine scope (local, driver, or generic)
+-|-------------------+                                                                     
  |                                                                                         
  +--> for each component                                                                   
       |                                                                                    
       |--> get current component                                                           
       |                                                                                    
       |--> if component type is 'sender'                                                   
       |    -                                                                               
       |    +--> compare component value with "org.freedesktop.DBus.Local"                  
       |                                                                                    
       |--> if component type is 'interface'                                                
       |    -                                                                               
       |    +--> compare component value with "org.freedesktop.DBus.Local"                  
       |                                                                                    
       +--> if component type is 'path'                                                     
            -                                                                               
            +--> compare component value with "/org/freedesktop/DBus/Local"                 
```

```
src/libsystemd/sd-bus/bus-control.c                                                                                    
+------------------------------+                                                                                        
| bus_add_match_internal_async | : prepare msg to call 'add match', prepare slot and send msg out                       
+-|----------------------------+                                                                                        
  |    +------------------+                                                                                             
  |--> | append_eavesdrop | determine eavesdrop string                                                                  
  |    +------------------+                                                                                             
  |    +--------------------------------+                                                                               
  |--> | sd_bus_message_new_method_call | prepare msg of method_call type, append path/member/interface/destination info
  |    +--------------------------------+                                                                               
  |    +-----------------------+                                                                                        
  |--> | sd_bus_message_append | append eavesdrop string to msg                                                         
  |    +-----------------------+                                                                                        
  |    +-------------------+                                                                                            
  +--> | sd_bus_call_async | prepare slot (install callback, insert to hashmap/prioq), send msg out                     
       +-------------------+                                                                                            
```

```
src/libsystemd/sd-bus/sd-bus.c                                                               
+-------------------+                                                                         
| sd_bus_call_async | : prepare slot (install callback, insert to hashmap/prioq), send msg out
+-|-----------------+                                                                         
  |    +----------------------------------+                                                   
  |--> | ordered_hashmap_ensure_allocated | ensure hashmap (bus->reply_callbacks) exists      
  |    +----------------------------------+                                                   
  |    +------------------------+                                                             
  |--> | prioq_ensure_allocated | ensure prioq (bus->reply_callbacks_prioq) exists            
  |    +------------------------+                                                             
  |    +------------------+                                                                   
  |--> | bus_seal_message |                                                                   
  |    +------------------+                                                                   
  |    +-----------------------+                                                              
  |--> | bus_remarshal_message | if necessary, remarshal msg                                  
  |    +-----------------------+                                                              
  |                                                                                           
  |--> if slot or callback is provided                                                        
  |    |                                                                                      
  |    |    +-------------------+                                                             
  |    |--> | bus_slot_allocate | prepare slot                                                
  |    |    +-------------------+                                                             
  |    |                                                                                      
  |    |--> install arg callback to slot                                                      
  |    |                                                                                      
  |    |--> insert slot to hashmap (bus->reply_callbacks)                                     
  |    |                                                                                      
  |    +--> insert slot to prioq (bus->reply_callbacks_prioq)                                 
  |                                                                                           
  |    +-------------+                                                                        
  +--> | sd_bus_send | seal, send msg out or append to wqueue                                 
       +-------------+                                                                        
```

```
src/libsystemd/sd-bus/bus-control.c                                                                                    
+------------------------+                                                                                              
| bus_add_match_internal | : prepare msg to call 'add match', send msg out, wait for reply                              
+-|----------------------+                                                                                              
  |    +------------------+                                                                                             
  |--> | append_eavesdrop | determine eavesdrop string                                                                  
  |    +------------------+                                                                                             
  |    +--------------------------------+                                                                               
  |--> | sd_bus_message_new_method_call | prepare msg of method_call type, append path/member/interface/destination info
  |    +--------------------------------+                                                                               
  |    +-----------------------+                                                                                        
  |--> | sd_bus_message_append | append eavesdrop string to msg                                                         
  |    +-----------------------+                                                                                        
  |    +-------------+                                                                                                  
  +--> | sd_bus_call | send msg out, wait for reply                                                                     
       +-------------+                                                                                                  
```

```
src/libsystemd/sd-bus/sd-bus.c                                                                                     
+-------------+                                                                                                     
| sd_bus_call | : send msg out, wait for reply                                                                      
+-|-----------+                                                                                                     
  |    +--------------------+                                                                                       
  |--> | bus_ensure_running | ensure bus is running                                                                 
  |    +--------------------+                                                                                       
  |    +------------------+                                                                                         
  |--> | bus_seal_message |                                                                                         
  |    +------------------+                                                                                         
  |    +-----------------------+                                                                                    
  |--> | bus_remarshal_message | if necessary, remarshal msg                                                        
  |    +-----------------------+                                                                                    
  |    +-------------+                                                                                              
  |--> | sd_bus_send | seal, send msg out or append to wqueue                                                       
  |    +-------------+                                                                                              
  |                                                                                                                 
  +--> endless loop                                                                                                 
       |                                                                                                            
       |--> while i < rqueue size                                                                                   
       |    |                                                                                                       
       |    |--> if cookie matches                                                                                  
       |    |    |                                                                                                  
       |    |    |    +-----------------+                                                                           
       |    |    |--> | rqueue_drop_one | remove msg from rqueue                                                    
       |    |    |    +-----------------+                                                                           
       |    |    |                                                                                                  
       |    |    +--> if arg reply is prvided, reutrn reply, and return                                             
       |    |                                                                                                       
       |    +--> i++                                                                                                
       |                                                                                                            
       |    +------------------+                                                                                    
       |--> | bus_read_message | prepare rbuffer, read data from bus's input fd, convert to msg and append to rqueue
       |    +------------------+                                                                                    
       |                                                                                                            
       |--> if read something, continue                                                                             
       |                                                                                                            
       |    +----------+                                                                                            
       |--> | bus_poll | determine flags and timeout, then poll                                                     
       |    +----------+                                                                                            
       |    +-----------------+                                                                                     
       +--> | dispatch_wqueue | for each msg in write queue: write to bus's output fd                               
            +-----------------+                                                                                     
```

```
src/libsystemd/sd-bus/bus-match.c                                                                  
+---------------+                                                                                   
| bus_match_add | : add match rule (leaf + several values)                                          
+-|-------------+                                                                                   
  |                                                                                                 
  |--> for each component                                                                           
  |    |                                                                                            
  |    |    +-----------------------------+                                                         
  |    +--> | bus_match_add_compare_value | ensure the node (type = match value) exists             
  |         +-----------------------------+                                                         
  |    +--------------------+                                                                       
  +--> | bus_match_add_leaf | alooc node (type = match leaf), add it between arg where and its child
       +--------------------+                                                                       
```

```
src/libsystemd/sd-bus/bus-match.c                                           
+-----------------------------+                                              
| bus_match_add_compare_value | : ensure the node (type = match value) exists
+-|---------------------------+                                              
  |                                                                          
  |--> find node that matches the arg type                                   
  |                                                                          
  |--> if found                                                              
  |    -                                                                     
  |    +--> get node and return it, return                                   
  |                                                                          
  |--> else                                                                  
  |    |                                                                     
  |    |--> alloc node and set up                                            
  |    |                                                                     
  |    +--> create hashmap (c->compare.children)                             
  |                                                                          
  |--> alloc node and set up                                                 
  |                                                                          
  +--> add to the newly created hashmap                                      
```

```
src/libsystemd/sd-bus/sd-bus.c                                                                                              
+---------------------+                                                                                                      
| sd_bus_attach_event | : prepare all kinds of sources for event                                                             
+-|-------------------+                                                                                                      
  |    +-------------------+                                                                                                 
  |--> | sd_event_add_time | set up timer fields of event (fd/queues), prepare 'source' and add to those queues              
  |    +-------------------+                                                                                                 
  |    +------------------------------+                                                                                      
  |--> | sd_event_source_set_priority | s->priority = priority, reposition source in data structure                          
  |    +------------------------------+                                                                                      
  |    +---------------------------------+                                                                                   
  |--> | sd_event_source_set_description | duplicate description into source                                                 
  |    +---------------------------------+                                                                                   
  |    +-------------------+                                                                                                 
  |--> | sd_event_add_exit | prepare 'source' for exit and add to even'ts exit queue                                         
  |    +-------------------+                                                                                                 
  |    +---------------------------------+                                                                                   
  |--> | sd_event_source_set_description | (the above is for time source, this one is for quit source)                       
  |    +---------------------------------+                                                                                   
  |    +----------------------+                                                                                              
  |--> | bus_attach_io_events | for input/output fd: prepare source and register to epoll & set source's priority/description
  |    +----------------------+                                                                                              
  |    +--------------------------+                                                                                          
  +--> | bus_attach_inotify_event | prepare source and register to epoll & set source's priority/description                 
       +--------------------------+                                                                                          
```

```
src/libsystemd/sd-bus/bus-objects.c                                                                                 
+-----------------------------------+                                                                                
| sd_bus_emit_interfaces_added_strv | : prepare msg (signal type), append path/interfaces/properties to msg, send out
+-|---------------------------------+                                                                                
  |    +--------------------------------+                                                                            
  |--> | bus_find_parent_object_manager | find parent object manager                                                 
  |    +--------------------------------+                                                                            
  |    +---------------------------+                                                                                 
  |--> | sd_bus_message_new_signal | prepare msg of signal type, append path/interface/member info                   
  |    +---------------------------+                                                                                 
  |    +-----------------------------+                                                                               
  |--> | sd_bus_message_append_basic | append path to msg                                                            
  |    +-----------------------------+                                                                               
  |    +-------------------------------+                                                                             
  |--> | sd_bus_message_open_container | ensure contents in signature, set up container for it and add to msg        
  |    +-------------------------------+                                                                             
  |                                                                                                                  
  |--> for each interfaces                                                                                           
  |    |                                                                                                             
  |    |    +-------------------------------+                                                                        
  |    |--> | sd_bus_message_open_container | ensure contents in signature, set up container for it and add to msg   
  |    |    +-------------------------------+                                                                        
  |    |    +-----------------------------+                                                                          
  |    |--> | interfaces_added_append_one | append target interface and its properties to msg                        
  |    |    +-----------------------------+                                                                          
  |    |    +--------------------------------+                                                                       
  |    +--> | sd_bus_message_close_container | close container and free signature                                    
  |         +--------------------------------+                                                                       
  |    +--------------------------------+                                                                            
  |--> | sd_bus_message_close_container | close container and free signature                                         
  |    +--------------------------------+                                                                            
  |    +-------------+                                                                                               
  +--> | sd_bus_send | seal, send msg out or append to wqueue                                                        
       +-------------+                                                                                               
```

```
src/libsystemd/sd-bus/bus-objects.c                              
+--------------------------------+                                
| bus_find_parent_object_manager | : find parent object manager   
+-|------------------------------+                                
  |    +-------------+                                            
  |--> | hashmap_get | given path, get node from hashmap          
  |    +-------------+                                            
  |                                                               
  |--> if not found                                               
  |    |                                                          
  |    |--> alloc buffer for prefix                               
  |    |                                                          
  |    +--> for each ancestor of path                             
  |         |                                                     
  |         |    +-------------+                                  
  |         |--> | hashmap_get | given path, get node from hashmap
  |         |    +-------------+                                  
  |         |                                                     
  |         +--> if found, break                                  
  |                                                               
  +--> while node has no object_managers                          
       -                                                          
       +--> node = node->parent                                   
```

```
src/libsystemd/sd-bus/bus-objects.c                                                                
+-----------------------------+                                                                     
| interfaces_added_append_one | : append target interface and its properties to msg                 
+-|---------------------------+                                                                     
  |    +------------------------------------+                                                       
  |--> | interfaces_added_append_one_prefix | append target interface and its properties to msg     
  |    +------------------------------------+                                                       
  |                                                                                                 
  |--> if append something, return                                                                  
  |                                                                                                 
  +--> alloc buf for prefix                                                                         
                                                                                                    
       for each ancestor of path                                                                    
       |                                                                                            
       |    +------------------------------------+                                                  
       |--> | interfaces_added_append_one_prefix | append target interface and its properties to msg
       |    +------------------------------------+                                                  
       |                                                                                            
       +--> if append something, return                                                             
```

```
src/libsystemd/sd-bus/bus-objects.c                                                                                   
+------------------------------------+                                                                                 
| interfaces_added_append_one_prefix | : append target interface and its properties to msg                             
+-|----------------------------------+                                                                                 
  |    +-------------+                                                                                                 
  |--> | hashmap_get | given path, get node from hashmap                                                               
  |    +-------------+                                                                                                 
  |                                                                                                                    
  |--> for each vtable in node                                                                                         
  |    |                                                                                                               
  |    |--> if it doesn't match arg interface, continue                                                                
  |    |                                                                                                               
  |    |    +--------------------------+                                                                               
  |    |--> | node_vtable_get_userdata | get user data                                                                 
  |    |    +--------------------------+                                                                               
  |    |                                                                                                               
  |    |--> if found_interface == false                                                                                
  |    |    |                                                                                                          
  |    |    |    +-----------------------------+                                                                       
  |    |    |--> | sd_bus_message_append_basic | append interface to msg                                               
  |    |    |    +-----------------------------+                                                                       
  |    |    |    +-------------------------------+                                                                     
  |    |    |--> | sd_bus_message_open_container | ensure contents in signature, set up container for it and add to msg
  |    |    |    +-------------------------------+                                                                     
  |    |    |                                                                                                          
  |    |    +--> found_interface = true                                                                                
  |    |                                                                                                               
  |    |    +------------------------------+                                                                           
  |    +--> | vtable_append_all_properties | given node, append all properties (name/value) to msg                     
  |         +------------------------------+                                                                           
  |                                                                                                                    
  +--> if found_interface                                                                                              
       |                                                                                                               
       |    +--------------------------------+                                                                         
       +--> | sd_bus_message_close_container | close container and free signature                                      
            +--------------------------------+                                                                         
```

```
src/libsystemd/sd-bus/bus-objects.c                                                                                  
+-------------------------------------+                                                                               
| sd_bus_emit_interfaces_removed_strv | : prepare msg (signal type), append path/interfaces to msg, send out          
+-|-----------------------------------+                                                                               
  |    +--------------------------------+                                                                             
  |--> | bus_find_parent_object_manager | find parent object manager                                                  
  |    +--------------------------------+                                                                             
  |    +---------------------------+                                                                                  
  |--> | sd_bus_message_new_signal | prepare msg of signal (InterfacesRemoved) type, append path/interface/member info
  |    +---------------------------+                                                                                  
  |    +-----------------------------+                                                                                
  |--> | sd_bus_message_append_basic | append path to msg                                                             
  |    +-----------------------------+                                                                                
  |    +----------------------------+                                                                                 
  |--> | sd_bus_message_append_strv | append each interface to msg                                                    
  |    +----------------------------+                                                                                 
  |    +-------------+                                                                                                
  +--> | sd_bus_send | seal, send msg out or append to wqueue                                                         
       +-------------+                                                                                                
```

```
src/libsystemd/sd-bus/bus-objects.c                                                                                        
+--------------------------+                                                                                                
| sd_bus_emit_object_added | : prepare signal (InterfacesAdded) msg, append all interfaces of all ancestors to msg, send out
+-|------------------------+                                                                                                
  |    +--------------------------------+                                                                                   
  |--> | bus_find_parent_object_manager | find parent object manager                                                        
  |    +--------------------------------+                                                                                   
  |    +---------------------------+                                                                                        
  |--> | sd_bus_message_new_signal | prepare msg of signal (InterfacesAdded) type, append path/interface/member info        
  |    +---------------------------+                                                                                        
  |    +-----------------------------+                                                                                      
  |--> | sd_bus_message_append_basic | append path to msg                                                                   
  |    +-----------------------------+                                                                                      
  |    +-------------------------------+                                                                                    
  |--> | sd_bus_message_open_container | ensure contents in signature, set up container for it and add to msg               
  |    +-------------------------------+                                                                                    
  |    +-------------------------+                                                                                          
  |--> | object_added_append_all | for all ancestors of path: append each interfaces to msg                                 
  |    +-------------------------+                                                                                          
  |    +--------------------------------+                                                                                   
  |--> | sd_bus_message_close_container | close container and free signature                                                
  |    +--------------------------------+                                                                                   
  |    +-------------+                                                                                                      
  +--> | sd_bus_send | seal, send msg out or append to wqueue                                                               
       +-------------+                                                                                                      
```

```
src/libsystemd/sd-bus/bus-objects.c                                                                   
+-------------------------+                                                                            
| object_added_append_all | for all ancestors of path: append each interfaces to msg                   
+-|-----------------------+                                                                            
  |    +-----------------+                                                                             
  |--> | ordered_set_new | create ordered set                                                          
  |    +-----------------+                                                                             
  |    +-----------------------+                                                                       
  |--> | sd_bus_message_append | append 'org.freedesktop.DBus.Peer' to msg                             
  |    +-----------------------+                                                                       
  |    +-----------------------+                                                                       
  |--> | sd_bus_message_append | append 'org.freedesktop.DBus.Introspectable' to msg                   
  |    +-----------------------+                                                                       
  |    +-----------------------+                                                                       
  |--> | sd_bus_message_append | append 'org.freedesktop.DBus.Properties' to msg                       
  |    +-----------------------+                                                                       
  |                                                                                                    
  |--> if path has object manager                                                                      
  |    |                                                                                               
  |    |    +-----------------------+                                                                  
  |    +--> | sd_bus_message_append | append 'org.freedesktop.DBus.ObjectManager' to msg               
  |         +-----------------------+                                                                  
  |    +--------------------------------+                                                              
  |--> | object_added_append_all_prefix | for each interface in node, add to set and append to msg     
  |    +--------------------------------+                                                              
  |                                                                                                    
  |--> alloc buf for prefix                                                                            
  |                                                                                                    
  +--> for each ancestor of path                                                                       
       |                                                                                               
       |    +--------------------------------+                                                         
       +--> | object_added_append_all_prefix | for each interface in node, add to set and append to msg
            +--------------------------------+                                                         
```

```
src/libsystemd/sd-bus/bus-objects.c                                                         
+--------------------------------+                                                           
| object_added_append_all_prefix | : for each interface in node, add to set and append to msg
+-|------------------------------+                                                           
  |                                                                                          
  |--> given prefix, get node from hashmap                                                   
  |                                                                                          
  +--> for each vtable in node                                                               
       |                                                                                     
       |    +--------------------------+                                                     
       |--> | node_vtable_get_userdata | get user data                                       
       |    +--------------------------+                                                     
       |                                                                                     
       +--> if current interface != previous_interface                                       
            |                                                                                
            |--> get interface from set                                                      
            |                                                                                
            |--> if got, continue                                                            
            |                                                                                
            |--> add interface to set                                                        
            |                                                                                
            |    +-----------------------+                                                   
            |--> | sd_bus_message_append | append interface to msg                           
            |    +-----------------------+                                                   
            |                                                                                
            +--> previous_interface = current one                                            
```

```
src/libsystemd/sd-bus/bus-objects.c                                                                                            
+----------------------------+                                                                                                  
| sd_bus_emit_object_removed | : prepare signal (InterfacesRemoved) msg, append all interfaces of all ancestors to msg, send out
+-|--------------------------+                                                                                                  
  |    +--------------------------------+                                                                                       
  |--> | bus_find_parent_object_manager | find parent object manager                                                            
  |    +--------------------------------+                                                                                       
  |    +---------------------------+                                                                                            
  |--> | sd_bus_message_new_signal | prepare msg of signal (InterfacesRemoved) type, append path/interface/member info          
  |    +---------------------------+                                                                                            
  |    +-----------------------------+                                                                                          
  |--> | sd_bus_message_append_basic | append path to msg                                                                       
  |    +-----------------------------+                                                                                          
  |    +---------------------------+                                                                                            
  |--> | object_removed_append_all | for all ancestors, append each interface to msg                                            
  |    +---------------------------+                                                                                            
  |    +-------------+                                                                                                          
  +--> | sd_bus_send | seal, send msg out or append to wqueue                                                                   
       +-------------+                                                                                                          
```

```
src/libsystemd/sd-bus/bus-objects.c                                                             
+---------------------------+                                                                    
| object_removed_append_all | : for all ancestors, append each interface to msg                  
+-|-------------------------+                                                                    
  |    +-----------------+                                                                       
  |--> | ordered_set_new | create ordered set                                                    
  |    +-----------------+                                                                       
  |    +-----------------------+                                                                 
  |--> | sd_bus_message_append | append 'org.freedesktop.DBus.Peer' to msg                       
  |    +-----------------------+                                                                 
  |    +-----------------------+                                                                 
  |--> | sd_bus_message_append | append 'org.freedesktop.DBus.Introspectable' to msg             
  |    +-----------------------+                                                                 
  |    +-----------------------+                                                                 
  |--> | sd_bus_message_append | append 'org.freedesktop.DBus.Properties' to msg                 
  |    +-----------------------+                                                                 
  |                                                                                              
  |--> if path has object manager                                                                
  |    |                                                                                         
  |    |    +-----------------------+                                                            
  |    +--> | sd_bus_message_append | append 'org.freedesktop.DBus.ObjectManager' to msg         
  |         +-----------------------+                                                            
  |    +----------------------------------+                                                      
  |--> | object_removed_append_all_prefix | for each vtable in node, append interface to msg     
  |    +----------------------------------+                                                      
  |                                                                                              
  |--> alloc buf for prefix                                                                      
  |                                                                                              
  +--> for each ancestor of path                                                                 
       |                                                                                         
       |    +----------------------------------+                                                 
       +--> | object_removed_append_all_prefix | for each vtable in node, append interface to msg
            +----------------------------------+                                                 
```

```
src/libsystemd/sd-bus/bus-objects.c                                                   
+----------------------------------+                                                   
| object_removed_append_all_prefix | : for each vtable in node, append interface to msg
+-|--------------------------------+                                                   
  |                                                                                    
  |--> given prefix, get node from hashmap                                             
  |                                                                                    
  +--> for each vtable in node                                                         
       |                                                                               
       |--> if current interface == previous_interface, continue                       
       |                                                                               
       |--> if we can get interface from ordered set, continue                         
       |                                                                               
       |    +--------------------------+                                               
       |--> | node_vtable_get_userdata | get user data                                 
       |    +--------------------------+                                               
       |    +-----------------+                                                        
       |--> | ordered_set_put | add interface to ordered set                           
       |    +-----------------+                                                        
       |    +-----------------------+                                                  
       |--> | sd_bus_message_append | append interface to msg                          
       |    +-----------------------+                                                  
       |                                                                               
       +--> previous_interface = current one                                           
```

```
src/libsystemd/sd-bus/bus-objects.c                                                                                                                     
+-------------------------------------+                                                                                                                  
| sd_bus_emit_properties_changed_strv | : prepare msg of signal (PropertiesChanged), append changed/invalidated properties to msg, send it out           
+-|-----------------------------------+                                                                                                                  
  |                                                                                                                                                      
  |--> alloc buf for prefix                                                                                                                              
  |                                                                                                                                                      
  |    +--------------------------------------+                                                                                                          
  |--> | emit_properties_changed_on_interface | prepare msg of signal (PropertiesChanged), append changed/invalidated properties to msg, send it out     
  |    +--------------------------------------+                                                                                                          
  |                                                                                                                                                      
  |--> if did something, return                                                                                                                          
  |                                                                                                                                                      
  +--> for each ancestor of path                                                                                                                         
       |                                                                                                                                                 
       |    +--------------------------------------+                                                                                                     
       +--> | emit_properties_changed_on_interface | prepare msg of signal (PropertiesChanged), append changed/invalidated properties to msg, send it out
            +--------------------------------------+                                                                                                     
```

```
src/libsystemd/sd-bus/bus-objects.c                                                                                                           
+--------------------------------------+                                                                                                       
| emit_properties_changed_on_interface | : prepare msg of signal (PropertiesChanged), append changed/invalidated properties to msg, send it out
+-|------------------------------------+                                                                                                       
  |                                                                                                                                            
  |--> given prefix, get node from hashmap                                                                                                     
  |                                                                                                                                            
  |    +---------------------------+                                                                                                           
  |--> | sd_bus_message_new_signal | prepare msg of signal (PropertiesChanged) type, append path/interface/member info                         
  |    +---------------------------+                                                                                                           
  |    +-----------------------+                                                                                                               
  |--> | sd_bus_message_append | append interface to msg                                                                                       
  |    +-----------------------+                                                                                                               
  |                                                                                                                                            
  |--> for each vtable in node                                                                                                                 
  |    |                                                                                                                                       
  |    |--> if it's not our target interface, continue                                                                                         
  |    |                                                                                                                                       
  |    |    +--------------------------+                                                                                                       
  |    |--> | node_vtable_get_userdata | get user data                                                                                         
  |    |    +--------------------------+                                                                                                       
  |    |                                                                                                                                       
  |    |--> if property name list is provided                                                                                                  
  |    |    -                                                                                                                                  
  |    |    +--> for each property name                                                                                                        
  |    |         -                                                                                                                             
  |    |         +--> get property from bus hashmap                                                                                            
  |    |         |                                                                                                                             
  |    |         |    +----------------------------+                                                                                           
  |    |         +--> | vtable_append_one_property | append property name/value to msg                                                         
  |    |              +----------------------------+                                                                                           
  |    |                                                                                                                                       
  |    +--> else                                                                                                                               
  |         -                                                                                                                                  
  |         +--> for each member in vtable                                                                                                     
  |              |                                                                                                                             
  |              |    +----------------------------+                                                                                           
  |              +--> | vtable_append_one_property | append property name/value to msg                                                         
  |                   +----------------------------+                                                                                           
  |                                                                                                                                            
  |--> if any member is invalidating                                                                                                           
  |    -                                                                                                                                       
  |    +--> append those members to msg                                                                                                        
  |                                                                                                                                            
  |    +-------------+                                                                                                                         
  +--> | sd_bus_send | seal, send msg out or append to wqueue                                                                                  
       +-------------+                                                                                                                         
```

```
src/libsystemd/sd-bus/bus-control.c                                                                                                 
+-------------------+                                                                                                                
| sd_bus_list_names | : prepare msg of 'method call' for ListNames and ListActivatableNames separately, return reply                 
+-|-----------------+                                                                                                                
  |                                                                                                                                  
  |--> if arg acquired is provided                                                                                                   
  |    |                                                                                                                             
  |    |    +--------------------+                                                                                                   
  |    |--> | sd_bus_call_method | prepare msg of 'method call' (ListNames), append types to msg, send out, wait for reply           
  |    |    +--------------------+                                                                                                   
  |    |    +--------------------------+                                                                                             
  |    +--> | sd_bus_message_read_strv | peek type, read data of that type from msg                                                  
  |         +--------------------------+                                                                                             
  |                                                                                                                                  
  +--> if arg activatable is provided                                                                                                
       |                                                                                                                             
       |    +--------------------+                                                                                                   
       |--> | sd_bus_call_method | prepare msg of 'method call' (ListActivatableNames), append types to msg, send out, wait for reply
       |    +--------------------+                                                                                                   
       |    +--------------------------+                                                                                             
       +--> | sd_bus_message_read_strv | peek type, read data of that type from msg                                                  
            +--------------------------+                                                                                             
```

```
src/libsystemd/sd-bus/bus-convenience.c                                                                                
+--------------------+                                                                                                  
| sd_bus_call_method | : prepare msg of 'method call', append types to msg, send out, wait for reply                    
+---------------------+                                                                                                 
| sd_bus_call_methodv | : prepare msg of 'method call', append types to msg, send out, wait for reply                   
+-|-------------------+                                                                                                 
  |    +--------------------------------+                                                                               
  |--> | sd_bus_message_new_method_call | prepare msg of method_call type, append path/member/interface/destination info
  |    +--------------------------------+                                                                               
  |                                                                                                                     
  |--> if arg type is valid                                                                                             
  |    |                                                                                                                
  |    |    +------------------------+                                                                                  
  |    +--> | sd_bus_message_appendv | append types to msg                                                              
  |         +------------------------+                                                                                  
  |    +-------------+                                                                                                  
  +--> | sd_bus_call | send msg out, wait for reply                                                                     
       +-------------+                                                                                                  
```

```
src/libsystemd/sd-bus/bus-message.c                                            
+--------------------------+                                                    
| sd_bus_message_read_strv | : peek type, read data of that type from msg       
+---------------------------------+                                             
| sd_bus_message_read_strv_extend | : peek type, read data of that type from msg
+-|-------------------------------+                                             
  |    +--------------------------+                                             
  |--> | sd_bus_message_peek_type | peek type (and signature)                   
  |    +--------------------------+                                             
  |    +---------------------------+                                            
  +--> | sd_bus_message_read_basic | read data of type from msg                 
       +---------------------------+                                            
```

```
src/libsystemd/sd-bus/bus-message.c                                                          
+------------------------------------+                                                        
| sd_bus_message_append_string_iovec |                                                        
+-|----------------------------------+                                                        
  |    +------------------------------------+                                                 
  |--> | sd_bus_message_append_string_space | append type string to signature, extend msg body
  |    +------------------------------------+                                                 
  |                                                                                           
  +--> copy io vectors to the extended msg body                                               
```

```
src/libsystemd/sd-bus/bus-message.c                                                     
+------------------------------------+                                                   
| sd_bus_message_append_string_space | : append type string to signature, extend msg body
+-|----------------------------------+                                                   
  |    +----------------------------+                                                    
  |--> | message_get_last_container | get last container from msg                        
  |    +----------------------------+                                                    
  |                                                                                      
  |--> ensure type string is in signature                                                
  |                                                                                      
  |    +---------------------+                                                           
  +--> | message_extend_body | extend msg body                                           
       +---------------------+                                                           
```

```
src/libsystemd/sd-bus/bus-message.c                                                        
+---------------------------------+                                                         
| sd_bus_message_new_method_error | : prepare msg of 'method error', append error name to it
+-|-------------------------------+                                                         
  |    +-------------------+                                                                
  |--> | message_new_reply | prepare msg of arg type                                        
  |    +-------------------+                                                                
  |    +-----------------------------+                                                      
  |--> | message_append_field_string | append error name to msg                             
  |    +-----------------------------+                                                      
  |                                                                                         
  +--> if error has msg                                                                     
       |                                                                                    
       |    +----------------------+                                                        
       +--> | message_append_basic | append type string to msg                              
            +----------------------+                                                        
```

```
src/libsystemd/sd-bus/bus-message.c                          
+---------------------+                                       
| sd_bus_message_skip | : determine type, and skip accordingly
+-|-------------------+                                       
  |                                                           
  |--> if type isn't provided                                 
  |    -                                                      
  |    +--> determine type from container's signature         
  |                                                           
  +--> switch type                                            
       |                                                      
       |    +---------------------+                           
       +--> | sd_bus_message_skip | (recursive call)          
            +---------------------+                           
```

```
src/libsystemd/sd-bus/bus-control.c                                                                                   
+---------------------+                                                                                                
| sd_bus_request_name | : prepare msg of 'method call' (RequestName), send out and wait for reply                      
+-|-------------------+                                                                                                
  |    +--------------------+                                                                                          
  |--> | sd_bus_call_method | prepare msg of 'method call' (RequestName), append types to msg, send out, wait for reply
  |    +--------------------+                                                                                          
  |    +---------------------+                                                                                         
  +--> | sd_bus_message_read | read data (type = uint32) from reply                                                    
       +---------------------+                                                                                         
```

```
src/libsystemd/sd-bus/sd-bus.c                                             
+-----------------------+                                                   
| sd_bus_default_system | : ensue we have a default bus                     
+-------------+---------+                                                   
| bus_default | : ensue we have a default bus                               
+-|-----------+                                                             
  |                                                                         
  |--> if the arg default_bus already exist                                 
  |    -                                                                    
  |    +--> ref++, return                                                   
  |                                                                         
  |--> call arg bus_open(), e.g.,                                           
  |    +--------------------+                                               
  |    | sd_bus_open_system | prepare bus, determine address and save in bus
  |    +--------------------+                                               
  |                                                                         
  |--> bus->default_bus_ptr = arg default_bus                               
  |                                                                         
  +--> bus->tid = gettid()                                                  
```

```
src/libsystemd/sd-bus/sd-bus.c                                                            
+--------------------+                                                                     
| sd_bus_open_system | : prepare bus, determine address and save in bus                    
+-------------------------------------+                                                    
| sd_bus_open_system_with_description | : prepare bus, determine address and save in bus   
+-|-----------------------------------+                                                    
  |    +------------+                                                                      
  |--> | sd_bus_new | alloc and init bus                                                   
  |    +------------+                                                                      
  |                                                                                        
  |--> if description is provided                                                          
  |    |                                                                                   
  |    |    +------------------------+                                                     
  |    +--> | sd_bus_set_description | bus->description = description                      
  |         +------------------------+                                                     
  |    +------------------------+                                                          
  +--> | bus_set_address_system | determine address and save in bus, set run_scope = system
       +------------------------+                                                          
```

```
src/libsystemd/sd-bus/sd-bus.c                                                              
+------------------------+                                                                   
| bus_set_address_system | : determine address and save in bus, set run_scope = system       
+-|----------------------+                                                                   
  |                                                                                          
  |--> get env 'DBUS_SYSTEM_BUS_ADDRESS'                                                     
  |                                                                                          
  |--> determine address, $DBUS_SYSTEM_BUS_ADDRESS or "unix:path=/run/dbus/system_bus_socket"
  |                                                             (probably our case)          
  |    +--------------------+                                                                
  |--> | sd_bus_set_address | bus->address = address                                         
  |    +--------------------+                                                                
  |                                                                                          
  +--> bus->runtime_scope = RUNTIME_SCOPE_SYSTEM                                             
```

```
src/libsystemd/sd-daemon/sd-daemon.c                                                                          
+---------------+                                                                                              
| sd_listen_fds | : for fd in target range, set FD_CLOEXEC to flags                                            
+-|-------------+                                                                                              
  |                                                                                                            
  |--> get env "LISTEN_PID"                                                                                    
  |                                                                                                            
  |    +-----------+                                                                                           
  |--> | parse_pid | parse string and convert to pid_t type                                                    
  |    +-----------+                                                                                           
  |                                                                                                            
  |--> if the pid isn't for us, return                                                                         
  |                                                                                                            
  |--> get env "LISTEN_FDS" and convert to n                                                                   
  |                                                                                                            
  |--> for fd in range [3, 3 + n)                                                                              
  |    |                                                                                                       
  |    |    +------------+                                                                                     
  |    +--> | fd_cloexec | set or unset FD_CLOEXEC                                                             
  |         +------------+                                                                                     
  |    +--------------+                                                                                        
  +--> | unsetenv_all | if unset_environment is specified, clear env "LISTEN_PID"/"LISTEN_FDS"/"LISTEN_FDNAMES"
       +--------------+                                                                                        
```

```
src/basic/fd-util.c                           
+------------+                                 
| fd_cloexec | : set or unset FD_CLOEXEC       
+-|----------+                                 
  |                                            
  |--> given fd, get its flags                 
  |                                            
  |--> given cloexec, OR nad NOT-AND FD_CLOEXEC
  |                                            
  +--> set updated flags to fd                 
```

```
src/libsystemd/sd-daemon/sd-daemon.c                                                                      
+--------------+                                                                                           
| sd_is_socket | : check if fd meets args (e.g., sock? STREAM? listening? PF_UNIX?)                        
+-|------------+                                                                                           
  |    +--------------------+                                                                              
  |--> | is_socket_internal | check if fd meets our arguments (e.g., is sock, type is STREAM, is listening)
  |    +--------------------+                                                                              
  |                                                                                                        
  +--> if arg family is specified, check it as well                                                        
```

```
src/libsystemd/sd-event/sd-event.c                                         
+---------------------+                                                     
| sd_event_add_signal | : add signal source for event                       
+-|-------------------+                                                     
  |                                                                         
  |--> if arg sig & SD_EVENT_SIGNAL_PROCMASK                                
  |    -                                                                    
  |    +--> block = true                                                    
  |                                                                         
  |--> else                                                                 
  |    -                                                                    
  |    +--> block = false                                                   
  |                                                                         
  |--> if arg callback isn't provided                                       
  |    |                                                                    
  |    |               +----------------------+                             
  |    +--> callback = | signal_exit_callback |                             
  |                    +----------------------+                             
  |                                                                         
  |--> ensure event has signal source                                       
  |                                                                         
  |    +------------+                                                       
  |--> | source_new | prepare 'source' for event                            
  |    +------------+                                                       
  |                                                                         
  |--> if block == true                                                     
  |    |                                                                    
  |    |    +-----------------+                                             
  |    +--> | pthread_sigmask | block                                       
  |         +-----------------+                                             
  |    +------------------------+                                           
  |--> | event_make_signal_data | ensure target signal is monitored by epoll
  |    +------------------------+                                           
  |    +---------------------------------+                                  
  +--> | sd_event_source_set_description | duplicate description into source
       +---------------------------------+                                  
```

```
src/libsystemd/sd-event/sd-event.c                                              
+--------------------+                                                           
| sd_event_add_child | : prepare source monitoring child task                    
+-|------------------+                                                           
  |                                                                              
  |--> ensure we have callback                                                   
  |                                                                              
  |--> ensure hashmap in event exists                                            
  |                                                                              
  |--> alloc and set up source                                                   
  |                                                                              
  |--> if the source is about pid event                                          
  |    |                                                                         
  |    |    +-----------------------------+                                      
  |    +--> | source_child_pidfd_register | add to epoll                         
  |         +-----------------------------+                                      
  |                                                                              
  |--> else                                                                      
  |    |                                                                         
  |    |    +------------------------+                                           
  |    +--> | event_make_signal_data | ensure target signal is monitored by epoll
  |         +------------------------+                                           
  |    +-------------+                                                           
  +--> | hashmap_put | hashmap[pid] = source                                     
       +-------------+                                                           
```

```
src/libsystemd/sd-bus/sd-bus.c                                        
+-------------------+                                                  
| sd_bus_add_filter | : prepare slot for filter, prepend to list of bus
+-|-----------------+                                                  
  |    +-------------------+                                           
  |--> | bus_slot_allocate | prepapre slot and prepend to bus->slots   
  |    +-------------------+                                           
  |                                                                    
  +--> prepend filter to list of bus                                   
```

```
src/libsystemd/sd-bus/sd-bus.c                                                                                                  
+--------------+                                                                                                                 
| sd_bus_start | : set bus state = opening, prepare socket/epoll, send msg (hello) to "org.freedesktop.DBus"                     
+-|------------+                                                                                                                 
  |    +---------------+                                                                                                         
  |--> | bus_set_state | set bus state = opening                                                                                 
  |    +---------------+                                                                                                         
  |                                                                                                                              
  |--> if bus has input fd                                                                                                       
  |    |                                                                                                                         
  |    |    +--------------+                                                                                                     
  |    +--> | bus_start_fd | set up socket and start auth                                                                        
  |         +--------------+                                                                                                     
  |                                                                                                                              
  |--> elif bus has address                                                                                                      
  |    |                                                                                                                         
  |    |    +-------------------+                                                                                                
  |    +--> | bus_start_address | given bus address, spawn child if required, prepare socket & connect, register sources to epoll
  |         +-------------------+                                                                                                
  |    +----------------+                                                                                                        
  +--> | bus_send_hello | prepare msg of method call (hello), send to "org.freedesktop.DBus"                                     
       +----------------+                                                                                                        
```

```
src/libsystemd/sd-bus/sd-bus.c                           
+--------------+                                          
| bus_start_fd | : set up socket and start auth           
+-|------------+                                          
  |                                                       
  |--> set nonblock and cloexec on input fd               
  |                                                       
  |--> if output fd != input fd                           
  |    -                                                  
  |    +--> set nonblock and cloexec on output fd         
  |                                                       
  |    +--------------------+                             
  +--> | bus_socket_take_fd | set up socket and start auth
       +--------------------+                             
```

```
src/libsystemd/sd-bus/bus-convenience.c                                                             
+---------------------------+                                                                        
| sd_bus_match_signal_async | : send msg ('add match' method) to bus clients, add match rule to bus  
+-|-------------------------+                                                                        
  |    +-----------------+                                                                           
  |--> | make_expression | make a long string  = type + sender + path + interface + member           
  |    +-----------------+                                                                           
  |    +------------------------+                                                                    
  +--> | sd_bus_add_match_async | send msg ('add match' method) to bus clients, add match rule to bus
       +------------------------+                                                                    
```

```
src/libsystemd/sd-bus/bus-convenience.c                                                                                
+--------------------------+                                                                                            
| sd_bus_call_method_async | : prepare msg (method call), install callback, send msg out                                
+---------------------------+                                                                                           
| sd_bus_call_method_asyncv | : prepare msg (method call), install callback, send msg out                               
+-|-------------------------+                                                                                           
  |    +--------------------------------+                                                                               
  |--> | sd_bus_message_new_method_call | prepare msg of method_call type, append path/member/interface/destination info
  |    +--------------------------------+                                                                               
  |                                                                                                                     
  |--> if arg type is provided                                                                                          
  |    |                                                                                                                
  |    |    +------------------------+                                                                                  
  |    +--> | sd_bus_message_appendv | append type to msg                                                               
  |         +------------------------+                                                                                  
  |    +-------------------+                                                                                            
  +--> | sd_bus_call_async | prepare slot (install callback, insert to hashmap/prioq), send msg out                     
       +-------------------+                                                                                            
```

```
src/basic/log.c                                                                                                                 
+----------+                                                                                                                     
| log_open | : given target, ensure its fd is ready                                                                              
+-|--------+                                                                                                                     
  |                                                                                                                              
  +--> if pid == 1                                                                                                               
       |                                                                                                                         
       |--> if prohibit_ipc == false                                                                                             
       |    |                                                                                                                    
       |    |--> if log target is auto, journal, or kmsg                                                                         
       |    |    |                                                                                                               
       |    |    |    +------------------+                                                                                       
       |    |    |--> | log_open_journal | ensure journal fd exists (prepare socket and connect to "/run/systemd/journal/socket")
       |    |    |    +------------------+                                                                                       
       |    |    |                                                                                                               
       |    |    +--> close syslog and console, return                                                                           
       |    |                                                                                                                    
       |    +--> if log target is syslog, or kmsg                                                                                
       |         |                                                                                                               
       |         |    +-----------------+                                                                                        
       |         |--> | log_open_syslog | ensure syslog fd exists (prepare socket and connect to "/dev/log")                     
       |         |    +-----------------+                                                                                        
       |         |                                                                                                               
       |         +--> close journal and console, return                                                                          
       |                                                                                                                         
       |--> if target is auto, journal, syslog, or kmsg                                                                          
       |    |                                                                                                                    
       |    |    +---------------+                                                                                               
       |    +--> | log_open_kmsg | ensure kmsg fd exists (prepare socket and connect to "/dev/log")                              
       |         +---------------+                                                                                               
       |                                                                                                                         
       +--> close jorunal, syslog, console, and return                                                                           
```

```
src/basic/log.c                                                                                             
+------------------+                                                                                         
| log_open_journal | : ensure journal fd exists (prepare socket and connect to "/run/systemd/journal/socket")
+-|----------------+                                                                                         
  |                                                                                                          
  |--> if journal_fd exists already, return                                                                  
  |                                                                                                          
  |    +-------------------+                                                                                 
  |--> | create_log_socket | create socket fd for journal                                                    
  |    +-------------------+                                                                                 
  |    +-------------------+                                                                                 
  +--> | connect_unix_path | connect to "/run/systemd/journal/socket"                                        
       +-------------------+                                                                                 
```

```
src/core/main.c                                                               
+------------------+                                                           
| initialize_clock | : determine epoch and set clock time (realtime base)      
+-|----------------+                                                           
  |    +--------------------+                                                  
  |--> | clock_is_localtime | open /etc/adjtime to adjust time (not our case)  
  |    +--------------------+                                                  
  |                                                                            
  |--> if not in initrd                                                        
  |    |                                                                       
  |    |    +----------------------+                                           
  |    +--> | clock_reset_timewarp | do a dummy call to trigger time warp magic
  |         +----------------------+                                           
  |    +-------------------+                                                   
  |--> | clock_apply_epoch | determine epoch and set clock time (realtime base)
  |    +-------------------+                                                   
  |                                                                            
  +--> if now isn't correct and need to be forwarded or backwarded, log it     
```

```
src/core/main.c                                                                                                  
+--------+                                                                                                        
| part 0 | : setup mount, open log, init clock, make null stdio                                                   
+--------+                                                                                                        
  |                                                                                                               
  |    +-------+                                                                                                  
  |--> | prctl | set task name = 'systemd'                                                                        
  |    +-------+                                                                                                  
  |                                                                                                               
  +--> if our pid == 1 （system scope)                                                                             
       |    +-------------------+                                                                                 
       |--> | mount_setup_early | mount each entry in mount_table                                                 
       |    +-------------------+                                                                                 
       |    +-----------------------+                                                                             
       |--> | log_parse_environment | parse cmdline/env_vars, and log                                             
       |    +-----------------------+                                                                             
       |    +----------+                                                                                          
       |--> | log_open |                                                                                          
       |    +----------+                                                                                          
       |    +------------------+                                                                                  
       |--> | initialize_clock |                                                                                  
       |    +------------------+                                                                                  
       |    +----------------+                                                                                    
       |--> | log_set_target | set log target to journal or kmsg                                                  
       |    +----------------+                                                                                    
       |    +---------------------+                                                                               
       |--> | initialize_coredump | config coredump                                                               
       |    +---------------------+                                                                               
       |    +-----------------+                                                                                   
       |--> | make_null_stdio | assign '/dev/null' to in/out/err                                                  
       |    +-----------------+                                                                                   
       |    +------------+                                                                                        
       |--> | kmod_setup | (probably do nothing bc of disabled config)                                            
       |    +------------+                                                                                        
       |    +-------------+                                                                                       
       |--> | mount_setup | setup mount, symlink, and make a few folders under /run                               
       |    +-------------+                                                                                       
       |    +-------------------------+                                                                           
       +--> | lock_down_efi_variables | set attribute=immutable on '/sys/firmware/efi/efivars/LoaderSystemToken*' 
       |    +-------------------------+                                                                           
       |    +----------------------------+                                                                        
       +--> | cache_efi_options_variable | (probably do nothing bc of disabled config)                            
            +----------------------------+                                                                        
```

```
src/shared/mount-setup.c                               
+-------------------+                                   
| mount_setup_early | : mount each entry in mount_table 
+--------------------+                                  
| mount_points_setup | : mount each entry in mount_table
+-|------------------+                                  
  |                                                     
  +--> for each entry in mount_table                    
       |                                                
       |    +-----------+                               
       +--> | mount_one | mount one entry               
            +-----------+                               
```

```
src/shared/mount-setup.c                                   
+-----------+                                               
| mount_one | : mount one entry                             
+-|---------+                                               
  |                                                         
  |--> if ->condition_fn exists, call it                    
  |                                                         
  |    +---------------------+                              
  |--> | path_is_mount_point | check if path is a mountpoint
  |    +---------------------+                              
  |                                                         
  |--> if true, return                                      
  |    |                                                    
  |    |    +---------+                                     
  |    +--> | mkdir_p | ensure mount point is there         
  |         +---------+                                     
  |                                                         
  +--> mount                                                
```

```
src/basic/log.c                                                              
+-----------------------+                                                     
| log_parse_environment | : parse cmdline/env_vars, and log                   
+-|---------------------+                                                     
  |                                                                           
  |--> if we should parse proc cmdline                                        
  |    |                                                                      
  |    |    +--------------------+                                            
  |    +--> | proc_cmdline_parse | parse /proc/cmdline and log                
  |         +--------------------+                                            
  |    +---------------------------------+                                    
  +--> | log_parse_environment_variables | parse environment variables and log
       +---------------------------------+                                    
```

```
src/basic/proc-cmdline.c                                                         
+--------------------+                                                            
| proc_cmdline_parse | : parse /proc/cmdline and log                              
+-|------------------+                                                            
  |    +----------------------------+                                             
  |--> | proc_cmdline_strv_internal | read /proc/cmdline and tokenize the the data
  |    +----------------------------+                                             
  |    +-------------------------+                                                
  +--> | proc_cmdline_parse_strv | parse key=value and log                        
       +-------------------------+                                                
```

```
src/basic/fd-util.h                                     
+-----------------+                                      
| make_null_stdio | : assign arg fd[] to in/out/err      
+-----------------+                                      
| rearrange_stdio | : assign arg fd[] to in/out/err      
+-|---------------+                                      
  |                                                      
  |--> if any of the arg fd is negative                  
  |    |                                                 
  |    |--> null_fd = open '/dev/null'                   
  |    |                                                 
  |    +--> ensure it's not in range [0, 2]              
  |                                                      
  |--> for each arg fd                                   
  |    |                                                 
  |    |--> if it's < 0, fd[i] = null_fd                 
  |    |                                                 
  |    +--> elif it's in range [0, 2]                    
  |         -                                            
  |         +--> move it out of range, and fd[i] = new fd
  |                                                      
  |--> for each fd in array                              
  |    |                                                 
  |    |    +------+                                     
  |    +--> | dup2 | duplicate fd to the right idx       
  |         +------+                                     
  |                                                      
  |--> if arg fd is out of range [0, 2], close them      
  |                                                      
  +--> close copied fd[]                                 
```

```
src/shared/mount-setup.c                                                
+-------------+                                                          
| mount_setup | : setup mount, symlink, and make a few folders under /run
+-|-----------+                                                          
  |    +--------------------+                                            
  |--> | mount_points_setup | mount each entry in mount_table            
  |    +--------------------+                                            
  |    +-----------+                                                     
  |--> | dev_setup | generate a few symlinks in /dev                     
  |    +-----------+                                                     
  |                                                                      
  |--> remount '/' as shared                                             
  |                                                                      
  +--> make a few folders under /run                                     
```

```
src/core/main.c                                                                                 
+---------------------+                                                                          
| parse_configuration | : parse config_files & proc_cmdline & env_vars                           
+-|-------------------+                                                                          
  |    +-----------------+                                                                       
  |--> | reset_arguments | reset config values to default                                        
  |    +-----------------+                                                                       
  |    +-------------------+                                                                     
  |--> | parse_config_file | parse config files                                                  
  |    +-------------------+                                                                     
  |                                                                                              
  |--> if runtime scope == system                                                                
  |    |                                                                                         
  |    |    +--------------------+                                                               
  |    +--> | proc_cmdline_parse | parse /proc/cmdline                                           
  |         +--------------------+                                                               
  |    +-----------------------+                                                                 
  |--> | log_parse_environment | parse cmdline/env_vars, and log                                 
  |    +-----------------------+                                                                 
  |    +----------------------------+                                                            
  |--> | setenv_manager_environment | for each token in arg_manager_environment, set env variable
  |    +----------------------------+                                                            
  |    +-----------------------+                                                                 
  +--> | log_parse_environment | parse cmdline/env_vars, and log                                 
       +-----------------------+                                                                 
```

```
src/core/main.c                                                            
+-------------------+                                                       
| parse_config_file | : parse config files                                  
+-|-----------------+                                                       
  |                                                                         
  |--> if runtime scope == system                                           
  |    |                                                                    
  |    |    +--------------------------+                                    
  |    +--> | config_parse_config_file | parse config file and drop-in files
  |         +--------------------------+                                    
  |                                                                         
  +--> else (scope == user)                                                 
       -                                                                    
       +--> (skip)                                                          
```

```
src/shared/conf-parser.c                                                           
+--------------------------+                                                        
| config_parse_config_file | : parse config file and drop-in files                  
+-|------------------------+                                                        
  |                                                                                 
  |--> for each possible path                                                       
  |    -                                                                            
  |    +--> assemble string = path + config_file                                    
  |         e.g., /usr/lib/systemd/system.conf.d (only this path exists in our case)
  |                                                                                 
  |    +----------------------+                                                     
  |--> | conf_files_list_strv | find *.conf and return list                         
  |    +----------------------+ e.g.,                                               
  |                             00-systemd-conf.conf                                
  |                             40-hardware-watchdog.conf                           
  |                             service-restart-policy.conf                         
  |                                                                                 
  |--> assemble config_file                                                         
  |                                                                                 
  |    +-------------------------+                                                  
  +--> | config_parse_many_files | parse config file and drop-in files              
       +-------------------------+                                                  
```

```
src/basic/conf-files.c                                                                   
+----------------------+                                                                  
| conf_files_list_strv | : find *.conf and return list                                    
+-|--------------------+                                                                  
  |                                                                                       
  |--> for each possible folder                                                           
  |    |                                                                                  
  |    |    +-------------------+                                                         
  |    |--> | chase_and_opendir |                                                         
  |    |    +-------------------+                                                         
  |    |    +-----------+                                                                 
  |    +--> | files_add | for each file end with arg suffix, e.g., ".conf", add to hashmap
  |         +-----------+                                                                 
  |    +----------------------------------+                                               
  +--> | copy_and_sort_files_from_hashmap | sort and return string (files)                
       +----------------------------------+                                               
```

```
src/basic/proc-cmdline.c                                                         
+--------------------+                                                            
| proc_cmdline_parse | : parse /proc/cmdline                                      
+-|------------------+                                                            
  |                                                                               
  |--> (ignore efi part)                                                          
  |                                                                               
  |    +----------------------------+                                             
  |--> | proc_cmdline_strv_internal | read /proc/cmdline and tokenize the the data
  |    +----------------------------+                                             
  |    +-------------------------+                                                
  +--> | proc_cmdline_parse_strv | parse all kinds of arguments                   
       +-------------------------+                                                
```

```
src/basic/proc-cmdline.c                                           
+-------------------------+                                         
| proc_cmdline_parse_strv | : parse all kinds of arguments          
+-|-----------------------+                                         
  |                                                                 
  +--> for each token                                               
       |                                                            
       |    +-------------+                                         
       |--> | mangle_word | filter out initrd related token         
       |    +-------------+                                         
       |                                                            
       +--> call arg parse_item(), e.g.,                            
            +-------------------------+                             
            | parse_proc_cmdline_item | parse all kinds of arguments
            +-------------------------+                             
```

```
src/core/main.c                                                     
+------------------------+                                           
| setup_console_terminal | : reset terminal settings of /dev/consolem
+-|----------------------+                                           
  |    +------------------+                                          
  |--> | release_terminal | open /dev/tty and ioctl notty            
  |    +------------------+                                          
  |    +---------------+                                             
  +--> | console_setup | reset terminal settings of /dev/consolem    
       +---------------+                                             
```

```
src/core/main.c                                                                                
+---------------+                                                                               
| console_setup | : reset terminal settings of /dev/consolem                                    
+-|-------------+                                                                               
  |                                                                                             
  |--> open /dev/console                                                                        
  |                                                                                             
  |    +-------------------+                                                                    
  +--> | reset_terminal_fd | reset terminal settings                                            
  |    +-------------------+                                                                    
  |    +-----------------------+                                                                
  |--> | proc_cmdline_tty_size | get tty size from kernel cmdline (but it's not set in our case)
  |    +-----------------------+                                                                
  |                                                                                             
  +--> if got                                                                                   
       |                                                                                        
       |    +----------------------+                                                            
       +--> | terminal_set_size_fd |                                                            
            +----------------------+                                                            
```

```
src/core/main.c
+--------------------+                                                                                        
| initialize_runtime | : install crash handler for signals, setup machine_id, setup loopback, disable watchdog
+-|------------------+                                                                                        
  |                                                                                                           
  |--> if runtime scope == system                                                                             
  |    |                                                                                                      
  |    |    +-----------------------+                                                                         
  |    |--> | install_crash_handler | install crash handler for many related signals                          
  |    |    +-----------------------+                                                                         
  |    |    +--------------------------+                                                                      
  |    |--> | mount_cgroup_controllers | (skip)                                                               
  |    |    +--------------------------+                                                                      
  |    |    +-------------------+                                                                             
  |    |--> | os_release_status | parse os release info                                                       
  |    |    +-------------------+                                                                             
  |    |    +-------------------+                                                                             
  |    |--> | read_etc_hostname | read /etc/hostname                                                          
  |    |    +-------------------+                                                                             
  |    |    +------------------+                                                                              
  |    |--> | machine_id_setup | setup machine id                                                             
  |    |    +------------------+                                                                              
  |    |    +----------------+                                                                                
  |    |--> | loopback_setup | setup loopback                                                                 
  |    |    +----------------+                                                                                
  |    |    +------------------+                                                                              
  |    |--> | setup_os_release | copy /etc/os-release to /run/systemd/propagate/os-release                    
  |    |    +------------------+                                                                              
  |    |    +---------------------+                                                                           
  |    +--> | watchdog_set_device | disable watchdog                                                          
  |         +---------------------+                                                                           
  |    +---------------------+                                                                                
  +--> | make_reaper_process | do nothing if pid == 1                                                         
       +---------------------+                                                                                
```

```
src/shared/loopback-setup.c                         
+----------------+                                   
| loopback_setup | : setup loopback                  
+-|--------------+                                   
  |    +-----------------+                           
  |--> | sd_netlink_open | open route-netlink        
  |    +-----------------+                           
  |    +------------------+                          
  |--> | add_ipv4_address | add ipv4 addr of loopback
  |    +------------------+                          
  |    +------------------+                          
  |--> | add_ipv6_address | add ipv6 addr of loopback
  |    +------------------+                          
  |    +----------------+                            
  |--> | start_loopback |                            
  |    +----------------+                            
  |                                                  
  +--> wait till it's brought up                     
```

```
src/core/manager.c                                                                                                        
+-------------------------+                                                                                                
| manager_setup_run_queue | : prepare soucr of 'defer', which run and monitor jobs in runqueue                             
+-|-----------------------+                                                                                                
  |    +--------------------+                                                                                              
  |--> | sd_event_add_defer | prepare 'source' for defer, set pending = true                                               
  |    +--------------------+ +----------------------------+                                                               
  |                           | manager_dispatch_run_queue | run and invalidate jobs in runqueue, monitor those in progress
  |                           +----------------------------+                                                               
  |                                                                                                                        
  |    +-----------------------------+                                                                                     
  |--> | sd_event_source_set_enabled | enabled = off                                                                       
  |    +-----------------------------+                                                                                     
  |    +---------------------------------+                                                                                 
  +--> | sd_event_source_set_description | description = 'manager-run-queue'                                               
       +---------------------------------+                                                                                 
```

```
src/core/manager.c                                                                                    
+----------------------------+                                                                         
| manager_dispatch_run_queue | : run and invalidate jobs in runqueue, monitor those in progress        
+-|--------------------------+                                                                         
  |                                                                                                    
  |--> for each job in run_queue (priority sorted)                                                     
  |    |                                                                                               
  |    |    +------------------------+                                                                 
  |    +--> | job_run_and_invalidate | set job state = running, add it to manager, run unit accordingly
  |         +------------------------+                                                                 
  |                                                                                                    
  +--> if manager still has running job                                                                
       |                                                                                               
       |    +--------------------------------+                                                         
       +--> | manager_watch_jobs_in_progress | ensure manage has a event source for 'job in progress'  
            +--------------------------------+                                                         
```

```
src/core/job.c                                                                              
+------------------------+                                                                   
| job_run_and_invalidate | : set job state = running, add it to manager, run unit accordingly
+-|----------------------+                                                                   
  |    +--------------+                                                                      
  |--> | prioq_remove | remove job from run_queue                                            
  |    +--------------+                                                                      
  |    +-----------------+                                                                   
  |--> | job_start_timer | ensure job has a timer source                                     
  |    +-----------------+                                                                   
  |    +---------------+                                                                     
  |--> | job_set_state | state = running                                                     
  |    +---------------+                                                                     
  |    +-----------------------+                                                             
  |--> | job_add_to_dbus_queue | add job to dbus queue                                       
  |    +-----------------------+                                                             
  |                                                                                          
  +--> given job type (start, stop, ...), perform on unit accordingly                        
```

```
src/core/job.c                                                                                                
+-----------------+                                                                                            
| job_start_timer | : ensure job has a timer source                                                            
+-|---------------+                                                                                            
  |                                                                                                            
  |--> if job is running                                                                                       
  |    |                                                                                                       
  |    |--> if source isn't expired, return                                                                    
  |    |                                                                                                       
  |    |--> calculate timeout                                                                                  
  |    |                                                                                                       
  |    +--> if job has timer source                                                                            
  |         |                                                                                                  
  |         |    +--------------------------+                                                                  
  |         |--> | sd_event_source_set_time | update 'next timeout' of source                                  
  |         |    +--------------------------+                                                                  
  |         +--> return                                                                                        
  |                                                                                                            
  |--> else                                                                                                    
  |    |                                                                                                       
  |    |--> if job has timer source already, return                                                            
  |    |                                                                                                       
  |    +--> calculate timeout                                                                                  
  |                                                                                                            
  |    +-------------------+                                                                                   
  |--> | sd_event_add_time | set up timer fields of event (fd/queues), prepare 'source' and add to those queues
  |    +-------------------+ +--------------------+                                                            
  |                          | job_dispatch_timer | uninstall job and free it, commit log                      
  |                          +--------------------+                                                            
  |    +---------------------------------+                                                                     
  +--> | sd_event_source_set_description | 'job-start'                                                         
       +---------------------------------+                                                                     
```

```
src/core/job.c                                                      
+--------------------+                                               
| job_dispatch_timer | : uninstall job and free it, commit log       
+-|------------------+                                               
  |    +---------------------------+                                 
  |--> | job_finish_and_invalidate | uninstall job ad free it, notify
  |    +---------------------------+                                 
  |    +------------------+                                          
  +--> | emergency_action | given action, commit log accordingly     
       +------------------+                                          
```

```
src/core/job.c                                                                                                  
+---------------------------+                                                                                    
| job_finish_and_invalidate | : uninstall job ad free it, notify                                                 
+-|-------------------------+                                                                                    
  |                                                                                                              
  |--> job->result = arg result (e.g., timeout)                                                                  
  |                                                                                                              
  |--> if arg already == false                                                                                   
  |    |                                                                                                         
  |    |    +-----------------------+                                                                            
  |    +--> | job_emit_done_message | determine format, commit log, print to console                             
  |         +-----------------------+                                                                            
  |                                                                                                              
  |--> if result == done && type == restart                                                                      
  |    |                                                                                                         
  |    |--> set job_type = start, job_state = waiting                                                            
  |    |                                                                                                         
  |    +--> add job to dbus/run/gc queues                                                                        
  |                                                                                                              
  |    +---------------+                                                                                         
  |--> | job_uninstall | send signal (job removed) to busses, add unit to gc/dbus queues, remove job from manager
  |    +---------------+                                                                                         
  |    +----------+                                                                                              
  |--> | job_free | release job                                                                                  
  |    +----------+                                                                                              
  |    +---------------------+                                                                                   
  |--> | unit_trigger_notify | (skip)                                                                            
  |    +---------------------+                                                                                   
  |                                                                                                              
  +--> submit unit to upheld/bound/unneeded queues of manager                                                    
```

```
src/core/job.c                                                             
+-----------------------+                                                   
| job_emit_done_message | : determine format, commit log, print to console  
+-|---------------------+                                                   
  |    +-------------------------+                                          
  |--> | job_done_message_format | given unit/type/result, get format string
  |    +-------------------------+                                          
  |    +--------------------+                                               
  |--> | unit_status_string | get status string from unit                   
  |    +--------------------+                                               
  |                                                                         
  |--> commit log                                                           
  |                                                                         
  +--> print to console                                                     
```

```
src/core/job.c                                                                                                             
+---------------+                                                                                                           
| job_uninstall | : send signal (job removed) to busses, add unit to gc/dbus queues, remove job from manager                
+-|-------------+                                                                                                           
  |    +---------------+                                                                                                    
  |--> | job_set_state | waiting                                                                                            
  |    +---------------+                                                                                                    
  |                                                                                                                         
  |--> if not daemon-reload                                                                                                 
  |    |                                                                                                                    
  |    |    +-----------------------------+                                                                                 
  |    +--> | bus_job_send_removed_signal | send signal (job removed) to private bus, and api bus (if someone is interested)
  |         +-----------------------------+                                                                                 
  |    +----------------------+                                                                                             
  |--> | unit_add_to_gc_queue |                                                                                             
  |    +----------------------+                                                                                             
  |    +------------------------+                                                                                           
  |--> | unit_add_to_dbus_queue |                                                                                           
  |    +------------------------+                                                                                           
  |    +----------------------+                                                                                             
  |--> | hashmap_remove_value | remove job from manager                                                                     
  |    +----------------------+                                                                                             
  |                                                                                                                         
  +--> job->installed = false                                                                                               
```

```
src/core/dbus-job.c                                                                                                   
+-----------------------------+                                                                                        
| bus_job_send_removed_signal | : send signal (job removed) to private bus, and api bus (if someone is interested)     
+-|---------------------------+                                                                                        
  |                                                                                                                    
  |--> if no new signal sent                                                                                           
  |    |                                                                                                               
  |    |    +----------------------------+                                                                             
  |    +--> | bus_job_send_change_signal | send change signal of unit first, then send change signal of job            
  |         +----------------------------+                                                                             
  |    +-------------------------------------+                                                                         
  |--> | bus_unit_send_pending_change_signal | move unit from dbus queue to gc queue, send signal to private/api busses
  |    +-------------------------------------+                                                                         
  |    +-----------------+                                                                                             
  +--> | bus_foreach_bus | send signal to private bus, and api bus (if someone is interested)                          
       +-----------------+                                                                                             
```

```
src/core/dbus-job.c                                                                                                   
+----------------------------+                                                                                         
| bus_job_send_change_signal | : send change signal of unit first, then send change signal of job                      
+-|--------------------------+                                                                                         
  |    +-------------------------------------+                                                                         
  |--> | bus_unit_send_pending_change_signal | move unit from dbus queue to gc queue, send signal to private/api busses
  |    +-------------------------------------+                                                                         
  |                                                                                                                    
  |--> if job is in dbus queue                                                                                         
  |    -                                                                                                               
  |    +--> transfer to gc queue                                                                                       
  |                                                                                                                    
  |    +-----------------+                                                                                             
  +--> | bus_foreach_bus | send signal to private bus, and api bus (if someone is interested)                          
       +-----------------+                                                                                             
```

```
src/core/dbus-unit.c                                                                                             
+-------------------------------------+                                                                           
| bus_unit_send_pending_change_signal | : move unit from dbus queue to gc queue, send signal to private/api busses
+-----------------------------+-------+                                                                           
| bus_unit_send_change_signal | : move unit from dbus queue to gc queue, send signal to private/api busses        
+-|---------------------------+                                                                                   
  |                                                                                                               
  |--> if unit is in dbus queue                                                                                   
  |    |                                                                                                          
  |    |    +-------------+                                                                                       
  |    |--> | LIST_REMOVE | remove unit from dbus queue                                                           
  |    |    +-------------+                                                                                       
  |    |    +----------------------+                                                                              
  |    +--> | unit_add_to_gc_queue | add unit to gc queue                                                         
  |         +----------------------+                                                                              
  |    +-----------------+                                                                                        
  +--> | bus_foreach_bus | send signal to private bus, and api bus (if someone is interested)                     
       +-----------------+                                                                                        
```

```
src/core/dbus.c                                                                                  
+-----------------+                                                                               
| bus_foreach_bus | : send signal to private bus, and api bus (if someone is interested)          
+-|---------------+                                                                               
  |                                                                                               
  |--> for each private bus in manager                                                            
  |    -                                                                                          
  |    +--> call arg send_message(), e.g.,                                                        
  |         +---------------------+                                                               
  |         | send_changed_signal | send signal (properties changed) out twice: specific + generic
  |         +-----------------+---+                                                               
  |         | send_new_signal | send signal (UnitNew) out                                         
  |         +-----------------+                                                                   
  |                                                                                               
  +--> if sombody subscribed this                                                                 
       -                                                                                          
       +--> call arg send_message(), except we send to api bus this time                                                                                                
```

```
src/core/dbus-unit.c                                                                                    
+---------------------+                                                                                  
| send_changed_signal | : send signal (properties changed) out twice: specific + generic                 
+-|-------------------+                                                                                  
  |    +----------------+                                                                                
  |--> | unit_dbus_path | given unit, get object path                                                    
  |    +----------------+                                                                                
  |    +-------------------------------------+                                                           
  |--> | sd_bus_emit_properties_changed_strv | prepare msg of signal (PropertiesChanged),                
  |    +-------------------------------------+ append changed/invalidated properties to msg, send it out 
  |    +-------------------------------------+                                                           
  +--> | sd_bus_emit_properties_changed_strv | (the above is for specific type, this is for generic unit)
       +-------------------------------------+                                                           
```

```
src/core/dbus-unit.c                                                                             
+-----------------+                                                                               
| send_new_signal | : send signal (UnitNew) out                                                   
+-|---------------+                                                                               
  |    +----------------+                                                                         
  |--> | unit_dbus_path | given unit, get object path                                             
  |    +----------------+                                                                         
  |    +---------------------------+                                                              
  |--> | sd_bus_message_new_signal | prepare msg of signal type, append path/interface/member info
  |    +---------------------------+                                                              
  |    +-----------------------+                                                                  
  |--> | sd_bus_message_append | append unit_id and that object_path                              
  |    +-----------------------+                                                                  
  |    +-------------+                                                                            
  +--> | sd_bus_send | seal, send msg out or append to wqueue                                     
       +-------------+                                                                            
```

```
src/core/manager.c                                                                                  
+-------------+                                                                                      
| manager_new | : alloc manager, get default event, prepare all kinds of sources and callbacks       
+-|-----------+                                                                                      
  |                                                                                                  
  |--> alloc manager and init to default                                                             
  |                                                                                                  
  |    +-----------------------------+                                                               
  |--> | manager_default_environment | set default env (e.g., path, locale, ...)                     
  |    +-----------------------------+                                                               
  |    +----------------------+                                                                      
  |--> | manager_setup_prefix | (???)                                                                
  |    +----------------------+                                                                      
  |    +------------------+                                                                          
  |--> | sd_event_default | get default event for manager                                            
  |    +------------------+                                                                          
  |    +-------------------------+                                                                   
  |--> | manager_setup_run_queue | prepare soucr of 'defer', which run and monitor jobs in runqueue  
  |    +-------------------------+                                                                   
  |    +-----------------------+                                                                     
  |--> | manager_setup_signals | install handler for signal source                                   
  |    +-----------------------+                                                                     
  |    +----------------------+                                                                      
  |--> | manager_setup_cgroup | (skip)                                                               
  |    +----------------------+                                                                      
  |    +---------------------------+                                                                 
  |--> | manager_setup_time_change | prepare source and callback for time change                     
  |    +---------------------------+                                                                 
  |    +----------------------------+                                                                
  |--> | manager_read_timezone_stat | read /etc/localtime to set up localtime fields in manager      
  |    +----------------------------+                                                                
  |    +---------------------------+                                                                 
  |--> | manager_setup_time_change | prepare inotify and callback for /etc/localtime change          
  |    +---------------------------+                                                                 
  |    +------------------------------------+                                                        
  |--> | manager_setup_sigchld_event_source | prepare source and callback for signal 'child'         
  |    +------------------------------------+                                                        
  |    +--------------------------------------------+                                                
  |--> | manager_setup_memory_pressure_event_source | prepare source and callback for memory pressure
  |    +--------------------------------------------+                                                
  |                                                                                                  
  +--> mkdir /run/systemd/units                                                                      
```

```
src/core/manager.c                                                                                     
+-----------------------+                                                                               
| manager_setup_signals | : install handler for signal source                                           
+-|---------------------+                                                                               
  |                                                                                                     
  |--> add many signals to mask                                                                         
  |                                                                                                     
  |    +----------+                                                                                     
  |--> | signalfd | flags = non-block | clo-exec                                                        
  |    +----------+                                                                                     
  |    +-----------------+                                                                              
  |--> | sd_event_add_io | prepare 'source' and add to arg 'event, register the source's io             
  |    +-----------------+ +----------------------------+                                               
  |                        | manager_dispatch_signal_fd | read signal from fd, and handle it accordingly
  |                        +----------------------------+                                               
  |                                                                                                     
  |    +---------------------------------+                                                              
  |--> | sd_event_source_set_description | 'manager-signal'                                             
  |    +---------------------------------+                                                              
  |                                                                                                     
  +--> if manager is in system scope                                                                    
       |                                                                                                
       |    +------------------------+                                                                  
       +--> | enable_special_signals | enable SIGWINCH                                                  
            +------------------------+                                                                  
```

```
src/core/manager.c                                                                                       
+----------------------------+                                                                            
| manager_dispatch_signal_fd | : read signal from fd, and handle it accordingly                           
+-|--------------------------+                                                                            
  |                                                                                                       
  |--> read signal from fd                                                                                
  |                                                                                                       
  |--> switch signal                                                                                      
  |                                                                                                       
  |--> case sigchld                                                                                       
  |    +--> enable sigchld source of manager                                                              
  |--> case sigterm, sigint                                                                               
  |    -    +-----------------------------+                                                               
  |    +--> | manager_handle_ctrl_alt_del | load unit from manager, add job to transaction and activate it
  |         +-----------------------------+                                                               
  |--> case sigwinch                                                                                      
  |    -    +-----------------------+                                                                     
  |    +--> | manager_start_special | load unit from manager, add job to transaction and activate it      
  |         +-----------------------+                                                                     
  |--> case sigpwr                                                                                        
  |    -    +-----------------------+                                                                     
  |    +--> | manager_start_special | load unit from manager, add job to transaction and activate it      
  |         +-----------------------+                                                                     
  +--> case sigusr                                                                                        
       -                                                                                                  
       +--> blabla                                                                                        
```

```
src/core/manager.c                                                                                       
+-----------------------------+                                                                           
| manager_handle_ctrl_alt_del | : load unit from manager, add job to transaction and activate it          
+-----------------------+-----+                                                                           
| manager_start_special | : load unit from manager, add job to transaction and activate it                
+-|---------------------+                                                                                 
  |    +----------------------------------+                                                               
  +--> | manager_add_job_by_name_and_warn | load unit from manager, add job to transaction and activate it
       +----------------------------------+                                                               
```

```
src/core/manager.c                                                                                  
+----------------------------------+                                                                 
| manager_add_job_by_name_and_warn | : load unit from manager, add job to transaction and activate it
+----------------------------------+                                                                 
| manager_add_job_by_name | : load unit from manager, add job to transaction and activate it         
+-|-----------------------+                                                                          
  |    +-------------------+                                                                         
  |--> | manager_load_unit | ensure unit is in load_queue of manager, load each unit from there      
  |    +-------------------+                                                                         
  |    +-----------------+                                                                           
  +--> | manager_add_job | alloc transaction, add job to it, activate transaction, free it           
       +-----------------+                                                                           
```

```
src/core/manager.c                                                                                
+-------------------+                                                                              
| manager_load_unit | : ensure unit is in load_queue of manager, load each unit from there         
+-|-----------------+                                                                              
  |    +---------------------------+                                                               
  |--> | manager_load_unit_prepare | ensure unit is in manager                                     
  |    +---------------------------+                                                               
  |    +-----------------------------+                                                             
  +--> | manager_dispatch_load_queue | for each unit in load_queue, load it and add to other queues
       +-----------------------------+                                                             
```

```
src/core/manager.c                                            
+---------------------------+                                  
| manager_load_unit_prepare | : ensure unit is in manager      
+-|-------------------------+                                  
  |    +------------------+                                    
  |--> | manager_get_unit | given name, get unit from manager  
  |    +------------------+                                    
  |                                                            
  |--> if unit found                                           
  |    -                                                       
  |    +--> if it's already loaded, return                     
  |                                                            
  |--> else                                                    
  |    |                                                       
  |    |    +----------+                                       
  |    +--> | unit_new |                                       
  |         +----------+                                       
  |    +---------------+                                       
  |--> | unit_add_name | add (name, unit) to hashmap of manager
  |    +---------------+                                       
  |                                                            
  +--> add unit to load/dbus/gc queues                         
```

```
src/core/manager.c                                                                           
+-----------------------------+                                                               
| manager_dispatch_load_queue | : for each unit in load_queue, load it and add to other queues
+-|---------------------------+                                                               
  |                                                                                           
  |--> while manager still has something in load_queue                                        
  |    |                                                                                      
  |    |    +-----------+                                                                     
  |    +--> | unit_load | remove unit from manager, call ->load(), add unit to queues         
  |         +-----------+                                                                     
  |    +------------------------------------+                                                 
  +--> | manager_dispatch_target_deps_queue | for unit in target_deps_queue, add dependencies 
       +------------------------------------+                                                 
```

```
src/core/unit.c                                                           
+-----------+                                                              
| unit_load | : remove unit from manager, call ->load(), add unit to queues
+-|---------+                                                              
  |                                                                        
  |--> remove unit from manager                                            
  |                                                                        
  |--> if unit->transient_file                                             
  |    -                                                                   
  |    +--> flush file and close it                                        
  |                                                                        
  |--> call ->load(), e.g.,                                                
  |    +-------------+                                                     
  |    | socket_load | merge socket unit, handle dependencies              
  |    +-------------+                                                     
  |                                                                        
  |--> if unit state == loaded, add more dependencies                      
  |                                                                        
  +--> add unit to dbus/gc queues                                          
```

```
src/core/socket.c                                                                             
+-------------+                                                                                
| socket_load | : merge socket unit, handle dependencies                                       
+-|-----------+                                                                                
  |    +-------------------------------+                                                       
  |--> | unit_load_fragment_and_dropin | load unit fragment (itself) and dropin files (others)
  |    +-------------------------------+                                                       
  |    +-------------------+                                                                   
  +--> | socket_add_extras | handle extra dependencies                                         
       +-------------------+                                                                   
```

```
src/core/unit.c                                                                                        
+-------------------------------+                                                                       
| unit_load_fragment_and_dropin | : load unit fragment (itself) and dropin files (others)
+-|-----------------------------+                                                                       
  |    +--------------------+                                                                           
  +--> | unit_load_fragment | load a .{service, socket, ...} file
  |    +--------------------+                                                                           
  |    +------------------+                                                                             
  +--> | unit_load_dropin | read files (.requires/.wants/.upholds) to figure out dependencies and add other units to load queue
       +------------------+                                                                             
```

```
src/core/load-fragment.c                                                                  
+--------------------+                                                                     
| unit_load_fragment | : load a .{service, socket, ...} file
+-|------------------+                                                                     
  |                                                                                        
  |--> if unit->transient, return                                                          
  |                                                                                        
  |    +--------------------------+                                                        
  |--> | unit_file_build_name_map | build two maps for unit->main and unit->aliases queries
  |    +--------------------------+                                                        
  |    +-------------------------+                                                         
  |--> | unit_file_find_fragment | given unit id, find fragment/names                      
  |    +-------------------------+                                                         
  |                                                                                        
  |--> if fragment found                                                                   
  |    |                                                                                   
  |    |--> open fragment (file)                                                           
  |    |                                                                                   
  |    |    +--------------+                                                               
  |    +--> | config_parse | parse file contents                                           
  |         +--------------+                                                               
  |    +----------------+                                                                  
  +--> | merge_by_names | merge units by names                                             
       +----------------+                                                                  
```

```
src/core/manager.c                                                                          
+-----------------+                                                                          
| manager_add_job | : alloc transaction, add job to it, activate transaction, free it        
+-|---------------+                                                                          
  |    +-----------------+                                                                   
  |--> | transaction_new | alloc transaction                                                 
  |    +-----------------+                                                                   
  |    +--------------------------------------+                                              
  |--> | transaction_add_job_and_dependencies | alloc job and add to transaction             
  |    +--------------------------------------+                                              
  |                                                                                          
  |--> if mode == isolate                                                                    
  |    |                                                                                     
  |    |    +------------------------------+                                                 
  |    +--> | transaction_add_isolate_jobs | (skip)                                          
  |         +------------------------------+                                                 
  |                                                                                          
  |--> if mode == triggering                                                                 
  |    |                                                                                     
  |    |    +---------------------------------+                                              
  |    +--> | transaction_add_triggering_jobs | (skip)                                       
  |         +---------------------------------+                                              
  |    +----------------------+                                                              
  |--> | transaction_activate | find important jobs & merge them, add to manager for handling
  |    +----------------------+                                                              
  |    +------------------+                                                                  
  +--> | transaction_free | remove transaction from hashmap, release it                      
       +------------------+                                                                  
```

```
src/core/transaction.c                                                                 
+----------------------+                                                                
| transaction_activate | : find important jobs & merge them, add to manager for handling
+-|--------------------+                                                                
  |    +---------------------------------------------+                                  
  |--> | transaction_find_jobs_that_matter_to_anchor | find jobs that matter            
  |    +---------------------------------------------+                                  
  |    +----------------------------+                                                   
  |--> | transaction_drop_redundant | drop redundant jobs                               
  |    +----------------------------+                                                   
  |                                                                                     
  |--> endless loop                                                                     
  |    |                                                                                
  |    |    +-----------------------------+                                             
  |    |--> | transaction_collect_garbage | remove unneeded jobs                        
  |    |    +-----------------------------+                                             
  |    |    +--------------------------+                                                
  |    |--> | transaction_verify_order |                                                
  |    |    +--------------------------+                                                
  |    |                                                                                
  |    +--> break if everything is good                                                 
  |                                                                                     
  |--> endless loop                                                                     
  |    |                                                                                
  |    |    +------------------------+                                                  
  |    |--> | transaction_merge_jobs | merge jobs                                       
  |    |    +------------------------+                                                  
  |    |    +-----------------------------+                                             
  |    +--> | transaction_collect_garbage | remove unneeded jobs                        
  |         +-----------------------------+                                             
  |    +----------------------------+                                                   
  |--> | transaction_drop_redundant | drop redundant jobs again                         
  |    +----------------------------+                                                   
  |    +-------------------+                                                            
  +--> | transaction_apply | add jobs of transaction to manager                         
       +-------------------+                                                            
```

```
src/core/manager.c                                             
+-------------------+                                           
| manager_setup_bus | : ensure api bus of manager is initialized
+-|-----------------+                                           
  |                                                             
  +--> if dbus is up                                            
       |                                                        
       |    +--------------+                                    
       |--> | bus_init_api | ensure api bus is initialized      
       |    +--------------+                                    
       |                                                        
       +--> if manager is in system cscope                      
            |                                                   
            |    +-----------------+                            
            +--> | bus_init_system | (do nothing in our case)   
                 +-----------------+                            
```

```
src/core/dbus.c                                                                                  
+--------------+                                                                                  
| bus_init_api | : ensure api bus is initialized                                                  
+-|------------+                                                                                  
  |                                                                                               
  |--> if api bus of manager is up, return                                                        
  |                                                                                               
  |--> if system bus of manager is up, get bus                                                    
  |                                                                                               
  |--> else                                                                                       
  |    |                                                                                          
  |    |    +-------------------------------------+                                               
  |    |--> | sd_bus_open_system_with_description | prepare bus, determine address and save in bus
  |    |    +-------------------------------------+                                               
  |    |    +---------------------+                                                               
  |    |--> | sd_bus_attach_event | prepare all kinds of sources for event                        
  |    |    +---------------------+                                                               
  |    |    +------------------------------+                                                      
  |    +--> | bus_setup_disconnected_match | register callback for dbus signal 'disconnedted'     
  |         +------------------------------+                                                      
  |    +---------------+                                                                          
  +--> | bus_setup_api | setup vtables, install bus_match/signal_match, request name              
       +---------------+                                                                          
```

```
src/core/dbus.c                                                                                                 
+------------------------------+                                                                                 
| bus_setup_disconnected_match | : register callback for dbus signal 'disconnedted'                              
+-|----------------------------+                                                                                 
  |    +---------------------------+                                                                             
  +--> | sd_bus_match_signal_async | send msg ('add match' method) to bus clients, add match rule to bus         
       +---------------------------+ +---------------------+                                                     
                                     | signal_disconnected | get bus from msg, destroy it and remove from manager
                                     +---------------------+                                                     
```

```
src/core/dbus.c                                                                                                                           
+---------------+                                                                                                                          
| bus_setup_api | : setup vtables, install bus_match/signal_match, request name                                                            
+-|-------------+                                                                                                                          
  |    +-----------------------+                                                                                                           
  |--> | bus_setup_api_vtables | setup vtables of bus_manager and manager_log_control                                                      
  |    +-----------------------+                                                                                                           
  |                                                                                                                                        
  |--> for each watch_bus in manager                                                                                                       
  |    |                                                                                                                                   
  |    |    +------------------------+                                                                                                     
  |    +--> | unit_install_bus_match | add match, get name owner                                                                           
  |         +------------------------+ +---------------------------+                                                                       
  |                                    | signal_name_owner_changed | read new owner from reply, call ->bus_name_owner_change() if it exists
  |                                    +---------------------------+                                                                       
  |    +---------------------------+                                                                                                       
  |--> | sd_bus_match_signal_async | send msg ('add match' method) to bus clients, add match rule to bus                                   
  |    +---------------------------+ +---------------------------+                                                                         
  |                                  | signal_activation_request | given name from msg, load its unit and add job to it                    
  |                                  +---------------------------+                                                                         
  |    +---------------------------+                                                                                                       
  |--> | sd_bus_request_name_async | prepare msg (method call, request_name), install callback, send msg out                               
  |    +---------------------------+                                                                                                       
  |    +----------------------------+                                                                                                      
  +--> | bus_register_malloc_status | prepare match rule and callback for malloc status                                                    
       +----------------------------+                                                                                                      
```

```
src/core/dbus.c                                                                                                            
+-----------------------+                                                                                                   
| bus_setup_api_vtables | : setup vtables of bus_manager and manager_log_control                                            
+-|---------------------+                                                                                                   
  |    +------------------------+                                                                                           
  |--> | bus_add_implementation | add implementation (vtable, fallback, node enumerator, manager), apply to children as well
  |    +------------------------+ (bus_manager_object)                                                                      
  |                                                                                                                         
  |    +------------------------+                                                                                           
  +--> | bus_add_implementation | (manager_log_control_object)                                                              
       +------------------------+                                                                                           
```

```
src/shared/bus-object.c                                                                                                   
+------------------------+                                                                                                 
| bus_add_implementation | : add implementation (vtable, fallback, node enumerator, manager), apply to children as well    
+-|----------------------+                                                                                                 
  |                                                                                                                        
  |--> for each vtable in vtables                                                                                          
  |    |                                                                                                                   
  |    |    +--------------------------+                                                                                   
  |    +--> | sd_bus_add_object_vtable | prepare node of path, add each vtable to bus hashmap, prepare slot and add to node
  |         +--------------------------+                                                                                   
  |                                                                                                                        
  |--> if there's node enumerator                                                                                          
  |    |                                                                                                                   
  |    |    +----------------------------+                                                                                 
  |    +--> | sd_bus_add_node_enumerator | alloc node/slot, link them                                                      
  |         +----------------------------+                                                                                 
  |                                                                                                                        
  |--> if there's manager                                                                                                  
  |    |                                                                                                                   
  |    |    +---------------------------+                                                                                  
  |    +--> | sd_bus_add_object_manager | prepare node of path, prepare slot and add to node                               
  |         +---------------------------+                                                                                  
  |                                                                                                                        
  +--> for each child                                                                                                      
       |                                                                                                                   
       |    +------------------------+                                                                                     
       +--> | bus_add_implementation | (recursive)                                                                         
            +------------------------+                                                                                     
```

```
src/core/unit.c                                                                                                                       
+------------------------+                                                                                                             
| unit_install_bus_match | : add match, get name owner                                                                                 
+-|----------------------+                                                                                                             
  |    +--------------------+                                                                                                          
  |--> | bus_add_match_full | send msg ('add match' method) to bus clients, add match rule to bus                                      
  |    +--------------------+ +---------------------------+                                                                            
  |                           | signal_name_owner_changed | read new owner from reply, call ->bus_name_owner_change() if it exists     
  |                           +---------------------------+                                                                            
  |                                                                                                                                    
  |    +--------------------------------+                                                                                              
  |--> | sd_bus_message_new_method_call | prepare msg of method_call type (GetNameOwner), append path/member/interface/destination info
  |    +--------------------------------+                                                                                              
  |    +-----------------------+                                                                                                       
  |--> | sd_bus_message_append | append name                                                                                           
  |    +-----------------------+                                                                                                       
  |    +-------------------+                                                                                                           
  +--> | sd_bus_call_async | prepare slot (install callback, insert to hashmap/prioq), send msg out                                    
       +-------------------+ +------------------------+                                                                                
                             | get_name_owner_handler | get new name_owner from msg                                                    
                             +------------------------+                                                                                
```

```
src/core/unit.c                                                                                      
+---------------------------+                                                                         
| signal_name_owner_changed | : read new owner from reply, call ->bus_name_owner_change() if it exists
+-|-------------------------+                                                                         
  |    +---------------------+                                                                        
  |--> | sd_bus_message_read | read data (type = string,string,string) from reply                     
  |    +---------------------+                                                                        
  |                                                                                                   
  +--> if ->bus_name_owner_change exists                                                              
       -                                                                                              
       +--> call it, e.g.,                                                                            
            +-------------------------------+                                                         
            | service_bus_name_owner_change | (skip)                                                  
            +-------------------------------+                                                         
```

```
src/core/unit.c                                        
+------------------------+                              
| get_name_owner_handler | : get new name_owner from msg
+-|----------------------+                              
  |    +---------------------+                          
  |--> | sd_bus_message_read | read new owner from msg  
  |    +---------------------+                          
  |                                                     
  +--> if ->bus_name_owner_change() exists              
       -                                                
       +--> call it                                     
```

```
src/core/dbus.c                                                                               
+---------------------------+                                                                  
| signal_activation_request | : given name from msg, load its unit and add job to it           
+-|-------------------------+                                                                  
  |    +---------------------+                                                                 
  |--> | sd_bus_message_read | read name from msg                                              
  |    +---------------------+                                                                 
  |    +-------------------+                                                                   
  |--> | manager_load_unit | ensure unit is in load_queue of manager, load each unit from there
  |    +-------------------+                                                                   
  |    +-----------------+                                                                     
  +--> | manager_add_job | alloc transaction, add job to it, activate transaction, free it     
       +-----------------+                                                                     
```

```
src/libsystemd/sd-bus/bus-control.c                                                         
+---------------------------+                                                                
| sd_bus_request_name_async | : prepare msg (method call), install callback, send msg out    
+-|-------------------------+                                                                
  |    +----------------------------------+                                                  
  |--> | validate_request_name_parameters |                                                  
  |    +----------------------------------+                                                  
  |    +--------------------------+                                                          
  +--> | sd_bus_call_method_async | prepare msg (method call), install callback, send msg out
       +--------------------------+                                                          
```

```
src/shared/bus-util.c                                                                                                  
+----------------------------+                                                                                          
| bus_register_malloc_status | : prepare match rule and callback for malloc status                                      
+-|--------------------------+                                                                                          
  |                                                                                                                     
  |--> prepare match rule                                                                                               
  |                                                                                                                     
  |    +------------------------+                                                                                       
  +--> | sd_bus_add_match_async | send msg ('add match' method) to bus clients, add match rule to bus                   
       +------------------------+ +--------------------------------+                                                    
                                  | method_dump_memory_state_by_fd | get fd from dump data and reply msg (method return)
                                  +--------------------------------+                                                    
```

```
src/shared/bus-util.c                                                                  
+--------------------------------+                                                      
| method_dump_memory_state_by_fd | : get fd from dump data and reply msg (method return)
+-|------------------------------+                                                      
  |    +----------------+                                                               
  |--> | memstream_init | init mem_stream                                               
  |    +----------------+                                                               
  |    +-------------+                                                                  
  +--> | malloc_info | get malloc info (?)                                              
  |    +-------------+                                                                  
  |    +--------------------+                                                           
  |--> | memstream_finalize | finalize mem_stream                                       
  |    +--------------------+                                                           
  |    +-----------------+                                                              
  |--> | acquire_data_fd | get fd from dump data                                        
  |    +-----------------+                                                              
  |    +----------------------------+                                                   
  +--> | sd_bus_reply_method_return | reply msg of 'method return'                      
       +----------------------------+                                                   
```

```
src/core/dbus.c                                         
+-----------------+                                      
| bus_init_system | : (do nothing in our case)           
+-|---------------+                                      
  |                                                      
  |--> if system bus is ready, return                    
  |                                                      
  |--> if manager is in system cscope && api bus is ready
  |    -                                                 
  |    +--> get api bus                                  
  |                                                      
  |--> else                                              
  |    -                                                 
  |    +--> (skip, probably not our case)                
  |                                                      
  |    +------------------+                              
  +--> | bus_setup_system | (do nothing in our case)     
       +------------------+                              
```

```
src/core/main.c                                                                                      
+-----------------+                                                                                   
| manager_startup | : set up and start manager, ensure journal and dbus are ready                     
+-|---------------+                                                                                   
  |    +---------------------------+                                                                  
  |--> | lookup_paths_init_or_warn | init lookup path                                                 
  |    +---------------------------+                                                                  
  |    +------------------------------------+                                                         
  |--> | manager_run_environment_generators | determine paths and execute them (?)                    
  |    +------------------------------------+                                                         
  |    +--------------------+                                                                         
  |--> | manager_preset_all | determin presets and execute them (?)                                   
  |    +--------------------+                                                                         
  |    +-----------------------------+                                                                
  |--> | manager_enumerate_perpetual | for each supported unit type, call its ->enumerate_perpetual() 
  |    +-----------------------------+                                                                
  |    +-------------------+                                                                          
  |--> | manager_enumerate | for each supported unit type, call its ->enumerate(), dispatch load queue
  |    +-------------------+                                                                          
  |    +------------------------+                                                                     
  |--> | manager_distribute_fds | for each unit in manager, call ->distribute_fds()                   
  |    +------------------------+                                                                     
  |    +----------------------+                                                                       
  |--> | manager_setup_notify | ensure manager has notify fd and source (callback)                    
  |    +----------------------+                                                                       
  |    +-------------------+                                                                          
  |--> | manager_setup_bus | ensure api bus of manager is initialized                                 
  |    +-------------------+                                                                          
  |    +----------------------+                                                                       
  |--> | manager_varlink_init | alloc and setup varlink server                                        
  |    +----------------------+                                                                       
  |    +------------------+                                                                           
  |--> | manager_coldplug | for each unit in manager: uninstall and release its jobs                  
  |    +------------------+                                                                           
  |    +----------------+                                                                             
  |--> | manager_vacuum | clean up runtime objects                                                    
  |    +----------------+                                                                             
  |    +---------------+                                                                              
  +--> | manager_ready | ensure journal and dbus are ready, touch file 'systemd-units-load'           
       +---------------+                                                                              
```

```
src/core/core-varlink.c                                                                                                    
+----------------------+                                                                                                    
| manager_varlink_init | : alloc and setup varlink server                                                                   
+-----------------------------+                                                                                             
| manager_varlink_init_system | : alloc and setup varlink server                                                            
+-|---------------------------+                                                                                             
  |                                                                                                                         
  |--> if server is ready, return                                                                                           
  |                                                                                                                         
  |    +------------------------------+                                                                                     
  |--> | manager_setup_varlink_server | alloc and setup varlink server, install several (method, callback)                  
  |    +------------------------------+                                                                                     
  |                                                                                                                         
  |--> if manager isn't for test run (our case)                                                                             
  |    |                                                                                                                    
  |    |--> mkdir of '/run/systemd/userdb'                                                                                  
  |    |                                                                                                                    
  |    |    +-------------------------------+                                                                               
  |    |--> | varlink_server_listen_address | get socket and set addr, listen, alloc server_socket and add to varlink_server
  |    |    +-------------------------------+ (/run/systemd/userdb/io.systemd.DynamicUser)                                  
  |    |    +-------------------------------+                                                                               
  |    +--> | varlink_server_listen_address | get socket and set addr, listen, alloc server_socket and add to varlink_server
  |         +-------------------------------+ (/run/systemd/io.system.ManagedOOM)                                           
  |    +-----------------------------+                                                                                      
  +--> | varlink_server_attach_event | for each server_socket in varlink_server: register io source                         
       +-----------------------------+                                                                                      
```

```
src/core/core-varlink.c                                                                             
+------------------------------+                                                                     
| manager_setup_varlink_server | : alloc and setup varlink server, install several (method, callback)
+-|----------------------------+                                                                     
  |    +--------------------+                                                                        
  |--> | varlink_server_new | alloc varlink server                                                   
  |    +--------------------+                                                                        
  |    +-----------------------------+                                                               
  |--> | varlink_server_set_userdata | save userdata (manager) in server                             
  |    +-----------------------------+                                                               
  |    +---------------------------------+                                                           
  |--> | varlink_server_bind_method_many | add (method, callback) to hashmap of server               
  |    +---------------------------------+                                                           
  |    +--------------------------------+                                                            
  +--> | varlink_server_bind_disconnect | install 'vl_disconnect()' to server                        
       +--------------------------------+                                                            
```

```
src/shared/varlink.h                                                                     
+---------------------------------+                                                       
| varlink_server_bind_method_many | : add (method, callback) to hashmap of server         
+---------------------------------+--------+                                              
| varlink_server_bind_method_many_internal | : add (method, callback) to hashmap of server
+-|----------------------------------------+                                              
  |                                                                                       
  |--> get method and callback from va_args                                               
  |                                                                                       
  |    +----------------------------+                                                     
  +--> | varlink_server_bind_method | add (method, callback) to hashmap of server         
       +----------------------------+                                                     
```

```
src/shared/varlink.c                                                                                             
+-------------------------------+                                                                                 
| varlink_server_listen_address | : get socket and set addr, listen, alloc server_socket and add to varlink_server
+-|-----------------------------+                                                                                 
  |    +----------------------+                                                                                   
  |--> | sockaddr_un_set_path | e.g., /run/systemd/userdb/io.systemd.DynamicUser                                  
  |    +----------------------+                                                                                   
  |    +--------+                                                                                                 
  |--> | socket |                                                                                                 
  |    +--------+                                                                                                 
  |    +--------------------+                                                                                     
  |--> | sockaddr_un_unlink | ensure the path is null terminated                                                  
  |    +--------------------+                                                                                     
  |    +--------+                                                                                                 
  |--> | listen |                                                                                                 
  |    +--------+                                                                                                 
  |    +----------------------------------------+                                                                 
  |--> | varlink_server_create_listen_fd_socket | alloc and setup server_socket                                   
  |    +----------------------------------------+                                                                 
  |    +--------------+                                                                                           
  +--> | LIST_PREPEND | add server_socket to varlink_server                                                       
       +--------------+                                                                                           
```

```
src/shared/varlink.c
+----------------------------------------+                                                      
| varlink_server_create_listen_fd_socket | : alloc and setup server_socket                      
+-|--------------------------------------+                                                      
  |                                                                                             
  |--> alloc server_socket                                                                      
  |                                                                                             
  |--> save server and fd in it                                                                 
  |                                                                                             
  +--> if varlink_server has event                                                              
       |                                                                                        
       |    +-----------------+                                                                 
       |--> | sd_event_add_io | prepare 'source' and add to arg 'event, register the source's io
       |    +-----------------+                                                                 
       |    +------------------------------+                                                    
       +--> | sd_event_source_set_priority |                                                    
            +------------------------------+                                                    
```

```
src/shared/varlink.c                                                                            
+-----------------------------+                                                                  
| varlink_server_attach_event | : for each server_socket in varlink_server: register io source   
+-|---------------------------+                                                                  
  |                                                                                              
  |--> ensure varlink_esrver has event                                                           
  |                                                                                              
  |--> for each server_socket in varlink_server                                                  
  |    |                                                                                         
  |    |    +----------------------------------------+                                           
  |    +--> | varlink_server_add_socket_event_source | prepare io source for varlink_server event
  |         +----------------------------------------+                                           
  |                                                                                              
  +--> save priority in varlink_server                                                           
```

```
src/shared/varlink.c                                                                      
+----------------------------------------+                                                 
| varlink_server_add_socket_event_source | : prepare io source for varlink_server event    
+-|--------------------------------------+                                                 
  |    +-----------------+                                                                 
  |--> | sd_event_add_io | prepare 'source' and add to arg 'event, register the source's io
  |    +-----------------+ +------------------+                                            
  |                        | connect_callback | (skip)                                     
  |                        +------------------+                                            
  |    +------------------------------+                                                    
  +--> | sd_event_source_set_priority |                                                    
       +------------------------------+                                                    
```

```
src/core/manager.c                                                            
+------------------+                                                           
| manager_coldplug | : for each unit in manager: uninstall and release its jobs
+-|----------------+                                                           
  |                                                                            
  +--> for each unit in manager                                                
       |                                                                       
       |    +---------------+                                                  
       +--> | unit_coldplug | uninstall and release jobs in unit               
            +---------------+                                                  
```

```
src/core/unit.c                                                                         
+---------------+                                                                        
| unit_coldplug | : uninstall and release jobs in unit                                   
+-|-------------+                                                                        
  |                                                                                      
  |--> if ->coldplug() exists                                                            
  |    -                                                                                 
  |    +--> call it                                                                      
  |                                                                                      
  +--> if unit->job                                                                      
  |    |                                                                                 
  |    |    +--------------+                                                             
  |    +--> | job_coldplug | move job to gc queue, set timer to uninstall and release job
  |         +--------------+                                                             
  |                                                                                      
  +--> if unit->nop_job                                                                  
       |                                                                                 
       |    +--------------+                                                             
       +--> | job_coldplug | move job to gc queue, set timer to uninstall and release job
            +--------------+                                                             
```

```
src/core/manager.c                                                                                                
+---------------+                                                                                                  
| manager_ready | : ensure journal and dbus are ready, touch file 'systemd-units-load'                             
+-|-------------+                                                                                                  
  |                                                                                                                
  |--> ->objective = manager_ok                                                                                    
  |                                                                                                                
  |    +-------------------------+                                                                                 
  |--> | manager_recheck_journal | given target, ensure its fd is ready                                            
  |    +-------------------------+                                                                                 
  |    +----------------------+                                                                                    
  |--> | manager_recheck_dbus | ensure api bus is initialized                                                      
  |    +----------------------+                                                                                    
  |    +-----------------+                                                                                         
  |--> | manager_catchup | (skip)                                                                                  
  |    +-----------------+                                                                                         
  |                                                                                                                
  +--> if manager is in system scope                                                                               
       -                                                                                                           
       +--> touch "/run/systemd/systemd-units-load" to indiate when the manager started loading units the last time
```

```
src/core/manager.c                                          
+----------------------+                                     
| manager_recheck_dbus | : ensure api bus is initialized     
+-|--------------------+                                     
  |                                                          
  +--> if manager dbus is running                            
       |                                                     
       |    +-------------+                                  
       |--> | bus_init_api| ensure api bus is initialized    
       |    +-------------+                                  
       |                                                     
       +--> if manager is in system scope                    
            |                                                
            |    +-----------------+                         
            +--> | bus_init_system | (do nothing in our case)
                 +-----------------+                         
```

```
src/core/main.c                                                                                                                    
+------+                                                                                                                            
| main |                                                                                                                            
+-|----+                                                                                                                            
  |    +--------+                                                                                                                   
  |--> | part 0 | setup mount, open log, init clock, make null stdio                                                                
  |    +--------+                                                                                                                   
  |    +--------+                                                                                                                   
  +--> | part 1 | reset signal handlers, parse config/args, prepare manager, ready manager/journal/dbus, endlessly handle units/jobs
       +--------+                                                                                                                   
```

```
src/core/main.c                                                                                                               
+--------+                                                                                                                     
| part 1 | : reset signal handlers, parse config/args, prepare manager, ready manager/journal/dbus, endlessly handle units/jobs
+-|------+                                                                                                                     
  |    +---------------------------+                                                                                           
  |--> | reset_all_signal_handlers | reset to default handler and set 'restart' flag for each signal                           
  |    +---------------------------+                                                                                           
  |    +----------------+                                                                                                      
  |--> | ignore_signals | ignore signal 'pipe'                                                                                 
  |    +----------------+                                                                                                      
  |    +---------------------+                                                                                                 
  |--> | parse_configuration | parse config_files & proc_cmdline & env_vars                                                    
  |    +---------------------+                                                                                                 
  |    +------------+                                                                                                          
  |--> | parse_argv | (skip, I don't see any argument usage in our case)                                                       
  |    +------------+                                                                                                          
  |                                                                                                                            
  |--> if action == run                                                                                                        
  |    |                                                                                                                       
  |    |    +------------------------+                                                                                         
  |    |--> | setup_console_terminal | reset terminal settings of /dev/console                                                 
  |    |    +------------------------+                                                                                         
  |    |    +----------+                                                                                                       
  |    +--> | log_open | given target, ensure its fd is ready                                                                  
  |         +----------+                                                                                                       
  |    +--------------------+                                                                                                  
  |--> | log_execution_mode | log execution info                                                                               
  |    +--------------------+                                                                                                  
  |    +--------------------+                                                                                                  
  |--> | initialize_runtime | install crash handler for signals, setup machine_id, setup loopback, disable watchdog            
  |    +--------------------+                                                                                                  
  |    +-------------+                                                                                                         
  |--> | manager_new | alloc manager, get default event, prepare all kinds of sources and callbacks                            
  |    +-------------+                                                                                                         
  |    +----------------------+                                                                                                
  |--> | set_manager_defaults | set some manager fields to default                                                             
  |    +----------------------+                                                                                                
  |    +----------------------+                                                                                                
  |--> | set_manager_settings | further set up manager                                                                         
  |    +----------------------+                                                                                                
  |    +------------------------+                                                                                              
  |--> | manager_set_first_boot | generate or remove /run/systemd/first-boot                                                   
  |    +------------------------+                                                                                              
  |    +-----------------+                                                                                                     
  |--> | manager_startup | set up and start manager, ensure journal and dbus are ready                                         
  |    +-----------------+                                                                                                     
  |    +------------------+                                                                                                    
  |--> | invoke_main_loop | loop: dispatch unit units/jobs, epoll, handle event                                                
  |    +------------------+                                                                                                    
  |                                                                                                                            
  +--> (normally it should reach here, skip)                                                                                   
```

```
src/core/main.c                                                                
+------------------+                                                            
| invoke_main_loop | : loop: dispatch unit units/jobs, epoll, handle event      
+-|----------------+                                                            
  |                                                                             
  +--> endless loop                                                             
       |                                                                        
       |    +--------------+                                                    
       |--> | manager_loop | loop: dispatch unit units/jobs, epoll, handle event
       |    +--------------+                                                    
       |                                                                        
       +--> switch objective                                                    
            (cases, skip)                                                       
```

```
src/core/manager.c                                                                                                  
+--------------+                                                                                                     
| manager_loop | : loop: dispatch unit units/jobs, epoll, handle event                                               
+-|------------+                                                                                                     
  |    +------------------------+                                                                                    
  |--> | manager_check_finished | (skip)                                                                             
  |    +------------------------+                                                                                    
  |    +-----------------------------+                                                                               
  |--> | sd_event_source_set_enabled |                                                                               
  |    +-----------------------------+                                                                               
  |                                                                                                                  
  +--> while ->objective == manager_ok                                                                               
       |                                                                                                             
       |    +---------------+                                                                                        
       |--> | watchdog_ping | ensure watchdog is opened, ping it                                                     
       |    +---------------+                                                                                        
       |    +-----------------------------+                                                                          
       |--> | manager_dispatch_load_queue | for each unit in load_queue, load it and add to other queues             
       |    +-----------------------------+                                                                          
       |    +-------------------------------+                                                                        
       |--> | manager_dispatch_gc_job_queue | for each job in gc_queue: uninstall and free it                        
       |    +-------------------------------+                                                                        
       |    +--------------------------------+                                                                       
       |--> | manager_dispatch_gc_unit_queue | handle each unit in gc_unit_queue                                     
       |    +--------------------------------+                                                                       
       |    +--------------------------------+                                                                       
       |--> | manager_dispatch_cleanup_queue | for each unit in cleanup_queue, free it                               
       |    +--------------------------------+                                                                       
       |    +------------------------------------------+                                                             
       |--> | manager_dispatch_start_when_upheld_queue | for each unit in upheld_queue, activate it                  
       |    +------------------------------------------+                                                             
       |    +----------------------------------------+                                                               
       |--> | manager_dispatch_stop_when_bound_queue | for each unit in bound_queue, activate it                     
       |    +----------------------------------------+                                                               
       |    +-------------------------------------------+                                                            
       |--> | manager_dispatch_stop_when_unneeded_queue | for each unit in unneeded_queue, activate it               
       |    +-------------------------------------------+                                                            
       |    +------------------------------------------+                                                             
       |--> | manager_dispatch_release_resources_queue | for each unit in resources_queue, call ->release_resources()
       |    +------------------------------------------+                                                             
       |    +-----------------------------+                                                                          
       |--> | manager_dispatch_dbus_queue | handle each unit and job                                                 
       |    +-----------------------------+                                                                          
       |    +--------------+                                                                                         
       +--> | sd_event_run | register event (sources) to epoll, wait till there's pending source, dispatch it        
            +--------------+                                                                                         
```

```
src/shared/watchdog.c                                           
+---------------+                                                
| watchdog_ping | : ensure watchdog is opened, ping it           
+-|-------------+                                                
  |                                                              
  |--> if watchdog_timeout isn't set, return                     
  |                                                              
  |--> if not watchdog_fd yet                                    
  |    |                                                         
  |    |    +---------------+                                    
  |    +--> | open_watchdog | config watchdog, enable and ping it
  |         +---------------+                                    
  |         return                                               
  |                                                              
  |    +-------------------+                                     
  +--> | watchdog_ping_now | ping watchdog thru ioctl            
       +-------------------+                                     
```

```
src/shared/watchdog.c                                       
+---------------+                                            
| open_watchdog | : config watchdog, enable and ping it      
+-|-------------+                                            
  |                                                          
  |--> open /dev/watchdog0, save path in watchdog_device     
  |                                                          
  |    +----------------+                                    
  +--> | update_timeout | config watchdog, enable and ping it
       +----------------+                                    
```

```
src/shared/watchdog.c                                                    
+----------------+                                                        
| update_timeout | : config watchdog, enable and ping it                  
+-|--------------+                                                        
  |                                                                       
  |--> if watchdog_timeout != infinity                                    
  |    |                                                                  
  |    |    +----------------------+                                      
  |    +--> | watchdog_set_timeout | set timeout of watchdog thru ioctl   
  |         +----------------------+                                      
  |                                                                       
  |--> if watchdog_timeout == infinity                                    
  |    |                                                                  
  |    |    +-----------------------+                                     
  |    +--> | watchdog_read_timeout | get timeout from watchdog thru ioctl
  |         +-----------------------+                                     
  |    +-------------------+                                              
  |--> | update_pretimeout | (skip)                                       
  |    +-------------------+                                              
  |    +---------------------+                                            
  +--> | watchdog_set_enable | enable watchdog                            
  |    +---------------------+                                            
  |    +-------------------+                                              
  +--> | watchdog_ping_now | ping watchdog thru ioctl                     
       +-------------------+                                              
```

```
src/core/manager.c                                                                
+-------------------------------+                                                  
| manager_dispatch_gc_job_queue | : for each job in gc_queue: uninstall and free it
+-|-----------------------------+                                                  
  |                                                                                
  +--> for each job in gc_job_queue                                                
       |                                                                           
       |    +-------------+                                                        
       |--> | LIST_REMOVE | remove job from list                                   
       |    +-------------+                                                        
       |    +---------------------------+                                          
       +--> | job_finish_and_invalidate | uninstall job ad free it, notify         
            +---------------------------+                                          
```

```
src/core/manager.c                                                                
+--------------------------------+                                                 
| manager_dispatch_gc_unit_queue | : handle each unit in gc_unit_queue             
+-|------------------------------+                                                 
  |                                                                                
  +--> for each unit in gc_unit_queue                                              
       |                                                                           
       |    +---------------+                                                      
       |--> | unit_gc_sweep | mark unit good or bad                                
       |    +---------------+                                                      
       |    +-------------+                                                        
       |--> | LIST_REMOVE | remove unit from list                                  
       |    +-------------+                                                        
       |                                                                           
       +--> if unit marker isn't good                                              
            |                                                                      
            |    +---------------------------+                                     
            +--> | unit_add_to_cleanup_queue | add unit to cleanup_queue of manager
                 +---------------------------+                                     
```

```
src/core/manager.c                                                                            
+------------------------------------------+                                                   
| manager_dispatch_start_when_upheld_queue | : for each unit in upheld_queue, activate it      
+-|----------------------------------------+                                                   
  |                                                                                            
  +--> for each unit in upheld_queue                                                           
       |                                                                                       
       |    +-------------+                                                                    
       |--> | LIST_REMOVE | remove unit from queue                                             
       |    +-------------+                                                                    
       |    +-----------------+                                                                
       +--> | manager_add_job | alloc transaction, add job to it, activate transaction, free it
            +-----------------+                                                                
```

```
src/core/manager.c                                                                                        
+------------------------------------------+                                                               
| manager_dispatch_release_resources_queue | : for each unit in resources_queue, call ->release_resources()
+-|----------------------------------------+                                                               
  |                                                                                                        
  +--> for each unit in resources_queue                                                                    
       |                                                                                                   
       |    +-------------+                                                                                
       |--> | LIST_REMOVE | remove unit from queue                                                         
       |    +-------------+                                                                                
       |    +------------------------+                                                                     
       +--> | unit_release_resources | call ->release_resources()                                          
            +------------------------+                                                                     
```

```
src/core/unit.c                                         
+------------------------+                               
| unit_release_resources | : call ->release_resources()  
+-|----------------------+                               
  |    +-------------------+                             
  |--> | unit_active_state | call ->active_state()       
  |    +-------------------+                             
  |    +-----------------------+                         
  |--> | unit_get_exec_context | get exec_context in unit
  |    +-----------------------+                         
  |                                                      
  +--> call ->release_resources()                        
```

```
src/core/manager.c                                                                                                 
+-----------------------------+                                                                                     
| manager_dispatch_dbus_queue | : handle each unit and job                                                          
+-|---------------------------+                                                                                     
  |                                                                                                                 
  |--> while we have budget && there's unit needs to be handled                                                     
  |    |                                                                                                            
  |    |    +-----------------------------+                                                                         
  |    +--> | bus_unit_send_change_signal | move unit from dbus queue to gc queue, send signal to private/api busses
  |         +-----------------------------+                                                                         
  |                                                                                                                 
  |--> while we have budget && there's job needs to be handled                                                      
  |    |                                                                                                            
  |    |    +----------------------------+                                                                          
  |    +--> | bus_job_send_change_signal | send change signal of unit first, then send change signal of job         
  |         +----------------------------+                                                                          
  |                                                                                                                 
  |--> if send_reloading_done == true                                                                               
  |    |                                                                                                            
  |    |    +----------------------------+                                                                          
  |    +--> | bus_manager_send_reloading | send signal of 'reloading' to private/api buses                          
  |         +----------------------------+                                                                          
  |                                                                                                                 
  +--> if pending_reload_message == true                                                                            
       |                                                                                                            
       |    +---------------------------------+                                                                     
       +--> | bus_send_pending_reload_message | send out msg of 'pending_reload'                                    
            +---------------------------------+                                                                     
```

```
src/core/dbus-manager.c                                                                     
+----------------------------+                                                               
| bus_manager_send_reloading | : send signal of 'reloading' to private/api buses             
+-|--------------------------+                                                               
  |    +-----------------+                                                                   
  +--> | bus_foreach_bus | send signal to private bus, and api bus (if someone is interested)
       +-----------------+ +----------------+                                                
                           | send_reloading | send signal of 'reloading'                     
                           +----------------+                                                
```

```
core/manager.c                                                                                                   
+-----------------------------+                                                                                   
| manager_enumerate_perpetual | : for each supported unit type, call its ->enumerate_perpetual()                  
+-|---------------------------+                                                                                   
  |                                                                                                               
  +--> for each unit type                                                                                         
       -                                                                                                          
       +--> if ->enumerate_perpetual() exists                                                                     
            -                                                                                                     
            +--> call it, e.g.,                                                                                   
                 +---------------------------+                                                                    
                 | mount_enumerate_perpetual | ensure unit of "-.mount" exists, add to both load/dbus queues      
                 +---------------------------+                                                                    
                 | slice_enumerate_perpetual | ensure two slice units exist and add them to both load/dbus queues 
                 +---------------------------+                                                                    
                 | scope_enumerate_perpetual | ensure unit of "init.scope" exists and add to both load/dbus queues
                 +---------------------------+                                                                    
```

```
core/mount.c                                                                                
+---------------------------+                                                                
| mount_enumerate_perpetual | : ensure unit of "-.mount" exists, add to both load/dbus queues
+-|-------------------------+                                                                
  |    +------------------+                                                                  
  |--> | manager_get_unit | given name (e.g., "-.mount"), get unit from manager              
  |    +------------------+                                                                  
  |                                                                                          
  |--> if it't not there                                                                     
  |    |                                                                                     
  |    |    +-------------------+                                                            
  |    +--> | unit_new_for_name | alloc unit, add to manager                                 
  |         +-------------------+                                                            
  |    +------------------------+                                                            
  |--> | unit_add_to_load_queue | prepend unit to load_queue of manager                      
  |    +------------------------+                                                            
  |    +------------------------+                                                            
  +--> | unit_add_to_dbus_queue | prepend unit to dbus_queue of manager                      
       +------------------------+                                                            
```

```
core/slice.c                                                                                               
+---------------------------+                                                                               
| slice_enumerate_perpetual | : ensure two slice units exist and add them to both load/dbus queues          
+-|-------------------------+                                                                               
  |     +----------------------+                                                                            
  |---> | slice_make_perpetual | ensure unit of "-.slice" exists and add to both load/dbus queues           
  |     +----------------------+                                                                            
  |                                                                                                         
  +---> if manager is in system scope                                                                       
        |                                                                                                   
        |     +----------------------+                                                                      
        +---> | slice_make_perpetual | ensure unit of "system.slice" exists and add to both load/dbus queues
              +----------------------+                                                                      
```

```
core/manager.c                                                                                                                               
+-------------------+                                                                                                                         
| manager_enumerate | : call ->enumerate() of each unit type, dispatch load queue                                                             
+-|-----------------+                                                                                                                         
  |                                                                                                                                           
  |--> for each supported unit                                                                                                                
  |    -                                                                                                                                      
  |    +--> if ->enumerate() exists                                                                                                           
  |         -                                                                                                                                 
  |         +--> call it, e.g.,                                                                                                               
  |              +-----------------+                                                                                                          
  |              | mount_enumerate | ensure manager has mount moinitor ready, prepare unit for each entry in mount table and add to load queue
  |              +-----------------+                                                                                                          
  |    +-----------------------------+                                                                                                        
  +--> | manager_dispatch_load_queue | for each unit in load_queue, load it and add to other queues                                           
       +-----------------------------+                                                                                                        
```

```
core/mount.c                                                                                                                  
+-----------------+                                                                                                            
| mount_enumerate | : ensure manager has mount moinitor ready, prepare unit for each entry in mount table and add to load queue
+-|---------------+                                                                                                            
  |                                                                                                                            
  |--> if manager has no mount_monitor yet                                                                                     
  |    |                                                                                                                       
  |    |--> alloc monitor                                                                                                      
  |    |                                                                                                                       
  |    |    +---------------------------+                                                                                      
  |    |--> | mnt_monitor_enable_kernel | ?                                                                                    
  |    |    +---------------------------+                                                                                      
  |    |    +------------------------------+                                                                                   
  |    |--> | mnt_monitor_enable_userspace | ?                                                                                 
  |    |    +------------------------------+                                                                                   
  |    |    +--------------------+                                                                                             
  |    |--> | mnt_monitor_get_fd |                                                                                             
  |    |    +--------------------+                                                                                             
  |    |    +-----------------+                                                                                                
  |    +--> | sd_event_add_io | prepare 'source' and add to arg 'event, register the source's io                               
  |         +-----------------+ +-------------------+                                                                          
  |                             | mount_dispatch_io |                                                                          
  |                             +-------------------+                                                                          
  |    +--------------------------------+                                                                                      
  +--> | mount_load_proc_self_mountinfo | for each entry in mount table, ensure it has mount unit and in load_queue            
       +--------------------------------+                                                                                      
```

```
core/mount.c                                                                                                        
+--------------------------------+                                                                                   
| mount_load_proc_self_mountinfo | : for each entry in mount table, ensure it has mount unit and in load_queue       
+-|------------------------------+                                                                                   
  |    +----------------+                                                                                            
  |--> | libmount_parse | determine source, parse, get table/iter                                                    
  |    +----------------+                                                                                            
  |                                                                                                                  
  +--> endless loop                                                                                                  
       |                                                                                                             
       |    +-------------------+                                                                                    
       |--> | mnt_table_next_fs | get next fs from table                                                             
       |    +-------------------+                                                                                    
       |                                                                                                             
       |--> if got nothing, break                                                                                    
       |                                                                                                             
       +--> get source/target/options/fs_type from fs                                                                
       |                                                                                                             
       |    +-------------------+                                                                                    
       |--> | device_found_node | ensur device unit is in load_queue, update its state                               
       |    +-------------------+                                                                                    
       |    +------------------+                                                                                     
       +--> | mount_setup_unit | ensure mount unit is in load_queue of manager                                       
            +------------------+                                                                                     
                                                                                                                     
                                                                                                                     
            dev, /dev, rw,relatime,size=403092k,nr_inodes=100773,mode=755, devtmpfs   <-- ignored in mount_setup_unit
            sys, /sys, rw,relatime, sysfs                                             <-- ignored in mount_setup_unit
            proc, /proc, rw,relatime, proc                                            <-- ignored in mount_setup_unit
            tmpfs, /run, rw,nodev,relatime,mode=755, tmpfs                            <-- ignored in mount_setup_unit
            /dev/mtdblock4, /run/initramfs/ro, ro,relatime,errors=continue, squashfs  <-- handled                    
```

```
core/device.c                                                              
+-------------------+                                                       
| device_found_node | : ensur device unit is in load_queue, update its state
+-|-----------------+                                                       
  |                                                                         
  |--> if found & mask                                                      
  |    |                                                                    
  |    |    +----------------------------+                                  
  |    |--> | sd_device_new_from_devname |                                  
  |    |    +----------------------------+                                  
  |    |    +-------------------+                                           
  |    +--> | device_setup_unit | ensure unit (device) is in load_queue     
  |         +-------------------+                                           
  |    +-----------------------------+                                      
  +--> | device_update_found_by_name | update device unit's state           
       +-----------------------------+                                      
```

```
core/device.c                                                                                    
+-------------------+                                                                             
| device_setup_unit | : ensure unit (device) is in load_queue                                     
+-|-----------------+                                                                             
  |                                                                                               
  |--> if dev is provided                                                                         
  |    -                                                                                          
  |    +--> get sysfs path from it                                                                
  |                                                                                               
  |    +---------------------+                                                                    
  |--> | unit_name_from_path | given path, get unit name '*.device'                               
  |    +---------------------+                                                                    
  |    +------------------+                                                                       
  |--> | manager_get_unit | given unit, get job from manager                                      
  |    +------------------+                                                                       
  |                                                                                               
  |--> if unit found                                                                              
  |    |                                                                                          
  |    |    +--------------------------+                                                          
  |    +--> | unit_remove_dependencies | given unit, remove dependencies from both sides          
  |         +--------------------------+                                                          
  |                                                                                               
  |--> else                                                                                       
  |    -                                                                                          
  |    +--> alloc unit and add to load_queue of manager                                           
  |                                                                                               
  |--> save path in unit (Device)                                                                 
  |                                                                                               
  |--> if sysfs                                                                                   
  |    |                                                                                          
  |    |    +------------------+                                                                  
  |    +--> | device_set_sysfs | add (sysfs, device) to hashmap in manager, add unit to dbus queue
  |         +------------------+                                                                  
  |    +---------------------------+                                                              
  |--> | device_update_description |                                                              
  |    +---------------------------+                                                              
  |                                                                                               
  +--> ensure unit (device) is in arg set                                                         
```

```
core/unit.c                                                                                                
+--------------------------+                                                                                
| unit_remove_dependencies | : given unit, remove dependencies from both sides                              
+-|------------------------+                                                                                
  |                                                                                                         
  +--> for each hashmap of dependency_type in target_unit                                                   
       -                                                                                                    
       +--> for each (other_unit, dep_info) in hashmap                                                      
            |                                                                                               
            |    +-----------------------------+                                                            
            |--> | unit_update_dependency_mask | remove (other_unit, value) from hashmap of target_unit     
            |    +-----------------------------+                                                            
            |                                                                                               
            |--> if other_unit also depends on target_unit                                                  
            |    |                                                                                          
            |    |    +-----------------------------+                                                       
            |    +--> | unit_update_dependency_mask | remove (target_unit, value) from hashmap of other_unit
            |         +-----------------------------+                                                       
            |    +----------------------+                                                                   
            |--> | unit_add_to_gc_queue | add other_unit to gc_queue                                        
            |    +----------------------+                                                                   
            |    +-----------------------------------------+                                                
            +--> | unit_submit_to_stop_when_unneeded_queue | add other_unit to stop_when_unneeded_queue     
                 +-----------------------------------------+                                                
```

```
core/mount.c
+------------------+
| mount_setup_unit | : ensure mount unit is in load_queue of manager
+-|----------------+
  |    +---------------------+
  |--> | unit_name_from_path | get unit name from path, e.g., /run/initramfs/ro --> run-initramfs-ro.mount
  |    +---------------------+
  |    +------------------+
  |--> | manager_get_unit | given name, get unit from manager
  |    +------------------+
  |
  |--> if found
  |    |
  |    |    +---------------------------+
  |    +--> | mount_setup_existing_unit | save mount info in mount unit
  |         +---------------------------+
  |
  +--> else
       |
       |    +----------------------+
       +--> | mount_setup_new_unit | alloc unit, add dependencies, add to load_queue
            +----------------------+
```

```
src/shared/bus-util.c                                                                                           
+---------------------------------------------+                                                                  
| bus_open_system_watch_bind_with_description | : get a bus, configure it and start                              
+-|-------------------------------------------+                                                                  
  |    +------------+                                                                                            
  |--> | sd_bus_new | get a bus                                                                                  
  |    +------------+                                                                                            
  |    +--------------------+                                                                                    
  |--> | sd_bus_set_address | bus->address = address                                                             
  |    +--------------------+                                                                                    
  |    +-----------------------+                                                                                 
  |--> | sd_bus_set_bus_client | bus->bus_client = arg                                                           
  |    +-----------------------+                                                                                 
  |    +------------------------+                                                                                
  |--> | sd_bus_negotiate_creds | (skip)                                                                         
  |    +------------------------+                                                                                
  |    +-----------------------+                                                                                 
  |--> | sd_bus_set_watch_bind | bus->watch_bind = arg                                                           
  |    +-----------------------+                                                                                 
  |    +-----------------------------+                                                                           
  |--> | sd_bus_set_connected_signal | bus->connected_signal = arg                                               
  |    +-----------------------------+                                                                           
  |    +--------------+                                                                                          
  +--> | sd_bus_start | set bus state = opening, prepare socket/epoll, send msg (hello) to "org.freedesktop.DBus"
       +--------------+                                                                                          
```

```
src/libsystemd/sd-bus/sd-bus.c                                                                 
+-------------------+                                                                           
| bus_start_running | : set bus state = running, prepare signal ('connected'), prepend to rqueue
+-|-----------------+                                                                           
  |                                                                                             
  |--> set bus state = running                                                                  
  |                                                                                             
  |    +-----------------------------+                                                          
  +--> | synthesize_connected_signal | prepare signal ('connected'), prepend to rqueue          
       +-----------------------------+                                                          
```

```
src/libsystemd/sd-bus/sd-bus.c                                                  
+-----------------------------+                                                  
| synthesize_connected_signal | : prepare signal ('connected'), prepend to rqueue
+-|---------------------------+                                                  
  |    +---------------------------+                                             
  |--> | sd_bus_message_new_signal | prepare msg of signal type                  
  |    +---------------------------+ obj: "/org/freedesktop/DBus/Local"          
  |                                  iface: "org.freedesktop.DBus.Local"         
  |                                  prop: "Connected"                           
  |                                                                              
  |    +------------------------------+                                          
  |--> | bus_message_set_sender_local | set sender = "org.freedesktop.DBus.Local"
  |    +------------------------------+                                          
  |    +----------------------------+                                            
  |--> | bus_seal_synthetic_message |                                            
  |    +----------------------------+                                            
  |    +----------------------+                                                  
  |--> | bus_rqueue_make_room | realloc rqueue for current msg                   
  |    +----------------------+                                                  
  |    +---------+                                                               
  |--> | memmove | shift every [n] to [n+1]                                      
  |    +---------+                                                               
  |                                                                              
  |--> rqueue[0] = current msg                                                   
  |                                                                              
  +--> update rqueue_size                                                        
```

```
src/libsystemd/sd-device/device-monitor.c                              
+-----------------------+                                               
| sd_device_monitor_new | : alloc and setup sd_device_monitor           
+-------------------------+                                             
| device_monitor_new_full | : alloc and setup sd_device_monitor         
+-|-----------------------+                                             
  |                                                                     
  |--> if arg fd is invalid                                             
  |    |                                                                
  |    |    +--------+                                                  
  |    +--> | socket |                                                  
  |         +--------+                                                  
  |                                                                     
  |--> alloc and setup sd_device_monitor                                
  |                                                                     
  +--> if arg fd is valid                                               
       |                                                                
       |    +------------------------+                                  
       +--> | monitor_set_nl_address | get sock name and save in monitor
            +------------------------+                                  
```

```
src/libsystemd/sd-device/device-monitor.c                                                                           
+-------------------------+                                                                                          
| sd_device_monitor_start | : attach event to monitor, register callback (setup sd_device) to 'io' source            
+-|-----------------------+                                                                                          
  |                                                                                                                  
  |--> if monitor has no event yet                                                                                   
  |    |                                                                                                             
  |    |    +--------------------------------+                                                                       
  |    +--> | sd_device_monitor_attach_event | ensure monitor has event                                              
  |         +--------------------------------+                                                                       
  |    +---------------------------------+                                                                           
  |--> | device_monitor_enable_receiving | enable receiving in monitor                                               
  |    +---------------------------------+                                                                           
  |                                                                                                                  
  |--> save callback/userdata in monitor                                                                             
  |                                                                                                                  
  |    +-----------------+                                                                                           
  |--> | sd_event_add_io | prepare 'source' and add to arg 'event, register the source's io                          
  |    +-----------------+ +------------------------------+                                                          
  |                        | device_monitor_event_handler | receive data to setup sd_device, call monitor->callback()
  |                        +------------------------------+                                                          
  |    +---------------------------------+                                                                           
  +--> | sd_event_source_set_description | "sd-device-monitor"                                                       
       +---------------------------------+                                                                           
```

```
src/libsystemd/sd-device/device-monitor.c                       
+---------------------------------+                              
| device_monitor_enable_receiving | : enable receiving in monitor
+-|-------------------------------+                              
  |    +---------------------------------+                       
  |--> | sd_device_monitor_filter_update | update packet filter  
  |    +---------------------------------+                       
  |                                                              
  +--> if monitor isn't bound yet                                
       |                                                         
       |    +------+                                             
       |--> | bind |                                             
       |    +------+                                             
       |                                                         
       |--> ->bound = true                                       
       |                                                         
       |    +------------------------+                           
       +--> | monitor_set_nl_address | save pid in monitor       
            +------------------------+                           
```

```
src/libsystemd/sd-device/device-monitor.c                                                  
+------------------------------+                                                            
| device_monitor_event_handler | : receive data to setup sd_device, call monitor->callback()
+-|----------------------------+                                                            
  |    +-------------------------------+                                                    
  |--> | device_monitor_receive_device | receive data to setup sd_device                    
  |    +-------------------------------+                                                    
  |                                                                                         
  +--> if ->callback() exists                                                               
       -                                                                                    
       +--> call it, e.g.,                                                                  
            +------------------------+                                                      
            | manager_process_uevent | process uevent                                       
            +------------------------+                                                      
```

```
src/libsystemd/sd-device/device-monitor.c                         
+-------------------------------+                                  
| device_monitor_receive_device | : receive data to setup sd_device
+-|-----------------------------+                                  
  |    +-----------------------+                                   
  |--> | next_datagram_size_fd | peek data size                    
  |    +-----------------------+                                   
  |                                                                
  |--> prepare msg buf of that size                                
  |                                                                
  |    +---------+                                                 
  |--> | recvmsg | receive data                                    
  |    +---------+                                                 
  |                                                                
  |--> determine msg source, and get offset properly               
  |                                                                
  |    +------------------------+                                  
  |--> | device_new_from_nulstr | alloc and setup sd_device        
  |    +------------------------+                                  
  |                                                                
  |--> if is initialized (source == udev)                          
  |    |                                                           
  |    |    +---------------------------+                          
  |    +--> | device_set_is_initialized |                          
  |         +---------------------------+                          
  |    +---------------+                                           
  +--> | passes_filter | check if device should be dropped         
       +---------------+                                           
```

```
src/libsystemd/sd-device/device-private.c                                     
+------------------------+                                                     
| device_new_from_nulstr | : alloc and setup sd_device                         
+-|----------------------+                                                     
  |    +----------------+                                                      
  |--> | device_new_aux | alloc and init sd_device                             
  |    +----------------+                                                      
  |                                                                            
  |--> for each (key, value) pair                                              
  |    |                                                                       
  |    |    +---------------+                                                  
  |    +--> | device_append | parse (key, value) pairs and get major/minor info
  |         +---------------+                                                  
  |                                                                            
  |--> if we got the major                                                     
  |    |                                                                       
  |    |    +-------------------+                                              
  |    +--> | device_set_devnum | add property to device                       
  |         +-------------------+                                              
  |    +---------------+                                                       
  +--> | device_verify |                                                       
       +---------------+                                                       
```

```
src/libsystemd/sd-device/device-private.c                                          
+---------------+                                                                   
| device_append | : parse (key, value) pairs and get major/minor info               
+-|-------------+                                                                   
  |                                                                                 
  |--> get value                                                                    
  |                                                                                 
  |--> if key is 'major'                                                            
  |    -                                                                            
  |    +--> major = value                                                           
  |                                                                                 
  |--> elif key is 'minor'                                                          
  |    -                                                                            
  |    +--> minor = value                                                           
  |                                                                                 
  +--> else                                                                         
       |                                                                            
       |    +--------------+                                                        
       +--> | device_amend | given different key, add property to device accordingly
            +--------------+                                                        
```

```
src/shared/conf-parser.c                                                                
+--------------------------+                                                             
| config_parse_many_nulstr | : collect related configs, parse them, add info to hashmap  
+-|------------------------+                                                             
  |    +------------------------+                                                        
  |--> | conf_files_list_nulstr | collect drop-in files in "systemd/networkd.conf.d"     
  |    +------------------------+                                                        
  |    +-------------------------+                                                       
  +--> | config_parse_many_files | alloc hashmap, parse config(s) and add info to hashmap
       +-------------------------+                                                       
```

```
src/shared/conf-parser.c                                                           
+-------------------------+                                                         
| config_parse_many_files | : alloc hashmap, parse config(s) and add info to hashmap
+-|-----------------------+                                                         
  |                                                                                 
  |    alloc hashmap                                                                
  |                                                                                 
  |--> for each config file found (in our case, only /etc/systemd/networkd.conf)    
  |    |                                                                            
  |    |    +--------------+                                                        
  |    |--> | config_parse | parse config and get stats                             
  |    |    +--------------+                                                        
  |    |    +---------------------------+                                           
  |    +--> | hashmap_put_stats_by_path | add stats to hashmap                      
  |         +---------------------------+                                           
  |                                                                                 
  +--> for each drop-in file (not our case)                                         
       |                                                                            
       |    +--------------+                                                        
       |--> | config_parse | parse config and get stats                             
       |    +--------------+                                                        
       |    +---------------------------+                                           
       +--> | hashmap_put_stats_by_path | add stats to hashmap                      
            +---------------------------+                                           
```

```
 src/libsystemd/sd-journal/sd-journal.c                       
 [sd_journal_open] : alloc journal, add search paths          
 [sd_journal_open_namespace] : alloc journal, add search paths
 |                                                            
 |--> [journal_new] alloc and setup 'journal'                 
 |                                                            
 +--> [add_search_paths] add search paths to journal          
```

```
 src/libsystemd/sd-journal/sd-journal.c                            
 [journal_new] : alloc and setup 'journal'                         
 |                                                                 
 |--> alloc 'journal'                                              
 |                                                                 
 |--> setup 'journal'                                              
 |                                                                 
 |--> for file handling, alloc hashmap and cache, save in 'journal'
 |                                                                 
 |--> for folders, alloc hashmap, save in 'journal'                
 |                                                                 
 +--> alloc mmap cache, save in 'journal'                          
```

```
 src/libsystemd/sd-journal/sd-journal.c                                         
 [add_search_paths] : add search paths to journal                               
 -                                                                              
 +--> for each predefined path                                                  
      -                                                                         
      +--> [add_root_directory] open folder, ensure it's in hashmap of 'journal'
```

```
 src/libsystemd/sd-journal/sd-journal.c                                 
 [add_root_directory] : open folder, ensure it's in hashmap of 'journal'
 |                                                                      
 |--> [directory_open]                                                  
 |                                                                      
 +--> add the directory to hashmap in 'journal'                         
```
