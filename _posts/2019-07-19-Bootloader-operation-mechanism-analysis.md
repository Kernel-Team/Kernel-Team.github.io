---
layout:     post
title:      Bootloader运行机制分析
date:       2019-07-19 14:08:00
summary:    Bootloader mechanism analysis
image:	    "https://github.com/lina-not-linus"
tags: bootloader
author:	    "shipujin"
excerpt_separator: <!--more-->
---

从bootloader的运行机理进行分析启动逻辑。excerpt-separator.<!--more-->

# bootloader 运行机制分析



## 一. bootloader 基础概念

### 1. bootloader 是什么

&emsp; bootloader 是在操作系统内核运行之前运行的一段小程序。通过这段小程序，我们可以初始化硬件设备、建立内存空间映射图，从而将系统的软硬件环境带到合适的状态，以便为最终调试操作系统内核准备好正确的环境。

&emsp; 通常 bootlooader 是严重依赖于硬件而实现的，特别是在嵌入式世界。因此，在嵌入式领域里建立一个通用的 bootloader 几乎是不可能的。尽管如此，我们仍然可以对 bootloader 归纳出一些通用的概念，以指导用户对特定的 bootloader 设计与实现。

### 2. bootLoader的作用，对于 Linux 系统来说，从软件的角度看通常可以分为4个层次。

```
    引导加载程序（boot代码和bootloader）
    Linux内核
    文件系统+（GUI）
    用户应用程序
```

### 3. bootloader 加载过程

&emsp; 引导加载程序是系统加电后运行的第一段软件代码，用于将内核映像从硬盘上读到 RAM 中，实现到核的入口点去运行，即开始启动操作系统。

&emsp; 简单的说，bootloader就是在操作系统内核运行之前运行的一小段程序，通过这段小程序，可以初始化硬件设备，建立内存空间的映射图，从而将系统的软硬件环境带到一个适合的状态，以便为最终调用操作系统内核准备好正确的环境。


## 二. bootloader 的两种模式

&emsp; 从 CPU 的角度来说，上电后并非立刻就可以运行操作系统的，系统的相关硬件必须初始化（CPU 工作模式的设置、内存初始化、关中断、关闭 MMU/Cache 等等），根据当前硬件条件判定当前是启动模式还是下载模式，然后走相应的功能分支。

&emsp; 所以概括 bootloader 的工作就是 —— 初始化系统的软硬件环境，使之满足操作系统的运行条件。当把一切软硬件环境都配置好之后，系统的控制权才会交给 OS 的内核，这个时候bootloader 的工作就结束链。


### 1. bootLoader操作模式：（大多bootloader都包含两种不同的操作模式）

&emsp; 启动模式：bootloader从目标机上的某个固态存储设备上将操作系统加载到RAM中运行，整个过程没有用户的介入。

&emsp; 下载模式：目标机上的Bootloader通过串口连接或联网等方式从主机上下载文件（内核映像、根文件系统映像...），保存到目标机的RAM中，接着再被bootloader写到目标机的Flash类固态存储设备，该模式下通常会向终端用户提供一个简单的命令行接口。

&emsp; bootloader 与主机进行文件传输所用的通用设备及协议:

&emsp; 最常见的情况就是，目标机上的 Boot Loader 通过串口与主机之间进行文件传输，传输协议通常是 xmodem／ymodem／zmodem 协议中的一种。但是，串口传输的速度是有限的，因此通过以太网连接并借助 TFTP 协议来下载文件是个更好的选择。此外，在论及这个话题时，主机方所用的软件也要考虑。比如，在通过以太网连接和 TFTP 协议来下载文件时，主机方必须有一个软件用来的提供 TFTP 服务。
                         

### 2. 常用的bootLoader:

``` 
	ARMBoot, PPCBoot, U-Boot, RedBoot, Blob, Vivi 
```
        
## 三. bootLoader基本原理

### 1. 对比 PC 上 bootloader 不同之处

&emsp; 引导加载程序是系统加电后运行的第一段软件代码。回顾 PC 的体系结构我们可以知道，PC 机中的引导程序由 BIOS(其本质就是一段固件程序)和位于硬盘 MBR 中的 OS bootloader(例如 LILO、GRUB 等)一起组成。BIOS 在完成硬件检测和资源分配后，将硬盘 MBR 中的 bootloader 读到系统的 RAM 中，然后将控制器交给 OS bootloader。bootloader 的主要任务就是将内核映像从硬盘读到 RAM 中，然后跳转到内核的入口点去运行，也即开始启动操作系统。

&emsp; 而在嵌入式系统中，通常并没有像 BIOS 那样的固件程序（有的嵌入式 CPU 也会内嵌一段短小的启动程序），因此整个系统的加载启动任务就完全由 bootloader 来完成。比如在一个基于 ARM7TDMI core 的嵌入式系统中，系统在上电或复位时通常都是从地址 0x00000000 处开始执行，而在这个地址处安排的通常就是系统的 bootloader 程序。


### 2. bootloader 的安装媒介(嵌入式 Installation Medium)

&emsp; 当系统加电或复位后，所有的 CPU 通常都从某个由 CPU 制造商预先安排的地址上取指令。比如，基于 ARM7TDMI core 的 CPU 在复位时通常都从地址 0x00000000 取它的第一条指令。而基于 CPU 构建的嵌入式系统通常都有某种类型的固态存储设备(比如：ROM、EPROM 或 FLASH 等)被映射到这个预先安排的地址上。因此在系统加电后，CPU 将首先执行 bootloader 程序。

&emsp; 是一个同时装有 Boot Loader、内核的启动参数、内核映像和根文件系统映像的固态存储设备的典型空间分配结构图。固态存储设备的典型空间分配结构

|||||
|:---|:---|:---|:---|
|bootloader|boot parameters|Kernel|root filesystem|

### 3. 用来控制 bootloader 的设备或机制

&emsp; 主机和目标机之间一般通过串口建立连接，bootloader 软件在执行时通常会通过串口来进行 I/O，比如：输出打印信息到串口，从串口读取用户控制字符等。

&emsp; Boot Loader 的启动过程是单阶段（Single Stage）还是多阶段（Multi-Stage）
通常多阶段的 Boot Loader 能提供更为复杂的功能，以及更好的可移植性。从固态存储设备上启动的 Boot Loader 大多都是 2 阶段的启动过程，也即启动过程可以分为 stage 1 和 stage 2 两部分。

### 4. bootloader 环境的依赖性

&emsp; （假定内核映像与根文件映像（可以在固态存储设备运行）都被加载到RAM中运行）

&emsp;         通常，bootloader是严重地依赖于硬件而实现的，除了依赖于CPU的体系结构外，bootLoader实际上也依赖于具体的嵌入式板级设备。
        
### 5. bootloader 分为 stage1 和 stage2 两部分

&emsp; 由于bootloader的实现依赖于CPU的体系结构，因此大多数bootloader分为stage1和stage2两个部分:

* stage1：存放依赖于CPU体系结构的代码，如设备初始化代码（通常用汇编语言实现）。

* stage2：存放实现更复杂的功能的代码，通常使用C语言实现，代码具有更好的可读性和可移植性。

```
 * bootloader的stage1通常包括以下步骤：
	（1）、硬件设备初始化
	（2）、为加载bootloader的stage2准备RAM空间
	（3）、复制bootloader的stage2到RAM空间中
	（4）、设置好堆栈
	（5）、跳转到stage2的C入口点
 * bootloader的stage2通常包括以下步骤：
	（1）、初始化本阶段要使用到的硬件设备
	（2）、检测系统内存映射
	（3） 、将内存映像和根文件系统映像从Flash上读到RAM空间中。
	（4）、为内核设置启动参数
	（5）、调用内核
```


## 四. bootloader 的 stage1 阶段细节

### 1. 基本的硬件初始化

	* __屏蔽所有的中断__：为中断提供服务通常是 OS 设备驱动程序的责任，因此在 Boot Loader 的执行全过程中可以不必响应任何中断。中断屏蔽可以通过写 CPU 的中断屏蔽寄存器或状态寄存器（比如 ARM 的 CPSR 寄存器）来完成。

	* __设置 CPU 的速度和时钟频率__：

	* __RAM 初始化__：包括正确地设置系统的内存控制器的功能寄存器以及各内存库控制寄存器等。

	* __初始化 LED__：典型地，通过 GPIO 来驱动 LED，其目的是表明系统的状态是 OK 还是 Error。如果板子上没有 LED，那么也可以通过初始化 UART 向串口打印 Boot Loader 的 Logo 字符信息来完成这一点。

	* __关闭 CPU 内部指令/数据 cache__
	
### 2. 为加载 stage2 准备 RAM 空间

&emsp; 为了获得更快的执行速度，通常把 stage2 加载到 RAM 空间中来执行，因此必须为加载 Boot Loader 的 stage2 准备好一段可用的 RAM 空间范围。

&emsp; 由于 stage2 通常是 C 语言执行代码，因此在考虑空间大小时，除了 stage2 可执行映象的大小外，还必须把堆栈空间也考虑进来。此外，空间大小最好是 memory page 大小(通常是 4KB)的倍数。一般而言，1M 的 RAM 空间已经足够了。具体的地址范围可以任意安排，比如 blob 就将它的 stage2 可执行映像安排到从系统 RAM 起始地址 0xc0200000 开始的 1M 空间内执行。但是，将 stage2 安排到整个 RAM 空间的最顶 1MB(也即(RamEnd-1MB) - RamEnd)是一种值得推荐的方法。

&emsp; 为了后面的叙述方便，这里把所安排的 RAM 空间范围的大小记为：stage2_size(字节)，把起始地址和终止地址分别记为：stage2_start 和 stage2_end(这两个地址均以 4 字节边界对齐)。因此：

```
stage2_end＝stage2_start＋stage2_size
```

&emsp; 另外，还必须确保所安排的地址范围的的确确是可读写的 RAM 空间，因此，必须对你所安排的地址范围进行测试。具体的测试方法可以采用类似于 blob 的方法，也即：以 memory page 为被测试单位，测试每个 memory page 开始的两个字是否是可读写的。为了后面叙述的方便，我们记这个检测算法为：test_mempage，其具体步骤如下：

```
1．先保存 memory page 一开始两个字的内容。
2．向这两个字中写入任意的数字。比如：向第一个字写入 0x55，第 2 个字写入 0xaa。
3．然后，立即将这两个字的内容读回。显然，我们读到的内容应该分别是 0x55 和 0xaa。如果不是，则说明这个 memory page 所占据的地址范围不是一段有效的 RAM 空间。
4．再向这两个字中写入任意的数字。比如：向第一个字写入 0xaa，第 2 个字中写入 0x55。
5．然后，立即将这两个字的内容立即读回。显然，我们读到的内容应该分别是 0xaa 和 0x55。如果不是，则说明这个 memory page 所占据的地址范围不是一段有效的 RAM 空间。
6．恢复这两个字的原始内容。测试完毕。
为了得到一段干净的 RAM 空间范围，我们也可以将所安排的 RAM 空间范围进行清零操作。
```


### 3. 拷贝 stage2 到 ROM

&emsp; 拷贝时要确定两点：(1) stage2 的可执行映象在固态存储设备的存放起始地址和终止地址；(2) RAM 空间的起始地址。

### 4. 设置堆栈指针 sp

&emsp; 堆栈指针的设置是为了执行 C 语言代码作好准备。通常我们可以把 sp 的值设置为(stage2_end-4)，也即在 3.1.2 节所安排的那个 1MB 的 RAM 空间的最顶端(堆栈向下生长)。

&emsp; 此外，在设置堆栈指针 sp 之前，也可以关闭 led 灯，以提示用户我们准备跳转到 stage2。

&emsp; 经过上述这些执行步骤后，系统的物理内存布局应该如下图所示。

![系统内存布局](https://www.ibm.com/developerworks/cn/linux/l-btloader/images/image002.gif)

### 5. 跳转到 stage2 的 C 入口点

&emsp; 在上述一切都就绪后，就可以跳转到 Boot Loader 的 stage2 去执行了。比如，在 ARM 系统中，这可以通过修改 PC 寄存器为合适的地址来实现。bootloader 的 stage2 可执行映象刚被拷贝到 RAM 空间时的系统内存布局如上图。


## 五. bootloader 的 stage2 阶段细节

&emsp; 正如前面所说，stage2 的代码通常用 C 语言来实现，以便于实现更复杂的功能和取得更好的代码可读性和可移植性。但是与普通 C 语言应用程序不同的是，在编译和链接 boot loader 这样的程序时，我们不能使用 glibc 库中的任何支持函数。其原因是显而易见的。这就给我们带来一个问题，那就是从那里跳转进 main() 函数呢？直接把 main() 函数的起始地址作为整个 stage2 执行映像的入口点或许是最直接的想法。但是这样做有两个缺点：1)无法通过main() 函数传递函数参数；2)无法处理 main() 函数返回的情况。一种更为巧妙的方法是利用 trampoline(弹簧床)的概念。也即，用汇编语言写一段trampoline 小程序，并将这段 trampoline 小程序来作为 stage2 可执行映象的执行入口点。然后我们可以在 trampoline 汇编小程序中用 CPU 跳转指令跳入 main() 函数中去执行；而当 main() 函数返回时，CPU 执行路径显然再次回到我们的 trampoline 程序。简而言之，这种方法的思想就是：用这段 trampoline 小程序来作为 main() 函数的外部包裹(external wrapper)。

&emsp; 下面给出一个简单的 trampoline 程序示例(来自blob)：

```
.text
.globl _trampoline
_trampoline:
    bl  main
    /* if main ever returns we just call it again */
    b   _trampoline
```

&emsp; 可以看出，当 main() 函数返回后，我们又用一条跳转指令重新执行 trampoline 程序――当然也就重新执行 main() 函数，这也就是 trampoline(弹簧床)一词的意思所在。

### 1. 初始化本阶段要使用的硬件设备

&emsp; 这通常包括：（1）初始化至少一个串口，以便和终端用户进行 I/O 输出信息；（2）初始化计时器等。

&emsp; 在初始化这些设备之前，也可以重新把 LED 灯点亮，以表明我们已经进入 main() 函数执行。

&emsp; 设备初始化完成后，可以输出一些打印信息，程序名字字符串、版本号等。

### 2. 检测系统的内存映射（memory map）

&emsp; 所谓内存映射就是指在整个 4GB 物理地址空间中有哪些地址范围被分配用来寻址系统的 RAM 单元。比如，在 SA-1100 CPU 中，从 0xC000,0000 开始的 512M 地址空间被用作系统的 RAM 地址空间，而在 Samsung S3C44B0X CPU 中，从 0x0c00,0000 到 0x1000,0000 之间的 64M 地址空间被用作系统的 RAM 地址空间。虽然 CPU 通常预留出一大段足够的地址空间给系统 RAM，但是在搭建具体的嵌入式系统时却不一定会实现 CPU 预留的全部 RAM 地址空间。也就是说，具体的嵌入式系统往往只把 CPU 预留的全部 RAM 地址空间中的一部分映射到 RAM 单元上，而让剩下的那部分预留 RAM 地址空间处于未使用状态。 __由于上述这个事实，因此 Boot Loader 的 stage2 必须在它想干点什么 (比如，将存储在 flash 上的内核映像读到 RAM 空间中) 之前检测整个系统的内存映射情况，也即它必须知道 CPU 预留的全部 RAM 地址空间中的哪些被真正映射到 RAM 地址单元，哪些是处于 "unused" 状态的。__

&emsp; &emsp; (1) 内存映射的描述
可以用如下数据结构来描述 RAM 地址空间中的一段连续(continuous)的地址范围：

```
typedef struct memory_area_struct {
    u32 start; /* the base address of the memory region */
    u32 size; /* the byte number of the memory region */
    int used;
} memory_area_t;
```

&emsp; 这段 RAM 地址空间中的连续地址范围可以处于两种状态之一：(1)used=1，则说明这段连续的地址范围已被实现，也即真正地被映射到 RAM 单元上。(2)used=0，则说明这段连续的地址范围并未被系统所实现，而是处于未使用状态。

&emsp; 基于上述 memory_area_t 数据结构，整个 CPU 预留的 RAM 地址空间可以用一个 memory_area_t 类型的数组来表示，如下所示：

```
memory_area_t memory_map[NUM_MEM_AREAS] = {
    [0 ... (NUM_MEM_AREAS - 1)] = {
        .start = 0,
        .size = 0,
        .used = 0
    },
};
```

&emsp; &emsp; (2) 内存映射的检测

下面我们给出一个可用来检测整个 RAM 地址空间内存映射情况的简单而有效的算法：

```
/* 数组初始化 */
for(i = 0; i < NUM_MEM_AREAS; i++)
    memory_map[i].used = 0;
/* first write a 0 to all memory locations */
for(addr = MEM_START; addr < MEM_END; addr += PAGE_SIZE)
    * (u32 *)addr = 0;
for(i = 0, addr = MEM_START; addr < MEM_END; addr += PAGE_SIZE) {
     /*
      * 检测从基地址 MEM_START+i*PAGE_SIZE 开始,大小为
* PAGE_SIZE 的地址空间是否是有效的RAM地址空间。
      */
     调用3.1.2节中的算法test_mempage()；
     if ( current memory page isnot a valid ram page) {
        /* no RAM here */
        if(memory_map[i].used )
            i++;
        continue;
    }
     
    /*
     * 当前页已经是一个被映射到 RAM 的有效地址范围
     * 但是还要看看当前页是否只是 4GB 地址空间中某个地址页的别名？
     */
    if(* (u32 *)addr != 0) { /* alias? */
        /* 这个内存页是 4GB 地址空间中某个地址页的别名 */
        if ( memory_map[i].used )
            i++;
        continue;
    }
     
    /*
     * 当前页已经是一个被映射到 RAM 的有效地址范围
     * 而且它也不是 4GB 地址空间中某个地址页的别名。
     */
    if (memory_map[i].used == 0) {
        memory_map[i].start = addr;
        memory_map[i].size = PAGE_SIZE;
        memory_map[i].used = 1;
    } else {
        memory_map[i].size += PAGE_SIZE;
    }
} /* end of for (…) */
```

&emsp; 在用上述算法检测完系统的内存映射情况后，Boot Loader 也可以将内存映射的详细信息打印到串口。

### 3. 加载内核映像和根文件系统映像

&emsp; &emsp; (1) 规划内存占用的布局

&emsp; 这里包括两个方面：(1)内核映像所占用的内存范围；（2）根文件系统所占用的内存范围。在规划内存占用的布局时，主要考虑基地址和映像的大小两个方面。

&emsp; 对于内核映像，一般将其拷贝到从(MEM_START＋0x8000) 这个基地址开始的大约1MB大小的内存范围内(嵌入式 Linux 的内核一般都不操过 1MB)。为什么要把从 MEM_START 到 MEM_START＋0x8000 这段 32KB 大小的内存空出来呢？这是因为 Linux 内核要在这段内存中放置一些全局数据结构，如：启动参数和内核页表等信息。

&emsp; 而对于根文件系统映像，则一般将其拷贝到 MEM_START+0x0010,0000 开始的地方。如果用 Ramdisk 作为根文件系统映像，则其解压后的大小一般是1MB。

&emsp; &emsp; (2) 从 Flash 上拷贝

&emsp; 由于像 ARM 这样的嵌入式 CPU 通常都是在统一的内存地址空间中寻址 Flash 等固态存储设备的，因此从 Flash 上读取数据与从 RAM 单元中读取数据并没有什么不同。用一个简单的循环就可以完成从 Flash 设备上拷贝映像的工作：

```
while(count) {
    *dest++ = *src++; /* they are all aligned with word boundary */
    count -= 4; /* byte number */
};
```

### 4. 设置内核启动参数

&emsp; 应该说，在将内核映像和根文件系统映像拷贝到 RAM 空间中后，就可以准备启动 Linux 内核了。但是在调用内核之前，应该作一步准备工作，即：设置 Linux 内核的启动参数。

&emsp; Linux 2.4.x 以后的内核都期望以标记列表(tagged list)的形式来传递启动参数。启动参数标记列表以标记 ATAG_CORE 开始，以标记 ATAG_NONE 结束。每个标记由标识被传递参数的 tag_header 结构以及随后的参数值数据结构来组成。数据结构 tag 和 tag_header 定义在 Linux 内核源码的include/asm/setup.h 头文件中：

```
/* The list ends with an ATAG_NONE node. */
#define ATAG_NONE   0x00000000
struct tag_header {
    u32 size; /* 注意，这里size是字数为单位的 */
    u32 tag;
};
.
.
.
struct tag {
    struct tag_header hdr;
    union {
        struct tag_core     core;
        struct tag_mem32    mem;
        struct tag_videotext    videotext;
        struct tag_ramdisk  ramdisk;
        struct tag_initrd   initrd;
        struct tag_serialnr serialnr;
        struct tag_revision revision;
        struct tag_videolfb videolfb;
        struct tag_cmdline  cmdline;
        /*
         * Acorn specific
         */
        struct tag_acorn    acorn;
        /*
         * DC21285 specific
         */
        struct tag_memclk   memclk;
    } u;
};
```

&emsp; 在嵌入式 Linux 系统中，通常需要由 Boot Loader 设置的常见启动参数有：ATAG_CORE、ATAG_MEM、ATAG_CMDLINE、ATAG_RAMDISK、ATAG_INITRD等。

&emsp; 比如，设置 ATAG_CORE 的代码如下：

```
params = (struct tag *)BOOT_PARAMS;
    params->hdr.tag = ATAG_CORE;
    params->hdr.size = tag_size(tag_core);
    params->u.core.flags = 0;
    params->u.core.pagesize = 0;
    params->u.core.rootdev = 0;
    params = tag_next(params);
```

&emsp; 其中，BOOT_PARAMS 表示内核启动参数在内存中的起始基地址，指针 params 是一个 struct tag 类型的指针。宏 tag_next() 将以指向当前标记的指针为参数，计算紧临当前标记的下一个标记的起始地址。注意，内核的根文件系统所在的设备ID就是在这里设置的。

&emsp; 下面是设置内存映射情况的示例代码：

```
for(i = 0; i < NUM_MEM_AREAS; i++) {
        if(memory_map[i].used) {
            params->hdr.tag = ATAG_MEM;
            params->hdr.size = tag_size(tag_mem32);
            params->u.mem.start = memory_map[i].start;
            params->u.mem.size = memory_map[i].size;
             
            params = tag_next(params);
        }
}
```

&emsp; 可以看出，在 memory_map［］数组中，每一个有效的内存段都对应一个 ATAG_MEM 参数标记。

&emsp; Linux 内核在启动时可以以命令行参数的形式来接收信息，利用这一点我们可以向内核提供那些内核不能自己检测的硬件参数信息，或者重载(override)内核自己检测到的信息。比如，我们用这样一个命令行参数字符串"console=ttyS0,115200n8"来通知内核以 ttyS0 作为控制台，且串口采用 "115200bps、无奇偶校验、8位数据位"这样的设置。下面是一段设置调用内核命令行参数字符串的示例代码：

```
char *p;
    /* eat leading white space */
    for(p = commandline; *p == ' '; p++)
        ;
    /* skip non-existent command lines so the kernel will still
    * use its default command line.
     */
    if(*p == '\0')
        return;
    params->hdr.tag = ATAG_CMDLINE;
    params->hdr.size = (sizeof(struct tag_header) + strlen(p) + 1 + 4) >> 2;
    strcpy(params->u.cmdline.cmdline, p);
    params = tag_next(params);
```

&emsp; 请注意在上述代码中，设置 tag_header 的大小时，必须包括字符串的终止符'\0'，此外还要将字节数向上圆整4个字节，因为 tag_header 结构中的size 成员表示的是字数。

&emsp; 下面是设置 ATAG_INITRD 的示例代码，它告诉内核在 RAM 中的什么地方可以找到 initrd 映象(压缩格式)以及它的大小：

```
params->hdr.tag = ATAG_INITRD2;
params->hdr.size = tag_size(tag_initrd);
 
params->u.initrd.start = RAMDISK_RAM_BASE;
params->u.initrd.size = INITRD_LEN;
 
params = tag_next(params);
```

&emsp; 下面是设置 ATAG_RAMDISK 的示例代码，它告诉内核解压后的 Ramdisk 有多大（单位是KB）：

```
params->hdr.tag = ATAG_RAMDISK;
params->hdr.size = tag_size(tag_ramdisk);
     
params->u.ramdisk.start = 0;
params->u.ramdisk.size = RAMDISK_SIZE; /* 请注意，单位是KB */
params->u.ramdisk.flags = 1; /* automatically load ramdisk */
     
params = tag_next(params);
```

&emsp; 最后，设置 ATAG_NONE 标记，结束整个启动参数列表：

```
static void setup_end_tag(void)
{
    params->hdr.tag = ATAG_NONE;
    params->hdr.size = 0;
}
```

### 5. 调用内核

&emsp; Boot Loader 调用 Linux 内核的方法是直接跳转到内核的第一条指令处，也即直接跳转到 MEM_START＋0x8000 地址处。在跳转时，下列条件要满足：

```
1．CPU 寄存器的设置：
R0＝0；
R1＝机器类型 ID；关于 Machine Type Number，可以参见 linux/arch/arm/tools/mach-types。
R2＝启动参数标记列表在 RAM 中起始基地址；
2．CPU 模式：
必须禁止中断（IRQs和FIQs）；
CPU 必须 SVC 模式；
3．Cache 和 MMU 的设置：
MMU 必须关闭；
指令 Cache 可以打开也可以关闭；
数据 Cache 必须关闭；
```

&emsp; 如果用 C 语言，可以像下列示例代码这样来调用内核：

```
void (*theKernel)(int zero, int arch, u32 params_addr) =
  (void (*)(int, int, u32))KERNEL_RAM_BASE;

theKernel(0, ARCH_NUMBER, (u32) kernel_params_start);
```

&emsp; 注意，theKernel()函数调用应该永远不返回的。如果这个调用返回，则说明出错。

### 四. 关于串口终端

&emsp; 在 boot loader 程序的设计与实现中，没有什么能够比从串口终端正确地收到打印信息能更令人激动了。此外，向串口终端打印信息也是一个非常重要而又有效的调试手段。但是，我们经常会碰到串口终端显示乱码或根本没有显示的问题。造成这个问题主要有两种原因：(1) boot loader 对串口的初始化设置不正确。(2) 运行在 host 端的终端仿真程序对串口的设置不正确，这包括：波特率、奇偶校验、数据位和停止位等方面的设置。

&emsp; 此外，有时也会碰到这样的问题，那就是在 boot loader 的运行过程中我们可以正确地向串口终端输出信息，但当 boot loader 启动内核后却无法看到内核的启动输出信息。对这一问题的原因可以从以下几个方面来考虑

&emsp; &emsp; (1) 首先请确认你的内核在编译时配置了对串口终端的支持，并配置了正确的串口驱动程序。

&emsp; &emsp; (2) 你的 boot loader 对串口的初始化设置可能会和内核对串口的初始化设置不一致。此外，对于诸如 s3c44b0x 这样的 CPU，CPU 时钟频率的设置也会影响串口，因此如果 boot loader 和内核对其 CPU 时钟频率的设置不一致，也会使串口终端无法正确显示信息。

&emsp; &emsp; (3) 最后，还要确认 boot loader 所用的内核基地址必须和内核映像在编译时所用的运行基地址一致，尤其是对于 uClinux 而言。假设你的内核映像在编译时用的基地址是 0xc0008000，但你的 boot loader 却将它加载到 0xc0010000 处去执行，那么内核映像当然不能正确地执行了。

```
bootloader 是在操作系统内核运行之前运行的一段小程序。通过这段小程序，
我们可以初始化硬件设备、建立内存空间映射图，从而将系统的软硬件环境带到
合适的状态，以便为最终调试操作系统内核准备好正确的环境。
```

> Linux 系统启动过程

| 		顺序			|		说明     		|
| :--------------------:|:---------------------:|
|		post			|		加电			|
|		BISO			|		进入BIOS		|
| bootloader(MBR)		|	加载磁盘主引导记录	|
| kernel(ramdisk)		|	加载内核			|
| rootfs			    |	初始化rootfs		|
| /sbin/init		    |系统初始化。这里的 init 在不同系统上还有所不同，systemd 系统守护进程(现在服务管理程序)|



[](参考资料：)

[](https://github.com/torvalds/linux/blob/v4.16/arch/mips/include/asm/mach-loongson64/kernel-entry-init.h)

[](https://0xax.gitbooks.io/linux-insides/Booting/linux-bootstrap-1.html)


[](https://www.jianshu.com/p/65e4d06a6cdb)

[](http://m.yoodao.org/news-show-597.html)

[](https://www.ibm.com/developerworks/cn/linux/l-btloader/index.html)
