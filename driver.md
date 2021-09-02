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
arch/arm/boot/dts/aspeed-bmc-opp-romulus.dts
```

To debug init calls, we add _initcall_debug_ to _bootargs_ in DTS.

```
    chosen {
        stdout-path = &uart5;
        bootargs = "console=ttyS4,115200 earlycon initcall_debug";
    };

```

Please note that by default the debug log won't display during boot time, and we need to utilize _dmesg_ to check it instead.
