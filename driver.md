> Study case: Linux version 5.15.0 on OpenBMC

## Index

- [Introduction](#introduction)
- [Device Tree](#device-tree)
- [To-Do List](#to-do-list)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

A system consists of numerous hardware components, from IP within the chip, such as timer, UART, to external devices on board, e.g., USB gadget or the PCIe card. 
By default, kernel compiles in lots of drivers, and all we have to do is provide the list of devices that we need the kernel to help register. 
Whenever kernel adds a device or a driver, they probe each other to see if there's a match.

## <a name="device-tree"></a> Device Tree

Device tree source (DTS) is the text file describing the list of devices of the SoC and gets compiled into Device tree blob (DTB). 
The kernel will know where to find them in memory for parsing and sequentially registering devices on the list. 
There are also DTS files for inclusion only (DTSI), which helps avoid content duplication.

```
arch/arm/boot/dts/aspeed-g5.dtsi            : base; list of supported components in processor 'apeed'
arch/arm/boot/dts/openbmc-flash-layout.dtsi : partition layout in SPI flash
arch/arm/boot/dts/aspeed-bmc-opp-romulus.dts: overwrite; specially tailored for machine 'romulus'
```

DTS files arrange devices in a hierarchy structure, and each parent can have a few properties and child nodes. 
Property **#address-cells** and **size-cells** in parent node specify how to interpret property **reg** of children. 
The most common node properties are the register base and hardware interrupt number, and the kernel will regard them as device resources when parsing.

```
    ahb {
        compatible = "simple-bus";
        #address-cells = <1>;   // 1 dword (4 bytes)
        #size-cells = <1>;      // 1 dword (4 bytes)
        ranges;                 // so we interpret child node's reg as < 1 dword addr + 1 dword size, and so on >

        fmc: spi@1e620000 {
            reg = < 0x1e620000 0xc4             // a. 1 dword addr (0x1e620000) + 1 dword size (0xc4)
                0x20000000 0x10000000 >;        // b. 1 dword addr (0x20000000) + 1 dword size (0x10000000)
            #address-cells = <1>;              
            #size-cells = <0>; 
            compatible = "aspeed,ast2500-fmc";
            clocks = <&syscon ASPEED_CLK_AHB>;
            status = "disabled";
            interrupts = <19>;                  // c. hardware irq
```

The kernel will add resources for the above item a, b, and c. So the driver can later access them by the usage:

```
platform_get_resource(pdev, IORESOURCE_MEM, 0); // access the MEM type resource with index = 0
platform_get_resource(pdev, IORESOURCE_MEM, 1); // access the MEM type resource with index = 1
platform_get_resource(pdev, IORESOURCE_IRQ, 0); // access the IRQ type resource with index = 0
```

If many DTS and DTSI files involve in the DTB complication, that might makes it difficult to know whether a device is enabled or disabled eventually.
Utility **dtc** can construct the DTS from DTB and give us a quick inspection without the need to boot up target system.

```
dtc -I dtb -O dts bcm2711-rpi-4-b.dtb           # construct dts from dtb
dtc -I fs -O dts /sys/firmware/devicetree/base  # construct dts from filesystem (not that useful to me)
```

## <a name="to-do-list"></a> To-Do List

(None)

## <a name="reference"></a> Reference

(None)

<details>
  <summary> Messy Notes </summary>
    
To debug init calls, we add _initcall_debug_ to _bootargs_ in DTS.

```
    chosen {
        stdout-path = &uart5;
        bootargs = "console=ttyS4,115200 earlycon initcall_debug";
    };

```

Please note that by default the debug log won't display during boot time, and we need to utilize _dmesg_ to check it instead. 

- Related log
```
[    1.489785] aspeed-smc 1e620000.spi: Using 50 MHz SPI frequency
[    1.493323] aspeed-smc 1e620000.spi: n25q256a (32768 Kbytes)
[    1.493947] aspeed-smc 1e620000.spi: CE0 window [ 0x20000000 - 0x22000000 ] 32MB
[    1.494494] aspeed-smc 1e620000.spi: CE1 window [ 0x22000000 - 0x2a000000 ] 128MB
[    1.494842] aspeed-smc 1e620000.spi: read control register: 203b0045
[    1.728658] random: crng init done
[    1.923572] 5 fixed-partitions partitions found on MTD device bmc
[    1.923940] Creating 5 MTD partitions on "bmc":
[    1.924305] 0x000000000000-0x000000060000 : "u-boot"
[    1.927399] 0x000000060000-0x000000080000 : "u-boot-env"
[    1.930167] 0x000000080000-0x0000004c0000 : "kernel"
[    1.932337] 0x0000004c0000-0x000001c00000 : "rofs"
[    1.934627] 0x000001c00000-0x000002000000 : "rwfs"

▲ fmc: spi@1e620000/flash@0
------------------------------------------------------------------------------------------
▼ spi1: spi@1e630000/flash@0

[    1.941091] aspeed-smc 1e630000.spi: Using 100 MHz SPI frequency
[    1.942534] aspeed-smc 1e630000.spi: mx66l1g45g (131072 Kbytes)
[    1.942806] aspeed-smc 1e630000.spi: CE0 window resized to 120MB (AST2500 HW quirk)
[    1.943841] aspeed-smc 1e630000.spi: CE0 window [ 0x30000000 - 0x37800000 ] 120MB
[    1.944343] aspeed-smc 1e630000.spi: CE1 window [ 0x37800000 - 0x38000000 ] 8MB
[    1.944715] aspeed-smc 1e630000.spi: CE0 window too small for chip 128MB
[    1.945007] aspeed-smc 1e630000.spi: read control register: 203c0045
[    1.947181] aspeed-smc 1e630000.spi: Calibration area too uniform, using low speed

Code flow:
[aspeed_smc_probe]
    prepare controller
    get 1st MEM resource and map it as registers
    get 2nd MEM resource and map it as AHB base <- ?
    [aspeed_smc_setup_flash]
        for each flash connected to the controller (e.g. only 1)
            prepare chip
        
```
    
</details>
