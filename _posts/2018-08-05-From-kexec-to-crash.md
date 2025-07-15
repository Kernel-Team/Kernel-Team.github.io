---
layout:     post
title:      Linux 内核转储机制
date:       2018-05-08 14:08:00
summary:    Linux kernel dump mechanism analysis
image:	    "https://github.com/lina-not-linus"
tags: crash
author:	    "shipujin"
excerpt_separator: <!--more-->
---


Linux kernel dump mechanism analysis <!--more-->
## Linux kernel dump mechanism analysis

* 想要了解 Linux 内核转储机制，就要从 kexec - > kdump - > crash 这个递进过程去分析。

&emsp;&emsp;只有清楚他们在各个过程负责的任务、功能，搞通了工作原理、功能后，就去实际环境中使用，最后再去分析其源码，只有这样你是真正的知道了整个转储机制。

##### 一. 工具介绍及其所负责任务

1. kexec
2. kdump
3. crash

###### 1. kexec工具

* kexec是内核更换，免于再次经历固件的工具，节省了内核开发者的时间。kexec实现了在一个内核里启动另一个内核。

###### 安装

```sh
yum install kexec-tools
```

kexec两种使用方式

```sh
kexec -l /boot/vmlinuz+  --initrd=/boot/initramfs+  --append="root=/dev/sdaX ro"
```

##### 2. kdump

* kdump是内核转储工具，kexec是实现kdump机制的关键。kdump是基于kexec的内核崩溃转储机制。内存转存机制。

![引用自：深入探索 Kdump，第 1 部分：带你走进 Kdump 的世界 过程逻辑图](/images/crash20180319.webp)

###### 安装

```sh
# yum install kexec-tools
# yum install kernel-debuginfo #包管理器会解决其依赖common包
```

* 需要检测本系统内核是否已经选中支持kexec system call，若在“/boot/config-XXXXX”中CONFIG_KEXEC=y则是本版本号的内核已开启；若=n，则需要重新编译内核，选中CONFIG_KEXEC。

* 替换内核的两个步骤，第一个把新内核的模块目录（在编译内核时创建一个临时目录安装编译好内核的模块）

```
# 编译好内核之后，建立一个临时目录lib
mkdir -pv lib-kexecKdump-kernel
# 安装内核模块
make modules_install INSTALL_MOD_PATH=./lib/
```

* 复制lib/下编译好内核版本号目录的模块，到/lib/modules/下。

一. 内核崩溃转储的本质

* 当系统崩溃时，kdump利用kexec启动另一个内核，这一个内核叫捕获内核，以很小的内存启动捕获内核，第一个内核保留了内存的一部分给捕获内核启动用。kdump利用kexec启动捕获内核，免去启动BIOS(绕过BIOS，没有经历BIOS，节省了系统启动的时间)，所以第一个内核的内存得以保留。

* 捕获内核只会使用分配给他的内存空间，不会污染第一内核的内存数据。


二. kdump由两部分组成

1. 内核空间的系统调用，由kexec_load在生产内核启动时把捕获内核加载到指定地址。

2. 用户空间的工具kexec-tools，将捕获内核的地址传递给生产内核，在系统崩溃时能找到捕获内核地址，并运行它。

三. 如何使用kdump

1. 定制自定义的转储捕获内核（编译内核前开启kdump、vmcore），即捕获内核。

2. 或者将第一个内核本身作为转储捕获内核使用，但此方法只支持可重定位内核的体系结构。

四. 观察前一个内核内存方式

1. 通过/dev/oldmem设备接口。

2. 通过/proc/vmcore

五.修改内核引导参数，为启动捕获内核预留内存

1. 在grub启动时，修改/boot/grub.cfg;若EFI启动则在/boot/efi/EFI/fedora/grub.cfg。其实只是在里面加入了捕获内核的预留内存值的关键字“crashkernel=X@Y”，修改位置在类似：“linux linuxefi /vmlinuz-4.17.7-200.fc28.x86_64 root=/dev/mapper/fedora-root ro”之后的位置，X：捕获内核预留的内存大小，Y：代表保留内存的起始位置地址。

* 要设置好“Y保留内存的起始位置地址”，则需要知道你编译捕获内核时的内核载入地址（会在在编译内核开启kdump、vmcore时的第二行会有设置修改）。

```sh
### 修改后的文件
menuentry 'Fedora (4.17.7-200.fc28.x86_64) 28 (Workstation Edition)' --class fedora --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-4.16.3-301.fc28.x86_64-advanced-56130196-332b-4c5d-8387-67bba2d45054' {
    load_video
    set gfxpayload=keep
    insmod gzio
    insmod part_gpt
    insmod ext2
    set root='hd0,gpt2'
    if [ x$feature_platform_search_hint = xy ]; then
      search --no-floppy --fs-uuid --set=root crashkernel=auto  --hint-bios=hd0,gpt2 --hint-efi=hd0,gpt2 --hint-baremetal=ahci0,gpt2  9cc3b942-1abe-4b8e-98fb-c0cd1186223e
    else
      search --no-floppy --fs-uuid --set=root 9cc3b942-1abe-4b8e-98fb-c0cd1186223e
    fi
    linuxefi /vmlinuz-4.17.7-200.fc28.x86_64 root=/dev/mapper/fedora-root ro resume=/dev/mapper/fedora-swap crashkernel=auto   rd.lvm.lv=fedora/root rd.lvm.lv=fedora/swap rhgb quiet LANG=zh_CN.UTF-8
    initrdefi /initramfs-4.17.7-200.fc28.x86_64.img
}
```

2. 开启kdump服务

	* x86平台：

	```sh
	systemctl start kdump && systemctl enable kdump
	```

	* mips平台：不用自己手动启动此服务。

3. 第一个内核启动后，载入转储捕获内核

载入转储捕获内核，命令规范：

```sh
   kexec -p <dump-capture-kernel-vmlinux-image> \
   --initrd=<initrd-for-dump-capture-kernel> --args-linux \
   --append="root=<root-dev> <arch-specific-options>"
```

* 当执行“kexec -p”后，可能会有root没有挂载的情况。是因为initrd参数未起作用，然后在“root=”变量使用设备号（/dev/sdaX），不要使用“hd0，msdogs”或“UUID”，则可以成功。若系统装有crash，则“/var/crash/”有vmcore;若未装vmcore存在于“/proc/vmcore”。

4. 测试配置是否有效

* 使用sysrq-c中断将系统崩溃，让kdump捕获崩溃时产生的vmcore，默认存放位置在/var/crash/host@time，配置文件/etc/dump.conf。

```sh
	echo c > /proc/sysrq-trigger
```

* 成功，重新进入稳定系统。若系统装有crash，则“/var/crash/”有vmcore;若未装vmcore存在于“/proc/vmcore”。

* 编译内核时小技巧

 ```sh
 make menconfig
 ```

* 启用或者禁用某项时，可按“/”搜索某项模块或服务，搜索结果会显示出此服务的地址，与此服务模块依赖的模块是否启用。

* 参考文献：linux kdump.txt：https://www.kernel.org/doc/Documentation/kdump/kdump.txt

##### crash

* 对内核崩溃转储文件做分析的工具，更好的使用此工具，请认真阅读man crash。

```
crash  kernel-debuginfo vmcore
```

