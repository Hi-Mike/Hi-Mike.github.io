---
layout: post
title: "Ubuntu下编译FFmpeg"
date: 2016-12-21
categories: ffmpeg
---

## 环境
NDK
>由于本篇的目的是要为Android编译可用的so库，因此需要用到NDK交叉编译。

## 下载FFmpeg
可以到官网[FFmpeg](http://www.ffmpeg.org/)下载，或者[GitHub镜像](https://github.com/FFmpeg/FFmpeg)，直接利用Git管理，也方便以后更新版本（初次下载较慢）

```git
git clone https://github.com/FFmpeg/FFmpeg.git
```

## 配置编译
### 修改configure文件
将

```bash
SLIBNAME_WITH_MAJOR='$(SLIBNAME).$(LIBMAJOR)'
LIB_INSTALL_EXTRA_CMD='$$(RANLIB) "$(LIBDIR)/$(LIBNAME)"'
SLIB_INSTALL_NAME='$(SLIBNAME_WITH_VERSION)'
SLIB_INSTALL_LINKS='$(SLIBNAME_WITH_MAJOR) $(SLIBNAME)'
```

改为

```bash
SLIBNAME_WITH_MAJOR='$(SLIBPREF)$(FULLNAME)-$(LIBMAJOR)$(SLIBSUF)'
LIB_INSTALL_EXTRA_CMD='$$(RANLIB)"$(LIBDIR)/$(LIBNAME)"'
SLIB_INSTALL_NAME='$(SLIBNAME_WITH_MAJOR)'
SLIB_INSTALL_LINKS='$(SLIBNAME)'
```

这样避免编译出Android不识别的以数字结尾的so文件，变为以.so结尾

### 建立编译脚本

```bash
#!/bin/bash

cd ffmpeg
make clean

#NDK路径
export NDK=/home/mike/android-ndk-r13
export TOOLCHAIN=$NDK/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64
export PLATFORM=$NDK/platforms/android-14/arch-arm
export PREFIX=../simplefflib

ADDI_CFLAGS="-marm"
ADDI_LDFLAGS=""

build_one(){
  ./configure \
	--prefix=$PREFIX \
	--target-os=linux \
	--enable-shared \
	--disable-static \
	--disable-doc \
	--disable-ffmpeg \
	--disable-ffplay \
	--disable-ffprobe \
	--disable-ffserver \
	--disable-symver \
	--enable-avresample \
	--enable-small \
	--cross-prefix=$TOOLCHAIN/bin/arm-linux-androideabi- \
	--arch=arm \
	--enable-cross-compile \
	--sysroot=$PLATFORM \
	--extra-cflags="-Os -fpic $ADDI_CFLAGS" \
	--extra-ldflags="$ADDI_LDFLAGS"
}

build_one

make
make install

cd ..
```

这样的配置编译出的库会很大，这里只编译了arm平台的库，在实际使用时会将很多不需要的编译选项去掉。具体的配置可以通过 ./configure --help查看。

## 编译带其他组件的FFmpeg库（如x264）
由于一些库与FFmpeg的licence不兼容，一些性能无敌的库并没有被默认包含在FFmpeg中，需要手动编译添加。

[x264地址](http://www.videolan.org/developers/x264.html)

```bash
#!/bin/bash

cd x264

make clean

export NDK=/home/mike/android-ndk-r13
export SYSROOT=$NDK/platforms/android-14/arch-arm
export TOOLCHAIN=$NDK/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64

CPU=arm
PREFIX=../x264lib
ADDI_CFLAGS="-marm"
ADDI_LDFLAGS=""

build_arm(){
  ./configure \
--prefix=$PREFIX \
--enable-shared \
--disable-asm \
--enable-pic \
--enable-strip \
--host=arm-linux-androideabi \
--cross-prefix=$TOOLCHAIN/bin/arm-linux-androideabi- \
--sysroot=$SYSROOT
--extra-cflags="-Os -fpic $ADDI_CFLAGS" \
--extra-ldflags="$ADDI_LDFLAGS"
}

build_arm
make
make install

cd ..
```

x264的编译与之前的编译差不多。

重新编译FFmpeg，需要在原来的文件中添加配置，其中cflags等的地址根据路径添加

```bash
--enable-decoder=h264 \
--enable-gpl \
--enable-libx264 \
--enable-yasm \
--extra-cflags="-I ../android-lib/include" \
--extra-ldflags="-L ../android-lib/lib"
```

>说明：在新的FFmpeg中faac的aac库被移除了，只保留了官方的aac库与fdk-aac，可以选择添加

## 测试
编译FFmpeg主要是为了在Android下使用，所以建了一个利用cmake构建的Android工程。其他配置以后再写，说一下开始遇到的问题。

在测试代码中直接调用了 **avcodec_register_all();** ，结果编译的时候一直显示 **undefined reference to** 这个函数。期间了解了各种链接顺序不对，调整了库的顺序依然如此，最后拿别人编译的库过来完全没问题。看了编译文件完全没什么毛病，只有configure文件我私自改的不一样。我把编译的库名称没有留版本号，最后差不多修改为最终的版本，测试才通过。

## 参考
[最简单的基于FFmpeg的移动端例子：Android HelloWorld](http://blog.csdn.net/leixiaohua1020/article/details/47008825)

[初识FFmpeg编译那些事](http://zhengxiaoyong.me/2016/11/13/%E5%88%9D%E8%AF%86FFmpeg%E7%BC%96%E8%AF%91%E9%82%A3%E4%BA%9B%E4%BA%8B/)

[FFmpeg wiki](https://trac.ffmpeg.org/)

[ffmpeg-android](https://github.com/WritingMinds/ffmpeg-android)