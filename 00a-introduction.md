## Index

1. [Introduction](#introduction)
2. [Environment Setup](#setup)
3. [Reference](#reference)

## <a name="introduction"></a> Introduction

Hi there, I plan to start a project that contains all my summarized study notes of the Linux kernel.
Instead of introducing how to operate like a geek through a command-line interface, it will discuss how each module works.
Also, I found the source code in notes is scary at first glance, and therefore diagram and code flow will replace them.
I sincerely hope these tutorials can bridge the gap between academic books and modern kernel implementation.
Any suggestion is highly welcome!

(Add a link of each completed tutorial here later)

## <a name="setup"></a> Environment Setup

For an apparent reason, it's inappropriate to use my working environment for the case study because no one can have the same setting except my colleagues.
Raspberry Pi used to be an ideal option for its affordable price and not many model variants.
However, frequently replacing the SD card merely to confirm code flow is exhausting...
And then QEMU comes into my life! I strongly recommend using it for its free access and GDB support, which ideally suits my code tracing needs.

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
- You can download my pre-built ones.

```
cd linux
wget https://github.com/yushiangfu/study-notes/raw/main/images/flash-romulus
wget https://github.com/yushiangfu/study-notes/raw/main/images/initramfs-romulus
```

- Or prepare your own by building OpenBMC first.

```
git clone https://github.com/openbmc/openbmc.git
cd openbmc
TEMPLATECONF=meta-ibm/meta-romulus/conf . openbmc-env
bitbake obmc-phosphor-image
```

- After a decade, the images will be ready at the below paths. Copy them into the Linux folder.

```
openbmc/build/tmp/deploy/images/romulus/flash-romulus
openbmc/build/tmp/deploy/images/romulus/obmc-phosphor-initramfs-romulus.cpio.xz
```

- Leave commit ID here for a record.

```
commit 5cc2f81c5b66da00cad24e18b0d23442af060c3f (HEAD -> master, origin/master, origin/HEAD)
Author: Andrew Geissler <openbmcbump-github@yahoo.com>
Date:   Fri Nov 19 19:10:18 2021 +0000

    intel-ipmi-oem: srcrev bump c573390054..9e58cfe1ba
    
    Sujoy Ray (1):
          SEL log Generator ID should Conform to IPMI Spec
    
    Change-Id: Icd80a93551de8c963c0528649058437719d1186f
    Signed-off-by: Andrew Geissler <openbmcbump-github@yahoo.com>
```

### Download QEMU

```
cd linux
wget https://jenkins.openbmc.org/job/latest-qemu-x86/lastSuccessfulBuild/artifact/qemu/build/qemu-system-arm
chmod +x qemu-system-arm
```

### Run QEMU

```
cd linux
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

# hostfwd=::2222-:22    map localhost:2222 to QEMU:22
# -s    wait for GDB connection on localhost:1234
# -S    emulation stops at the beginning, and it usually works with GDB
```

### Run GDB

```
cd linux
gdb-multiarch -s vmlinux
(gdb) target remote :1234

# -s    specify symbol file
```

### Transfer a File

Sometimes it's helpful to copy a file into or from QEMU.

```
scp -P 2222 test-file root@localhost:/home/root

# -P    specify which port to connect to on the remote system
```

Now it's showtime!

## <a name="reference"></a> Reference
- [OpenBMC cheatsheet](https://github.com/openbmc/docs/blob/master/cheatsheet.md)
- [J. Stanley, QEMU and OpenBMC](https://drive.google.com/file/d/1oWSaiSA1gtDELCNkgGMLBYR_iUV37n3t/view)
