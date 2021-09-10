> Note: browser extension [GitHub + Mermaid](https://chrome.google.com/webstore/detail/github-%20-mermaid/goiiopgdnkogdbjmncgedmgpoajilohe?hl=en)
> is needed to view the flow chart of mermaid syntax.  
> Note: any suggestion or opinion is extremely welcome!

## Index

1. [DTS](#dts)

## <a name="dts"></a> DTS

- DTS: **d**evice **t**ree **s**ource
- DTSI: **d**evice **t**ree **s**ource to be **i**ncluded
- DTB: **d**evice **t**ree **b**lob

DTS is the text file describing device information of the SoC and is compiled into DTB for the kernel to parse and register a bunch of devices. 
Like C language headers, information in DTS can be separated into DTSI as a module, and included by other DTS.
I'm using OpenBMC kernel for my study reference and the corresponding DTS is 

```
arch/arm/boot/dts/aspeed-g5.dtsi <---------- base
arch/arm/boot/dts/aspeed-bmc-opp-romulus.dts <---------- change, e.g. change device status from 'disabled' to 'okay'
arch/arm/boot/dts/openbmc-flash-layout.dtsi <---------- partition layout in SPI flash
```

DTS format
```
    ahb {
        compatible = "simple-bus";
        #address-cells = <1>; <---------- size of 'address': 1 dword (4 bytes)
        #size-cells = <1>; <---------- size of 'size': 1 dword (4 bytes), with these two fields we then know how to parse below property 'reg'
        ranges;

        fmc: spi@1e620000 {
            reg = < 0x1e620000 0xc4 <---------- 1. address = 0x1e620000, size = 0xc4
                0x20000000 0x10000000 >; <---------- 2. address = 0x20000000, size = 0x10000000
            #address-cells = <1>; 
            #size-cells = <0>; 
            compatible = "aspeed,ast2500-fmc";
            clocks = <&syscon ASPEED_CLK_AHB>;
            status = "disabled";
            interrupts = <19>; <---------- 3. interrupt number
          
Above 1/2/3 will be registered as device resources for later use in probe function by:
platform_get_resource(pdev, IORESOURCE_MEM, 0); // access the 1st MEM type resource
platform_get_resource(pdev, IORESOURCE_MEM, 1); // access the 2nd MEM type resource
platform_get_resource(pdev, IORESOURCE_IRQ, 0); // access the 1st IRQ type resouroce
```


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
