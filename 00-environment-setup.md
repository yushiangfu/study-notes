## Index

1. [Environment Setup](#setup)
2. [Reference](#reference)

## <a name="setup"></a> Environment Setup
- Download Linux source code & build
```
wget https://github.com/openbmc/linux/archive/refs/heads/dev-5.15.zip
unzip dev-5.15.zip
mv linux-dev-5.15 linux
cd linux

export ARCH=arm CROSS_COMPILE=arm-linux-gnueabi-
make aspeed_g5_defconfig
make -j`nproc`
```

- Download initrd
```
My link failed...
```
## <a name="reference"></a> Reference
