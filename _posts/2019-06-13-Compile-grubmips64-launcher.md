---
layout:     post
title:      构建mips64el架构的grub-efi
date:       2019-06-13 14:08:00
summary:    编译 64-bit grub-mips.efi 启动映像
image:	    "https://github.com/lina-not-linus"
tags: grub
author:	    "shipujin"
excerpt_separator: <!--more-->
---


编译grub2-mips启动器，制作 MIPS 架构的 grubmips64el.efi. <!--more-->

## 编译grub2-mips启动器，制作 MIPS 架构的 grubmips64el.efi 
##### Compile the 64-bit grub-mips.efi launcher


一. 实验环境

```shell
主机：龙芯3A3000 1701
系统：Fedora28_for_loongson-MATE-Live-2.iso [下载地址](http://mirror.lemote.com/fedora/fedora28-live/Fedora28_for_loongson-MATE-Live-2.iso) 
工具链：mips64el-redhat-linux(8.x)
内核：4.19.5-2.fc28.lemote.1.mips64el
grub64-efi 源代码下载地址： [下载地址](https://github.com/loongson-community/grub.git)
```

> 其实龙芯平台下的固件已具有 bootloader 功能，最后 grub 只有一个二级引导的作用了(grub.efi)


二.  grub 介绍

1. grub 是什么？

   * GNU GRUB 是引导加载程序、启动器，可以加载各种各样的操作系统，以及带有链式加载的专有操作系统， 它是解决启动个人计算机的复杂性。引导加载程序是计算机启动运行的第一个软件程序。它负责将控制加载和转移到操作系统内核软件（例如 Linux 内核），之后内核初始化操作系统的其余部分。
   * GNU GRUB 的一个重要的特性是灵活性；GRUB 知道文件系统和内核可执行格式，因此使用者可以按照自己喜欢的方式加载任意操作系统，而无须记录内核在磁盘上的物理位置。因此只需指定其文件名以及内核所在的驱动器和分区即可加载内核。

2. 磁盘分区命名习惯和文件路径表示方法

   * grub 对设备与分区的命名规则: 磁盘从"0"开始计数，分区从"1"开始计数

     ```shell
     (fd0)           ：表示第一块软盘
     (hd0,msdos2)    ：表示第一块硬盘的第二个mbr分区。grub2中分区从1开始编号，传统的grub是从0开始编号的
     (hd0,msdos5)    ：表示第一块硬盘的第一个逻辑分区
     (hd0,gpt1)      ：表示第一块硬盘的第一个gpt分区
     /boot/vmlinuz   ：相对路径，基于根目录，表示根目录下的boot目录下的vmlinuz，
                     ：如果设置了根目录变量root为(hd0,msdos1)，则表示(hd0,msdos1)/boot/vmlinuz
     (hd0,msdos1)/boot/vmlinuz：绝对路径，表示第一硬盘第一分区的boot目录下的vmlinuz文件
     ```

     

   * 文件的命名方法: (1)绝对路径表示法，(2)相对路径表示法

     ```shell
     (fd0)/grldr                 第一软盘根目录下的"grldr"文件[绝对路径]
     (hd0,gpt1)/boot/vmlinuz     第一硬盘的第一GPT分区"boot"目录下的"vmlinuz"文件[绝对路径]
     /boot/vmlinuz               根设备"boot"目录下的"vmlinuz"文件[相对路径]，
                                 当"root"环境变量等于"(hd0,gpt1)"时，等价于"(hd0,gpt1)/boot/vmlinuz"
     ```

     

   * 磁盘块的命名方法：(1)绝对路径表示法，(2)相对路径表示法。

     ```shell
     (hd1,1)0+1  在第二硬盘的第一分区上，从第"0"个磁盘块(首扇区)起，长度为"1"的连续块。[绝对路径]
     (hd1,1)+1   含义与上一个相同，因为当从第"0"个磁盘块(首扇区)起时，"0"可以省略不写。[绝对路径]
     +1          在根设备上，从第"0"个磁盘块(首扇区)起，长度为"1"的连续块。[相对路径]
                 当"root"环境变量等于"(hd1,1)"时，等价于"(hd1,1)0+1"
     ```

3. 引导操作系统的两种方式

   * 直接引导操作系系统

     ```shell
     使用命令：boot
     ```

   * 链式装载：链式加载另一个加载程序，然后另一个加载程序实际加载操作系统。

     ```shell
     # 使用命令：chainloader
     # 要加一些GRUB模块并设置适当的根设备，以 Windows 为例：
     menuentry "Windows" {
     	insmod chain
     	insmod ntfs
     	set root=(hd0,1)
     	chainloader +1
     }
     ```

三. 在x86架构下，对grub1 与 grub2 引导阶段剖析(再去结合龙芯PMON、昆仑固件、UEFI固件分析启动过程，因为龙芯固件已包含 bootloader 功能，对比分析)

> 使用dd读取前512字节的内容写到MBR.in文件，然后使用od采取十六进制格式、ASCII打印 MBR.in文件内容
> 运行以下内容即可在终端观察到系统MBR的内容：
> dd if=/dev/sda of=MBR.in bs=512 count=1
> od -xa MBR.in 


1. 老式 grub 启动的三个阶段：stage1、stage1.5、stage2

   * staege1: 主要负责当 BIOS 交给 grub 时，载入存在于各分区中的开机文件，也就是所谓的开机管理程序，/boot/grub 中的 stage1 文件大小为 512 Byte（提醒下MBR大小也是 512 Byte），stage1文件其实就是 MBR 中的 bootloader 的备份文件，或者说管理程序的备份文件（管理程序有可能装在 bootsector 中）。bootloader 只有前 446 Byte 和 MBR 一样，紧随之后的 64 Byte 则和现存的 MBR 没有关系，MBR 的接下来的 64 Byte 是 Partition Table，记录分区信息，MBR 最后 2 Byte 是 Magic Number 55AA 目的只是让存在于 bootloader 区的管理程序辨认扇区时，可以确认所存储的这个地方就是 MBR，有点像 MBR 的标识。 

     ```shell
     # 使用dd读取前512字节的内容写到MBR.in文件，然后使用od采取十六进制格式、ASCII
     # 打印 MBR.in文件内容
     运行以下内容即可在终端观察到系统MBR的内容：
     dd if=/dev/sda of=MBR.in bs=512 count=1
     od -xa MBR.in 
     ```


   * stage1.5: stage1.5 阶段是连接 stage1 到 stage2 的一个信道，里面唯一存放着的是该系统文件的格式。在 grub 目录下有若干 stage1.5 的文件，如 e2fs_stage1_5、fat_stage1_5、jfs_stage1_5、reiserfs_stage1_5，这些都属于 stage1.5 阶段功能的文件。

     * stage1 在加载 stage1.5 后，比如 e2fs_stage1_5(ext2 的文件系统)，就可以识别 ext2 文件系统的格式。在 stage1 加载 stage1.5 之后，stage1 就可以识别 ext2  并将 stage2 加载。因为在 stage1.5 被加载时，就已经赋予 GRUB 访问文件系统目录的能力，所以，自然就可以在开始找不到 stage2 的情况下，从文件系统目录中找出 stage2 的所在位置，并激活 Linux 。

   * stage2: stage2 文件是 GRUB 的核心程序，能让用户以选项方式将操作系统加载、新增参数、添加选项，这些全都是 stage2 的功能。对 GRUB 来说，stage2 除了不能自己激活外，剩余的事情全被 stage2 包了。stage2 文件存放在各分区的 Bootsector 中，主要功能为：

     * 提供选项
     * 访问设置文件
     * 连接下一个 boot sector

2. grub2 与 grub 的区别

   ```
   1.配置文件的名称改变了。在grub中，配置文件为grub.conf或menu.lst(grub.conf的一个软链接)，在grub2中改名为grub.cfg。
   2.grub2增添了许多语法，更接近于脚本语言了，例如支持变量、条件判断、循环。
   3.grub2中，设备分区名称从1开始，而在grub中是从0开始的。
   4.grub2使用img文件，不再使用grub中的stage1、stage1.5和stage2。
   5.支持图形界面配置grub，但要安装grub-customizer包，epel源提供该包。
   6.在已进入操作系统环境下，不再提供grub命令，也就是不能进入grub交互式界面，只有在开机时才能进入，算是一大缺憾。
   7.在grub2中没有了好用的find命令，算是另一大缺憾。
   ```


3. 在x86架构下，分析bootloader 与 新老 版本 grub 关系(再去结合龙芯PMON、昆仑固件、UEFI固件分析启动过程，因为龙芯固件已包含 bootloader 功能)

   * img文件是grub2生成的，stage文件是传统grub生成的。

   * 当使用grub来管理启动菜单时，那么boot loader都是grub程序安装的。

   * 传统的grub将stage1转换后的内容安装到MBR(VBR或EBR)中的boot loader部分，将stage1_5转换后的内容安装在紧跟在MBR后的扇区中，将stage2转换后的内容安装在/boot分区中。

   * grub2将boot.img转换后的内容安装到MBR(VBR或EBR)中的boot loader部分，将diskboot.img和kernel.img结合成为core.img，同时还会嵌入一些模块或加载模块的代码到core.img中，然后将core.img转换后的内容安装到磁盘的指定位置处。

     ![引用骏马金龙 grub2 ](/images/grub2-image-Relationship.png)

4. 在 x86 架构下编译 grub2 后身成一些 .img 文件 和 mod 文件，其 img 各文件的作用为(结合 grub 的三个stage 阶段分析 )：

   * img 列表
     
     ![grub2 img list](/images/grub2-img-list.png)
     
   * boot.img
     
     * 在BIOS平台下，boot.img是grub启动的第一个img文件，它被写入到MBR中或分区的boot sector中，因为boot sector的大小是512字节，所以该img文件的大小也是512字节
     * boot.img唯一的作用是读取属于core.img的第一个扇区并跳转到它身上，将控制权交给该扇区的img。由于体积大小的限制，boot.img无法理解文件系统的结构，因此grub2-install将会把core.img的位置硬编码到boot.img中，这样就一定能找到core.img的位置。
     
   * core.ing
     * core.img根据diskboot.img、kernel.img和一系列的模块被grub2-mkimage程序动态创建。core.img中嵌入了足够多的功能模块以保证grub能访问/boot/grub，并且可以加载相关的模块实现相关的功能，例如加载启动菜单、加载目标操作系统的信息等，由于grub2大量使用了动态功能模块，使得core.img体积变得足够小。
     * core.img中包含了多个img文件的内容，包括diskboot.img/kernel.img等。
     * core.img的安装位置随MBR磁盘和GPT磁盘而不同，这在文中分区类型中已经说明过了。
     
   * diskboot.img

     * 如果启动设备是硬盘，即从硬盘启动时，core.img中的第一个扇区的内容就是diskboot.img。diskboo.img的作用是读取core.img中剩余的部分到内存中，并将控制权交给kernel.img，由于此时还不识别文件系统，所以将core.img的全部位置以block列表的方式编码，使得diskboot.img能够找到剩余的内容。
     * 该img文件因为占用一个扇区，所以体积为512字节。

   * cdboot.img

     * 如果启动设备是光驱(cd-rom)，即从光驱启动时，core.img中的第一个扇区的的内容就是cdboo.img。它的作用和diskboot.img是一样的。

   * pexboot.img

     * 如果是从网络的PXE环境启动，core.img中的第一个扇区的内容就是pxeboot.img。

   * kernel.img

     * kernel.img文件包含了grub的基本运行时环境：设备框架、文件句柄、环境变量、救援模式下的命令行解析器等等。很少直接使用它，因为它们已经整个嵌入到了core.img中了。注意，kernel.img是grub的kernel，和操作系统的内核无关
     * 如果细心的话，会发现kernel.img本身就占用28KB空间，但嵌入到了core.img中后，core.img文件才只有26KB大小。这是因为core.img中的kernel.img是被压缩过的。

   * lnxboot.img

     * 该img文件放在core.img的最前部位，使得grub像是linux的内核一样，这样core.img就可以被LILO的"image="识别。当然，这是配合LILO来使用的，但现在谁还适用LILO呢？

   * *.mod

     * 各种功能模块，部分模块已经嵌入到core.img中，或者会被grub自动加载，但有时也需要使用insmod命令手动加载。

5. 老版本的 grub stage 阶段与 grub img 对比

   * stage1 
     * stage1文件在功能上等价于boot.img文件。目的是跳转到stage1_5或stage2的第一个扇区上。
   * *_stage1_5
     * *stage1_5文件包含了各种识别文件系统的代码，使得grub可以从文件系统中读取体积更大功能更复杂的stage2文件。从这一方面考虑，它类似于core.img中加载对应文件系统模块的代码部分，但是core.img的功能远比stage1_5多。
     * stage1_5一般安装在MBR后、第一个分区前的那段空闲空间中，也就是MBR gap空间，它的作用是跳转到stage2的第一个扇区。
     * 其实传统的grub在某些环境下是可以不用stage1_5文件就能正常运行的，但是grub2则不能缺少core.img。
   * stage2
     * stage2的作用是加载各种环境和加载内核，在grub2中没有完全与之相对应的img文件，但是core.img中包含了stage2的所有功能
     * 当跳转到stage2的第一个扇区后，该扇区的代码负责加载stage2剩余的内容。
     * 注意，stage2是存放在磁盘上的，并没有像core.img一样嵌入到磁盘上。
   * stage2_eltorito
     * 功能上等价于grub2中的core.img中的cdboot.img部分。一般在制作救援模式的grub时才会使用到cd-rom相关文件
   * pxegrub
     * 功能上等价于grub2中的core.img中的pxeboot.img部分。

6. 只要有/boot/grub2/i386-pc下的img文件就一定能通过grub2相关程序再次生成boot loader(写入到磁盘中)。



四. 分区类型选择(X86情况下)

1. MBR 分区：这种格式允许四个主分区和额外的逻辑分区，使用这种格式的分区表。有两种方式安装GURB：

   * 嵌入到MBR和第一个分区中间的空间，这部分就是大众所称的"boot track","MBR gap"或"embedding area"，它们大致需要31kB的空间；

   * 将core.img安装到某个文件系统中，然后使用分区的第一个扇区(严格地说不是第一个扇区，而是第一个block)存储启动它的代码。

   * 使用嵌入的方式安装grub，就没有保留的空闲空间来保证安全性，例如有些专门的软件就是使用这段空间来实现许可限制的；另外分区的时候，虽然会在MBR和第一个分区中间留下空闲空间，但可能留下的空间会比这更小。

   * 方法二安装grub到文件系统，但这样的grub是脆弱的。例如，文件系统的某些特性需要做尾部包装，甚至某些fsck检测，它们可能会移动这些block

   * GRUB开发团队建议将GRUB嵌入到MBR和第一个分区之间，除非有特殊需求，但仍必须要保证第一个分区至少是从第31kB(第63个扇区)之后才开始创建的。

   * 现在的磁盘设备，一般都会有分区边界对齐的性能优化提醒，所以第一个分区可能会自动从第1MB处开始创建。

     

2. GPT 分区

   * 一些新的系统使用GUID分区表(GPT)格式，这种格式是EFI固件所指定的一部分。但如果操作系统支持的话，GPT也可以用于BIOS平台(即MBR风格结合GPT格式的磁盘)，使用这种格式，需要使用独立的BIOS boot分区来保存GRUB，GRUB被嵌入到此分区，不会有任何风险。
   * 当在gpt磁盘上创建一个BIOS boot分区时，需要保证两件事：
     * (1)它最小是31kB大小，但一般都会为此分区划分1MB的空间用于可扩展性；
     * (2)必须要有合理的分区类型标识(flag type)。


五. 制作 grubmips64el.efi

1. 编译64位 mips grub2 

   ```shell
   git clone https://github.com/loongson-community/grub.git
   cd grub
   bash autogen.sh
   ./configure --prefix=安装目录
   make ; make install
   ```


2. grub64.efi 制作脚本

   ```
   grub-mkimage -p /boot/EFI/BOOT/  -d  mips64el-efi/  -c grub.cfg  -o grub.efi-config-3  -O mips64el-efi 后跟所要加载的模块
   ```

   * 参数含义

     ```
     -d 表示指定查找模块目录
     -c 表示指定配置文件，这个配置文件会集成到efi文件内，就是我们刚刚编写的grub.cfg
     -p 设置偏好文件夹，cfg文件中会调
     -o 表示生成的目标文件
     -O 表示集成的模块
     ```

3. GRUB2 模块介

   - **grub2生成了的 mod 文件，分布在/usr/lib64/grub/mips64el-efi目录下、/boot/grub2/mips64el-efi目录下。**
   - 命令模块[command.list]
     - 提供了各种不同的功能，类似标准 Unix 命令，一共将近 100 个。例如：cat cpuid echo halt lspci chainloader initrd linux password ...
   - 加密模块[crypto.list]
     - 提供了各种数据完整性校验与密码算法支持，一共 20 多个。例如：gcry_rijndael crc64 fcry_md5 ...
   - 文件系统模块[fs.list]
     - 提供了访问各种文件系统的功能，一共 30 多个。例如：btrfs cpio exfat ext2 fat iso9660 ntfs tar xfs zfs ...
   - 分区模块[partmap.list]
     - 提供了识别各种分区格式的功能，一共 10 多个。例如：part_bsd part_gpt part_msdo ...
   - 分区工具[parttool.list]
     - 提供了操作各种分区格式的功能，目前只有 msdospart 这一个。
   - 终端模块[terminal.list]
     - 提供各种不同终端的支持，一共不到 10 个。例如：serial gfxterm vga_text at_keyboard ...
   - 视频模块[video.list]
     - 提供各种不同的视频模式支持，一共 6 个。例如：vga vbe efi_gop efi_uga ...
   - 其他模块：所有未在上述分类文件中列出的模块都归为这一类，一共将近 100 个。值得关注的有以下几个：
     - “all_video” 可用于一次性加载当前所有可用的视频模块；
     - “gfxmenu” 可用于提供主题支持；
   - “jpeg png tag" 可用于提供特定格式的背景图片支持；
     - “xzio gzio lzopio” 可用于提供特定压缩格式支持(常配合“initrd”命令使用)；

4. 制作 grubmips64el.efi 脚本(下面我把所有的模块都加载进 grub.efi 文件，但有个弊端就是会影响到系统启动)

   ```shell
   grub-mkimage -p /boot/EFI/BOOT/  -d  mips64el-efi/  -c grub.cfg  -o grub.efi-config-3  -O mips64el-efi acpi adler32 affs afs all_video archelp bfs bitmap bitmap_scale blocklist boot bswap_test btrfs bufio cat cbfs chain cmdline_cat_test cmp cmp_test configfile cpio_be cpio crc64 cryptodisk crypto ctz_test datehook date datetime diskfilter disk div div_test dm_nv echo efifwsetup efi_gop efinet elf eval exfat exfctest ext2 extcmd fat file font fshelp functional_test gcry_arcfour gcry_blowfish gcry_camellia gcry_cast5 gcry_crc gcry_des gcry_dsa gcry_idea gcry_md4 gcry_md5 gcry_rfc2268 gcry_rijndael gcry_rmd160 gcry_rsa gcry_seed gcry_serpent gcry_sha1 gcry_sha256 gcry_sha512 gcry_tiger gcry_twofish gcry_whirlpool geli gettext gfxmenu gfxterm_background gfxterm_menu gfxterm gptsync gzio halt hashsum hello help hexdump hfs hfspluscomp hfsplus http iso9660 jfs jpeg keystatus ldm linux loadenv loopback lsacpi lsefimmap lsefi lsefisystab lsmmap ls lssal luks lvm lzopio macbless macho mdraid09_be mdraid09 mdraid1x memdisk memrw minicmd minix2_be minix2 minix3_be minix3 minix_be minix mmap mpi msdospart mul_test net newc nilfs2 normal ntfscomp ntfs odc offsetio part_acorn part_amiga part_apple part_bsd part_dfly part_dvh part_gpt part_msdos part_plan part_sun part_sunpc parttool password password_pbkdf2 pbkdf2 pbkdf2_test png priority_queue probe procfs progress raid5rec raid6rec read reboot regexp reiserfs relocator romfs scsi search_fs_file search_fs_uuid search_label search serial setjmp setjmp_test sfs shift_test signature_test sleep sleep_test squash4 syslinuxcfg tar terminal terminfo test_blockarg testload test testspeed tftp tga time trig tr true udf ufs1_be ufs1 ufs2 verify video_colors video_fb videoinfo video videotest_checksum videotest xfs xnu_uuid xnu_uuid_test xzio zfscrypt zfsinfo zfs
   ```


5. grub.cfg 制作 grubmips64el.efi 的配置文件

   ```shell
   search.file /boot/EFI/BOOT/grub.cfg root
   set prefix=($root)/boot/
   configfile ($root)/boot/EFI/BOOT/grub.cfg
   ```
   

6. 启动参数

   ```shell
   救援模式：systemd.unit=rescue.target
   0 运行级别：systemd.unit=poweroff.target
   1,s,single 运行级别(单用户模式)：systemd.unit=rescue.target
   2,4 运行级别(多用户模式，通常识别为级别3，多用户、无图形界面。用户可以通过终端或网络登录)：systemd.unit=multi-user.target
   5 运行级别(多用户、图形界面。继承级别 3 的服务，并启动图形界面服务)：systemd.unit=graphical.target
   6 运行级别(重启)：systemd.unit=reboot.target
   emergency 运行级别(急救模式[emergency shell])：systemd.unit=emergency.target
   rdinit=/sbin/init
   ```

7. 生成 grub 配置文件 grub.cfg

   ```shell
   # 根据/etc/default/grub.d/下的文件来创建配置文件
   grub2-mkconfig -o /boot/grub2/grub.cfg
   ```


8. grub 全局配置文件 /etc/default/grub

   ```shell
   GRUB_TIMEOUT=5
   GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
   GRUB_DEFAULT=saved   
   GRUB_DISABLE_SUBMENU=true
   GRUB_TERMINAL_OUTPUT="gfxterm"
   GRUB_CMDLINE_LINUX="rhgb quiet"
   GRUB_DISABLE_RECOVERY="true"
   
   # GRUB_TIMEOUT=5 开机选择项的超时时间
   # GRUB_DEFAULT=saved 设置为 saved 则是上一次登录的系统条目，可设置“0-N” 0为最上面一条系统条目 
   # GRUB_CMDLINE_LINUX="rhgb quiet" 添加到菜单中的内核启动参数
   ```



九. 参考资料

* [GNU GRUB 手册](<https://www.gnu.org/software/grub/manual/grub/html_node>)
* [GRUB2配置文件"grub.cfg"详解](<http://www.jinbuguo.com/linux/grub.cfg.html>)
* 《Linux 操作系统之奥秘》
