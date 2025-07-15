---
layout:     post
title:      "使用BusyBox构建MIPS的ramdisk"
date:       2019-04-04 14:08:00
summary:    Build MIPS-ramdisk using BusyBox
image:	    "https://github.com/lina-not-linus"
tags: "BusyBox ramdisk"
author:	    "shipujin"
---

## 使用 BusyBox 构建 ramdisk

一. 软件准备

```js
主机：龙芯3A3000 
系统：Fedora28_for_loongson-MATE-Live-2.iso [下载地址](http://mirror.lemote.com/fedora/fedora28-live/Fedora28_for_loongson-MATE-Live-2.iso) 
工具链：mips64el-redhat-linux(8.x)
内核：4.19.5-2.fc28.lemote.1.mips64el
BusyBox：1.26.2-3.fc28.lemote.mips64el [下载地址](http://mirror.lemote.com:8000/fedora/releases/28/os/Packages/busybox-1.26.2-3.fc28.lemote.mips64el.rpm)
```

二. 安装 BusyBox，创建基本目录结构


1. 创建基本目录结构

```sh
# mkdir mnt tmp var usr sys proc etc lib dev bin sbin root home
# mkdir usr/bin usr/sbin usr/lib lib/modules
```


2. 下载安装 BusyBox

```sh
dnf install --installroot=`pwd`/ROOTFS busybox

# 安装后目录结构
# ls ROOTFS
bin  dev  etc  home  lib  mnt  proc  root  sbin  sys  tmp  usr  var
```


4. 创建设备节点

* 可以在本系统目录下复制出 /dev 下各设备结点

```sh
# cd dev 
# mknod -m 666 sda	8 0
# mknod -m 666 sda1	8 1
# mknod -m 666 sda2	8 2
# mknod -m 666 console c 5 1 
# mknod -m 666 null c 1 3 
# ln -s fd/0 stdin
# ln -s fd/1 stdout
# ln -s fd/2 stderr
# ln -s vcs0 vcs
# mkdir net pts
```

三. 链接出命令工具

1. 首先查看 BusyBox 所有工具命令

```sh
./busybox  --list
```

2. 链接出所有命令

```sh
cd ROOTFS/sbin/
./busybox --list >listCmd.txt
for line1 in $(cat listCmd.txt)
do      
        ln -sv busybox ${line1}
done
cd ../;rm -rf usr/bin usr/sbin;rm -rf bin/
cp -rf sbin/ bin/;cp -rf sbin/ bin/ usr/ 
```

四. 实验制作好的基本 buysox 工具

```sh
cd ROOTFS;chroot . /bin/sh
```

* 若出现以下现象，则表明"/usr/bin"下无此工具，或软链接路径有问题(若完全按上面链接路径做，不会出现路径问题)，亦或者busybox二进制文件有损坏等。

```sh
/bin/sh: xxx : not found
```

五. etc/下创建配置文件

1. 创建配置文件

```sh
touch busybox.conf group inittab motd passwd  fb.modes resolv.conf
touch fstab mtab profile shadow ld.so.cache  ld.so.conf
```
2. 创建配置目录 init.d

```sh
# init.d是从busybox源代码目录下拷贝出
cp -R examples/bootflopyp/etc/init.d /mnt/rootfs/etc/
```

* 创建 fstab 配置文件

```sh
# fstab
/dev/fd0 /  ext2  defaults 0 0
none  /proc  proc  defaults 0 0
none  /dev/pts devpts  mode=0622 0 0
```

* 创建 group 配置文件，若此 ramdisk 下有其他用户、或者服务用户(例如tftp等)加入其中

```sh
root:x:0:
```

* 创建 inittab 配置文件

```sh
::sysinit:/etc/init.d/rcS
::respawn:/sbin/getty 115200 ttyS0
::respawn:/bin/sh
::askfirst:/bin/sh
# Stuff to do when restarting the init process
::restart:/sbin/init
# Stuff to do before rebooting
::ctrlaltdel:/sbin/reboot
::shutdown:/bin/umount -a -r
::shutdown:/sbin/swapoff -a

```

* 创建 passwd 配置文件

```sh
root::0:0:root:/root:/bin/sh
```

* 创建 profile 配置文件

```sh
# /etc/profile: system-wide .profile file for the Bourne shells
export PS1="BusyBox@[\w]>"
alias ll='ls -l'11 ll
alias du='du -h'
alias df='df -h'
alias rm='rm -i'
export PATH=/mnt/bin:$PATH
```

* 创建 resolv.conf 配置文件

```sh
# resolv.conf：dns域名解析
nameserver 127.0.0.1
search localhost
```

* 创建 shadow 配置文件，根据实际情况设置

* 创建 init.d/rcS 配置文件

```sh
# 可以参考 busybox 文档修改

#! /bin/sh
mount -o remount,rw /

/sbin/ifconfig lo 127.0.0.1
#/sbin/ifconfig eth0 192.168.1.2

/bin/mount -a
>/etc/mtab
echo
echo "+++++++++++++ Welcom to Lianic Arm Linux ++++++++++++++++++ "
echo "+ This is a arm linux system."

echo "+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++ "
echo
hostname RAMDISK

#chmod a+x rcS
```


* 创建 mtab 配置文件

```sh
none /proc proc  0 0
n /sys sysfs  0 0
```

* 建立 lib 模块目录

```sh
cp -Rfa  /usr/lib/modules/版本号 usr/lib/modules/ 
```

六. busybox 的用途

1. 可以编译到内核，制作运行中内存中的系统。
2. 代替系统中的 initrd、initramfs。
3. 系统中的工具包集合。

