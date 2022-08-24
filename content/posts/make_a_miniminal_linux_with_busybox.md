---
title: "使用busybox制作rootfs"
date: 2022-08-22T21:43:34+08:00
---

### before you start
1. download [busybox](https://busybox.net/) and [linux kernel](https://kernel.org/) source code.
2. create some necessary directories.
    ~~~bash
    mkdir initrd
    cd initrd
    mkdir {proc,sys,etc}
    ~~~
    then create `init` file
    ~~~bash
    #!/bin/sh
    mount -t proc proc /proc
    mount -t sysfs sysfs /sys
    mknod /dev/ttyS0 c 4 64
    setsid sh -c 'exec sh </dev/ttyS0 >/dev/ttyS0 2>&1'
    ~~~
> note:  
don't forget to make `init` executable
### 1. compile linux kernel
here, I use default configuratonfile, you can tweak it with gui through `make menuconfig` or manually configure `.config` file. you can get more help from `make help`.
~~~bash
make defconfig
make -j $(nproc)
make install
~~~

### 2. compile busybox
~~~bash
make defconfig
sed -i 's/#\s*\(CONFIG_STATIC\).*/\1=y/g' .config
make -j $(nproc)
make install CONFIG_PREFIX=../initrd
~~~

### 3. make virtual file system
~~~bash
find . | cpio -o -H newc | gzip -9 > initramfs.img
~~~
or
~~~bash
find . -print0 | cpio -0 -o -H newc | gzip -9 > initramfs.img
~~~

### 4. boot linux with qemu
~~~bash
qemu-system-x86_64 \
    -kernel kernel/bzImage \
    -initrd initrd/initramfs.img \
    -nographic \
    -append "panic=1 console=ttyS0"
~~~

> alternatively you can pack rootfs into a disk  
> ~~~bash
> dd if=/dev/zero of=rootfs.img bs=1M count=10
> mkfs.ext4 rootfs.img
> mkdir rootfs
> sudo mount rootfs.img rootfs
> # rootfs directory
> mkdir {proc,sys}
> # busybox directory
> make install CONFIG_PREFIX=../rootfs
> sudo umount rootfs
> ~~~
> then boot with `qemu`
> ~~~bash
> qemu-system-x86_64 \
>     -kernel kernel/bzImage \
>     -hda rootfs.img \
>     -append "root=/dev/sda console=ttyS0" \
>     -nographic
> ~~~


**method to quit qemu**
* `Ctrl + A` then `X`
* `Ctrl + A` then `C` then type `quit`

---

### reference
1. [Build a minimal Linux with only Busybox in 1 hour](https://www.youtube.com/watch?v=asnXWOUKhTA)
2. [How to build bare minimal Linux system | Busybox, RAMfs ( initrd ) & system boot with Linux Kernel](https://www.youtube.com/watch?v=c4j6z2huJxs)
3. [Building Bare-Minimum Linux for QEMU](https://www.youtube.com/watch?v=MBx3JPgHYfI)
4. [从头制作一个基于 Busybox 的小型 Linux 系统](https://www.bilibili.com/video/BV1wg411k7J1?spm_id_from=333.337.search-card.all.click)
