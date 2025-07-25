---
layout:     post
title:      "交叉编译"
date:       2019-03-19 14:08:00
summary:    交叉编译配置，并解释了三元组
image:	    "https://github.com/lina-not-linus"
tags: "cross_compile"
author:	    "shipujin"
excerpt_separator: <!--more-->
---

深入理解交叉编译(Cross Compile)  <!--more-->
## 深入理解交叉编译(Cross Compile)



> 首先你要了解一下，三个名词："build, haost, target"，和三元组。
> build：构建 gcc 编译器的平台系统环境，编译该软件使用的平台。
> host:：是执行 gcc 编译器的平台系统环境，该软件运行的平台。
> target：是让 gcc 编译器产生能在什么格式运行的平台的系统环境，该软件处理的目标平台。
> 三元组：架构-设备厂家-位

一. build、host、target的三种组合

- build与host不同是交叉编译器；build与target不同是交叉编译链；三者都相同则为本地编译。

1. 指定：- -build=X86,  - -host=X86, - -target=X86

   使用X86下构建X86的gcc编译器，编译出能在X86下运行的程序。



2. 指定：- -build=X86,  - -host=X86, - -target=MIPS

   在X86下交叉编译出能在MIPS下运行的可执行程序。

   

3. 指定：- -build=X86,  - -host=MIPS, - -target=X86

   在X86下构建 gcc交叉编译器，在MIPS上运行 gcc交叉编译器，编译出能在 ARM 上运行的可执行程序。

   

4. 指定：- -build=X86,  - -host=ARM, - -target=MIPS

   在X86下构建 gcc交叉编译器，在ARM上运行 gcc交叉编译器，编译出能在 MIPS 运行的可执行程序。



二. 构建 MIPS 交叉编译链



1. 指定ABI变量

   | ABI  | CLF_ABI = Value |               Notes               |
   | :--: | :-------------: | :-------------------------------: |
   | O32  |       32        |             32 位CPU              |
   | N32  |       N32       | 对于在 32 位模式下运行 64 位的CPU |
   | N64  |       64        | 对于在 64 位模式下运行 64 位的CPU |

   

```js
export CLFS_ABI="[From Chart]"
echo export CLFS_ABI=\""${CLFS_ABI}\"" >> ~/.bashrc
```




2. 设置三元组变量

   |      位与大小端      | 三元组(target triplet) | MIPS 级别 |
   | :------------------: | :--------------------: | --------- |
   | MIPS 32 bit 小端对齐 |   mipsel-linux-musl    | 1         |
   | MIPS 32 bit 大端对齐 |    mips-linux-musl     | 1         |
   | MIPS 64 bit 小端对齐 |  mips64el-linux-musl   | 3         |
   | MIPS 64 bit 大端对齐 |   mips64-linux-musl    | 3         |

   

 

```js
# 指定host、target
export CLFS_HOST=$(echo ${MACHTYPE} | sed "s/-[^-]*/-cross/")
export CLFS_TARGET="[target triplet]"

# 实验判断大小端
export CLFS_ARCH=mips
export CLFS_ENDIAN=$(echo ${CLFS_ARCH} | sed -e 's/mipsel/little/' -e 's/mips/big/')

# 指定 mips 指令级别
export CLFS_MIPS_LEVEL="[mips level]"

# 指定软浮点还是硬浮点，loongson是软浮点
export CLFS_FLOAT="[hard or soft]"

# 把指定好的三元组和变量写入用户级配置文件内
echo export CLFS_HOST=\""${CLFS_HOST}\"" >> ~/.bashrc
echo export CLFS_TARGET=\""${CLFS_TARGET}\"" >> ~/.bashrc
echo export CLFS_ARCH=\""${CLFS_ARCH}\"" >> ~/.bashrc
echo export CLFS_ENDIAN=\""${CLFS_ENDIAN}\"" >> ~/.bashrc
echo export CLFS_MIPS_LEVEL=\""${CLFS_MIPS_LEVEL}\"" >> ~/.bashrc
echo export CLFS_FLOAT=\""${CLFS_FLOAT}\"" >> ~/.bashrc
```



3. 导出内核头文件，编译时需要用到系统函数调用

   ```js
   make mrproper
   make ARCH=${CLFS_ARCH} headers_check
   make ARCH=${CLFS_ARCH} INSTALL_HDR_PATH=${CLFS}/cross-tools/${CLFS_TARGET} headers_install
   ```

4. 编译汇编工具 Binutils

   - target：指定跨平台的架构
   - disable-multilib：关闭多库

```js
../binutils/configure \
   --prefix=${CLFS}/cross-tools \
   --target=${CLFS_TARGET} \
   --with-sysroot=${CLFS}/cross-tools/${CLFS_TARGET} \
   --disable-nls \
   --disable-multilib

make configure-host
make
make install
```



5. 第一次编译gcc
   - 指定build、host、target
   - without-shard：禁止共享库的创建
   - without-headers：禁止使用 C 库中的任何头文件，因为尚未构建 C 库，防止使用宿主机的 C 介入
   - with-newlib：这个是重要的，告诉编译器不使用任何 C 库下编译 gcc
   - enable-languages=c：表示仅构建 C 编译器
   - with-abi：指定之前设置好的ABI
   - with-arch：设置mips体系结构的ISA
   - with-endian：指定字节序

```js
../gcc/configure \
  --prefix=${CLFS}/cross-tools \
  --build=${CLFS_HOST} \
  --host=${CLFS_HOST} \
  --target=${CLFS_TARGET} \
  --with-sysroot=${CLFS}/cross-tools/${CLFS_TARGET} \
  --disable-nls  \
  --disable-shared \
  --without-headers \
  --with-newlib \
  --disable-decimal-float \
  --disable-libgomp \
  --disable-libmudflap \
  --disable-libssp \
  --disable-libatomic \
  --disable-libquadmath \
  --disable-threads \
  --enable-languages=c \
  --disable-multilib \
  --with-mpfr-include=$(pwd)/../gcc-6.2.0/mpfr/src \
  --with-mpfr-lib=$(pwd)/mpfr/src/.libs \
  --with-abi=${CLFS_ABI} \
  --with-arch=mips${CLFS_MIPS_LEVEL} \
  --with-float=${CLFS_FLOAT} \
  --with-endian=${CLFS_ENDIAN}
  
make all-gcc all-target-libgcc
make install-gcc install-target-libgcc
```



6. 编译 musl

```js
./configure \
  CROSS_COMPILE=${CLFS_TARGET}- \
  --prefix=/ \
  --target=${CLFS_TARGET}

make
DESTDIR=${CLFS}/cross-tools/${CLFS_TARGET} make install
```



7. 最后的 gcc

   

```js
../gcc-6.2.0/configure \
  --prefix=${CLFS}/cross-tools \
  --build=${CLFS_HOST} \
  --host=${CLFS_HOST} \
  --target=${CLFS_TARGET} \
  --with-sysroot=${CLFS}/cross-tools/${CLFS_TARGET} \
  --disable-nls \
  --enable-languages=c \
  --enable-c99 \
  --enable-long-long \
  --disable-libmudflap \
  --disable-multilib \
  --with-mpfr-include=$(pwd)/../gcc-6.2.0/mpfr/src \
  --with-mpfr-lib=$(pwd)/mpfr/src/.libs \
  --with-abi=${CLFS_ABI} \
  --with-arch=mips${CLFS_MIPS_LEVEL} \
  --with-float=${CLFS_FLOAT} \
  --with-endian=${CLFS_ENDIAN}
  
  make
  make install
```



8. cross-tools使用配置



```js
echo export CC=\""${CLFS_TARGET}-gcc --sysroot=${CLFS}/targetfs\"" >> ~/.bashrc
echo export CXX=\""${CLFS_TARGET}-g++ --sysroot=${CLFS}/targetfs\"" >> ~/.bashrc
echo export AR=\""${CLFS_TARGET}-ar\"" >> ~/.bashrc
echo export AS=\""${CLFS_TARGET}-as\"" >> ~/.bashrc
echo export LD=\""${CLFS_TARGET}-ld --sysroot=${CLFS}/targetfs\"" >> ~/.bashrc
echo export RANLIB=\""${CLFS_TARGET}-ranlib\"" >> ~/.bashrc
echo export READELF=\""${CLFS_TARGET}-readelf\"" >> ~/.bashrc
echo export STRIP=\""${CLFS_TARGET}-strip\"" >> ~/.bashrc
source ~/.bashrc
```

