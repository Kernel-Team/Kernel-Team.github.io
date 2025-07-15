---
layout:     post
title:      "构建交叉工具链"
date:       2019-05-31 14:08:00
summary:    Build a cross-tool chain
image:	    "https://github.com/lina-not-linus"
tags: cross_toolchain
author:	    "shipujin"
excerpt_separator: <!--more-->
---

自动化构建 MIPS 交叉工具链 <!--more-->

## 自动化构建 MIPS 交叉工具链

> 推荐下载地址：https://github.com/loongson-community

一. 系统环境

1. 宿主机环境

```js
  主机: X86
  系统：Redhat 8
  工具链：x86_64-redhat-linux(8.x)
  内核：4.18.0-80.el8.x86_64
```

2. 软件选择，及下载相关 patch 文件

> 推荐下载地址：https://github.com/loongson-community

    * binutils-2.30.tar.xz
    * gcc-8.1.0.tar.xz
    * glibc-2.27.tar.xz
    * linux-4.0.6.tar.xz
    * gdb-8.1.tar.xz


二. 自动化脚本

1. 工具链安装位置配置

```
set -e

sudo chmod 0777 /opt
sudo rm -rf /opt/mips64el-toolchain
```

2. 内核头文件提取

```
# linux
pushd .
cd linux
INSTALL_DIR="${PWD}/install"
tar xf linux-4.0.6.tar.xz
cd linux-4.0.6
make ARCH=mips INSTALL_HDR_PATH=${INSTALL_DIR} headers_install
find ${INSTALL_DIR} -name .install -delete
find ${INSTALL_DIR} -name ..install.cmd -delete
popd
```

3. 第一遍编译 gcc

```
# libstdc++ abi: new | gcc4-compatible
LIBSTDCXX_ABI=gcc4-compatible

# gcc stage0
pushd .
cd gcc
INSTALL_DIR="${PWD}/install"
tar xf gcc-8.1.0.tar.xz
cd gcc-8.1.0
for p in ../*.patch; do
        patch -Np1 -i ${p}
done
cd ..
mkdir build
cd build
../gcc-8.1.0/configure --prefix=/opt/mips64el-toolchain --build=x86_64-unknown-linux-gnu --host=x86_64-unknown-linux-gnu --target=mips64el-unknown-linux-gnu  --with-sysroot=/opt/mips64el-toolchain/platforms/current --enable-languages=c --disable-shared --disable-threads --disable-nls --enable-multilib --with-newlib --with-default-libstdcxx-abi=${LIBSTDCXX_ABI} --with-linker-hash-style=both --with-arch=loongson3a --with-abi=64 --without-odd-spreg-32
make all-gcc all-target-libgcc -j`nproc`
make DESTDIR=${INSTALL_DIR} install-strip-gcc install-strip-target-libgcc
rm -rf ${INSTALL_DIR}/opt/mips64el-toolchain/include
cp -a ${INSTALL_DIR}/opt/mips64el-toolchain /opt/
popd
```

4. 编译 glibc

```
rm -rf /opt/mips64el-toolchain/platforms/current

mkdir -p /opt/mips64el-toolchain/platforms/current/usr/
cp -a linux/install/* /opt/mips64el-toolchain/platforms/current/usr/

# glibc linux-2.6
pushd .
cd glibc
tar xf glibc-2.27.tar.xz
cd glibc-2.27
for p in ../*.patch; do
	patch -Np1 -i ${p}
done
cd ..
# n64
INSTALL_DIR="${PWD}/linux-2.6/install-n64"
rm -rf build
mkdir build
cd build
export CFLAGS="-march=loongson3a -mabi=64 -O2"
export CXXFLAGS="-march=loongson3a -mabi=64 -O2"
cp ../config-n64.cache config.cache
../glibc-2.27/configure --prefix=/usr --build=x86_64-unknown-linux-gnu --host=mips64el-unknown-linux-gnu --disable-profile --enable-add-ons --with-tls --enable-kernel=2.6.32 --with-__thread --cache-file=config.cache
make -j`nproc`
make DESTDIR=${INSTALL_DIR} install
mv ${INSTALL_DIR}/sbin/* ${INSTALL_DIR}/usr/sbin
mv ${INSTALL_DIR}/usr/lib/* ${INSTALL_DIR}/usr/lib64
rm -rf ${INSTALL_DIR}/{sbin,lib,usr/lib}
ln -sf usr/bin ${INSTALL_DIR}/bin
ln -sf usr/sbin ${INSTALL_DIR}/sbin
ln -sf usr/lib ${INSTALL_DIR}/lib
ln -sf usr/lib32 ${INSTALL_DIR}/lib32
ln -sf usr/lib64 ${INSTALL_DIR}/lib64
sed -i -e 's|lib\/|lib64\/|g' ${INSTALL_DIR}/usr/lib64/libc.so ${INSTALL_DIR}/usr/lib64/libpthread.so
cp -a ${INSTALL_DIR}/* /opt/mips64el-toolchain/platforms/current/
# n32
cd  ..
INSTALL_DIR="${PWD}/linux-2.6/install-n32"
rm -rf build
mkdir build
cd build
export CFLAGS="-march=loongson3a -mabi=n32 -O2"
export CXXFLAGS="-march=loongson3a -mabi=n32 -O2"
cp ../config-n32.cache config.cache
../glibc-2.27/configure --prefix=/usr --build=x86_64-unknown-linux-gnu --host=mips64el-unknown-linux-gnu --disable-profile --enable-add-ons --with-tls --enable-kernel=2.6.32 --with-__thread --cache-file=config.cache
make -j`nproc`
make DESTDIR=${INSTALL_DIR} install
mv ${INSTALL_DIR}/usr/lib/* ${INSTALL_DIR}/usr/lib32
rm -rf ${INSTALL_DIR}/{etc,sbin,var}
rm -rf ${INSTALL_DIR}/usr/{bin,sbin,share,lib,libexec}
sed -i -e 's|lib\/|lib32\/|g' \
       -e 's|elf64-tradlittlemips|elf32-ntradlittlemips|g' ${INSTALL_DIR}/usr/lib32/libc.so ${INSTALL_DIR}/usr/lib32/libpthread.so
cp -a ${INSTALL_DIR}/* /opt/mips64el-toolchain/platforms/current/
# o32
cd  ..
INSTALL_DIR="${PWD}/linux-2.6/install-o32"
rm -rf build
mkdir build
cd build
export CFLAGS="-march=loongson3a -mabi=32 -O2"
export CXXFLAGS="-march=loongson3a -mabi=32 -O2"
cp ../config-o32.cache config.cache
../glibc-2.27/configure --prefix=/usr --build=x86_64-unknown-linux-gnu --host=mips64el-unknown-linux-gnu --disable-profile --enable-add-ons --with-tls --enable-kernel=2.6.32 --with-__thread --cache-file=config.cache
make -j`nproc`
make DESTDIR=${INSTALL_DIR} install
rm -rf ${INSTALL_DIR}/{etc,sbin,var}
rm -rf ${INSTALL_DIR}/usr/{bin,sbin,share,libexec}
sed -i -e 's|elf64-tradlittlemips|elf32-tradlittlemips|g' ${INSTALL_DIR}/usr/lib/libc.so ${INSTALL_DIR}/usr/lib/libpthread.so
cp -a ${INSTALL_DIR}/* /opt/mips64el-toolchain/platforms/current/
popd

mv /opt/mips64el-toolchain/platforms/current /opt/mips64el-toolchain/platforms/linux-2.6
ln -sf linux-2.6 /opt/mips64el-toolchain/platforms/current
echo "Finish ..."
```

5. 第二遍编译 gcc

```
# libstdc++ abi: new | gcc4-compatible
LIBSTDCXX_ABI=gcc4-compatible

# gcc stage1
pushd .
cd gcc
INSTALL_DIR="${PWD}/install"
rm -rf ${INSTALL_DIR}
rm -rf gcc-8.1.0
tar xf gcc-8.1.0.tar.xz
cd gcc-8.1.0
for p in ../*.patch; do
        patch -Np1 -i ${p}
done
cd ..
rm -rf build
mkdir build
cd build
../gcc-8.1.0/configure --prefix=/opt/mips64el-toolchain --build=x86_64-unknown-linux-gnu --host=x86_64-unknown-linux-gnu --target=mips64el-unknown-linux-gnu  --with-sysroot=/opt/mips64el-toolchain/platforms/current --enable-languages=c,c++,objc,obj-c++,fortran,go,lto --enable-shared --enable-threads --enable-checking=release --disable-nls --enable-multilib --with-newlib --with-default-libstdcxx-abi=${LIBSTDCXX_ABI} --enable-__cxa_atexit --disable-libunwind-exceptions --enable-gnu-unique-object --enable-linker-build-id --with-linker-hash-style=both --enable-plugin --enable-initfini-array --disable-libgcj --enable-gnu-indirect-function --with-long-double-128 --with-arch=loongson3a --with-abi=64 --without-odd-spreg-32
make all-gcc all-target-libgcc -j`nproc`
make DESTDIR=${INSTALL_DIR} install-strip-gcc install-strip-target-libgcc
mkdir -p ${INSTALL_DIR}/opt/mips64el-toolchain/mips64el-unknown-linux-gnu/include/
ln -sf ../../platforms/current/usr/include/c++ ${INSTALL_DIR}/opt/mips64el-toolchain/mips64el-unknown-linux-gnu/include/c++
rm -rf ${INSTALL_DIR}/opt/mips64el-toolchain/include
cp -a ${INSTALL_DIR}/opt/mips64el-toolchain /opt/
popd
```

6. 编译 c++ 库

```
# libstdc++ abi: new | gcc4-compatible
LIBSTDCXX_ABI=gcc4-compatible

# gcc libstdc++
pushd .
cd gcc
INSTALL_DIR="${PWD}/install"
rm -rf ${INSTALL_DIR}
rm -rf gcc-8.1.0
tar xf gcc-8.1.0.tar.xz
cd gcc-8.1.0
for p in ../*.patch; do
        patch -Np1 -i ${p}
done
cd ..
rm -rf build
mkdir build
cd build
../gcc-8.1.0/libstdc++-v3/configure --prefix=/usr --build=x86_64-unknown-linux-gnu --host=mips64el-unknown-linux-gnu --target=mips64el-unknown-linux-gnu --enable-shared --disable-nls --enable-multilib --with-default-libstdcxx-abi=${LIBSTDCXX_ABI} 
make -j`nproc`
make DESTDIR=${INSTALL_DIR} install-strip
cp -a ${INSTALL_DIR}/* /opt/mips64el-toolchain/platforms/linux-2.6/
popd
```


7. 编译 gdb

```
# gdb
pushd .
cd gdb
INSTALL_DIR="${PWD}/install"
rm -rf ${INSTALL_DIR}
tar xf gdb-8.1.tar.xz
cd gdb-8.1
for p in ../*.patch; do
        patch -Np1 -i ${p}
done
cd ..
rm -rf build
mkdir build
cd build
../gdb-8.1/configure --prefix=/opt/mips64el-toolchain --build=x86_64-unknown-linux-gnu --host=x86_64-unknown-linux-gnu --target=mips64el-unknown-linux-gnu --enable-languages=c,c++ --disable-nls --enable-multilib --enable-interwork
make -j`nproc`
make DESTDIR=${INSTALL_DIR} install
cp -a ${INSTALL_DIR}/opt/mips64el-toolchain /opt/
popd
```
> end
