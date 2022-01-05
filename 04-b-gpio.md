> Study case: Linux version 5.15.0 on OpenBMC

## Index

- [Introduction](#introduction)
- [Boot Flow](#boot-flow)
- [To-Do List](#to-do-list)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

Generic purpose input-output (GPIO) can work as output to control target device or as input to receive signals from outside. 
In modern design, these pins are usually multi-functional. One pin can be an SDA in SMBus protocol or a TX in UART with the proper configuration. 
Pin control (pinctrl) is the mechanism implemented in the kernel to properly configure those pins when we expect them to perform a specific function.

## <a name="boot-flow"></a> Boot Flow

The pinctrl driver and the device in DTS/DTB fit each other because their property 'compatible' hold the same value (**aspeed,ast2500-pinctrl**). 
Then the match triggers probe function **aspeed_g5_pinctrl_probe** for further structure allocation and set up.

- Driver

```
static const struct of_device_id aspeed_g5_pinctrl_of_match[] = {
    { .compatible = "aspeed,ast2500-pinctrl", },
    /*
     * The aspeed,g5-pinctrl compatible has been removed the from the
     * bindings, but keep the match in case of old devicetrees.
     */
    { .compatible = "aspeed,g5-pinctrl", },
    { },
};

static struct platform_driver aspeed_g5_pinctrl_driver = {
    .probe = aspeed_g5_pinctrl_probe,
    .driver = {
        .name = "aspeed-g5-pinctrl",
        .of_match_table = aspeed_g5_pinctrl_of_match,
    },
};
```

- Device

```
                pinctrl: pinctrl@80 {
                    compatible = "aspeed,ast2500-pinctrl";
                    reg = <0x80 0x18>, <0xa0 0x10>;
                    aspeed,external-nodes = <&gfx>, <&lhc>;
                };
```

## <a name="to-do-list"></a> To-Do List

(None)

## <a name="reference"></a> Reference

(None)


