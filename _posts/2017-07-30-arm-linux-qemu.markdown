---
layout: post
title:  "Running Linux on ARM QEMU Emulator"
date:   2017-07-30 21:15:10 +0800
categories: arm linux qemu busybox
---
要在QEMU上打造一個精簡的ARM Linux環境, 需要的東西有QEMU, toolchain, busybox, Linux kernel source package.

### QEMU
[QEMU][]是一套virtual machine emulator, 可以讓你在PC上模擬其它平台(例如: ARM)的環境. 從官網下載source package後解壓縮後

```bash
$ cd qemu-2.8.1.1
# 新建一個build資料夾, 編譯時產生的檔案就不會污染source tree
$ mkdir build
$ cd build
# 預設是所有支援的平台, 可以用--target-list限定只有ARM平台
$ ../configure --target-list=arm-softmmu --prefix=$HOME/qemu-2.8.1.1
$ make
$ make install
```

完成後, 可以看到qemu-2.8.1.1安裝在家目錄, 裡頭的bin/qemu-system-arm就是要之後要使用的Emulator.

[qemu]: http://www.qemu.org/


### Toolchain
Linaro提供了各種版本的toolchain binary, 可以在[這裡][1]找到. 本文使用的版本是[gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf][2]. 下載完後直接解壓縮, 切換到bin目錄, export路徑到環境變數.

```bash
$ cd gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf
$ export PATH=$PWD:$PATH
```

[1]: http://releases.linaro.org/components/toolchain/binaries/
[2]: http://releases.linaro.org/components/toolchain/binaries/4.9-2017.01/arm-linux-gnueabihf/

### Busybox
[Busybox][]是embedded linux很常用到的一些系統工具的輕量化集合, 我們將以它做為rootfs(root file system).

```bash
$ cd busybox-1.27.1
$ make menuconfig
```

選擇"Busybox Settings"
!["snapshot of busybox #1"][snapshot-1]
選擇"Build Busybox as a static library"
!["snapshot of busybox #2"][snapshot-2]
指定要使用的toolchain prefix
!["snapshot of busybox #3"][snapshot-3]
填入arm-linux-gnueabihf-後存檔離開
!["snapshot of busybox #4"][snapshot-4]

```bash
$ make
$ make install
```

完成後會安裝至_install資料夾. 但要做為rootfs, 還需要做一些處理

+ 建立/dev/console的device node

```bash
$ cd _install
$ mkdir dev
$ cd dev
$ sudo mknod console c 5 1
```

+ 新增/etc/inittab, 內容如下, 詳細的格式可以參考busybox/examples/inittab的說明.

```bash
::sysinit:/etc/rc.local
::respawn:-/bin/sh
```

+ /etc/rc.local則是負責一些system init的相關工作, 例如: mount file systems...等.

```bash
#!/bin/sh

mount -t proc none /proc
```

[busybox]: https://busybox.net/
[snapshot-1]: /images/posts/2017-07-30/busybox-1.png
[snapshot-2]: /images/posts/2017-07-30/busybox-2.png
[snapshot-3]: /images/posts/2017-07-30/busybox-3.png
[snapshot-4]: /images/posts/2017-07-30/busybox-4.png

### Linux Kernel
Linue kernel source package可以在[kernel.org][]下載, 本文使用的版本是4.1.42. 下面是我使用的build script, kernel config選用vexpress.

```bash
#!/bin/sh

export ARCH=arm
export CROSS_COMPILE=arm-linux-gnueabihf-

make distclean
make vexpress_defconfig
make menuconfig
make
```

因為要使用initramfs, kernel config需要做一個修改, config選項路徑:

> General setup
>
>   ---> Initramfs source file(s)

填入前面busybox的安裝路徑
!["snapshot of linux"][snapshot-5]

[kernel.org]: https://www.kernel.org/
[snapshot-5]: /images/posts/2017-07-30/linux.png

### Running on QEMU
最後, 只要使用前面產生的qemu-system-arm和zImage(已包含了rootfs), 搭配上對應的dtb, 就可以成功開機. 以下是我使用的run script.

```bash
#!/bin/sh

BOOT=arch/arm/boot

qemu-system-arm -nographic -M vexpress-a9 -kernel $BOOT/zImage \
	-dtb $BOOT/dts/vexpress-v2p-ca9.dtb \
	-append "console=ttyAMA0 rdinit=/sbin/init" \
	-s
```
