## Index

1. [Environment Setup](#setup)
2. [Reference](#reference)

## <a name="setup"></a> Environment Setup
### Download Linux source code and build

```
wget https://github.com/openbmc/linux/archive/refs/heads/dev-5.15.zip
unzip dev-5.15.zip
mv linux-dev-5.15 linux
cd linux

export ARCH=arm CROSS_COMPILE=arm-linux-gnueabi-
make aspeed_g5_defconfig
make -j`nproc`
```

### Prepare flash image and initramfs
- You can download my pre-built [flash image](images/flash-romulus) and [initramfs](images/initramfs-romulus).
- Or prepare your own by building OpenBMC first.

```
git clone https://github.com/openbmc/openbmc.git
cd openbmc
TEMPLATECONF=meta-ibm/meta-romulus/conf . openbmc-env
bitbake obmc-phosphor-image
```

- After a decade, the images will be ready at the below paths.

```
openbmc/build/tmp/deploy/images/romulus/flash-romulus
openbmc/build/tmp/deploy/images/romulus/obmc-phosphor-initramfs-romulus.cpio.xz
```

### Download QEMU

```
wget https://jenkins.openbmc.org/job/latest-qemu-x86/lastSuccessfulBuild/artifact/qemu/build/qemu-system-arm
chmod +x qemu-system-arm
```

### Run QEMU
- Replace the below arguments with your actual paths.
  - -kernel 
  - -dtb
  - -initrd
  - -drive

```
./qemu-system-arm \
    -M romulus-bmc \
    -nic user,hostfwd=::2222-:22 \
    -nographic \
    -kernel arch/arm/boot/zImage \
    -dtb arch/arm/boot/dts/aspeed-bmc-opp-romulus.dtb \
    -initrd initramfs-romulus \
    -drive file=flash-romulus,if=mtd,format=raw \
    -s \
    -S

# hostfwd=::2222-:22
#     map localhost:2222 to QEMU:22
# -s
#     wait for GDB connection on localhost:1234
# -S
#     stop at the beginning
#     you might remove it if using GDB isn't an option.


```

## <a name="reference"></a> Reference
- [OpenBMC cheatsheet](https://github.com/openbmc/docs/blob/master/cheatsheet.md)
- [J. Stanley, QEMU and OpenBMC](https://drive.google.com/file/d/1oWSaiSA1gtDELCNkgGMLBYR_iUV37n3t/view)
