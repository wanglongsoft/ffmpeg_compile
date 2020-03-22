#### 1. 交叉编译
交叉编译，是在一个平台上生成另一个平台上的可执行代码。

#### 2. 编译资源
[NDK r17c](https://developer.android.google.cn/ndk/downloads/older_releases)  
[FFmpeg 4.2.2](http://ffmpeg.org/download.html#releases)  
[RTMP](https://github.com/yixia/librtmp)  

#### 3. 准备工作
#### 3.1 注释clang，采用gcc编译  
修改FFmpeg源码根目录：configure文件  
```java
//4210 - 4213（行号）
# set_default target_os
# if test "$target_os" = android; then
#     cc_default="clang"
# fi
```
##### 3.2 关闭FFmpeg自身的rtmp(实现不完整)  
注释FFmpeg自身的rtmp  
```java
// 6256行
# enabled librtmp           && require_pkg_config librtmp librtmp librtmp/rtmp.h RTMP_Socket
```
#### 4. 配置脚本
##### 4.1 RTMP编译
```java
//x86平台
#!/bin/bash

NDK_ROOT=/Users/wanglong/MyProject/android_sdk/Android/sdk/ndk/android-ndk-r17c

CPU=x86

TOOLCHAIN=$NDK_ROOT/toolchains/$CPU-4.9/prebuilt/darwin-x86_64

export XCFLAGS="-isysroot $NDK_ROOT/sysroot -isystem $NDK_ROOT/sysroot/usr/include/i686-linux-android -D__ANDROID_API__=21"
export XLDFLAGS="--sysroot=${NDK_ROOT}/platforms/android-21/arch-x86"
export CROSS_COMPILE=$TOOLCHAIN/bin/i686-linux-android-

make clean
make -j16

make install SYS=android prefix=`pwd`/result/x86 CRYPTO=  SHARED=  XDEF=-DNO_SSL

//arm64-v8a平台
#!/bin/bash

NDK_ROOT=/Users/wanglong/MyProject/android_sdk/Android/sdk/ndk/android-ndk-r17c

CPU=aarch64-linux-android

TOOLCHAIN=$NDK_ROOT/toolchains/$CPU-4.9/prebuilt/darwin-x86_64

export XCFLAGS="-isysroot $NDK_ROOT/sysroot -isystem $NDK_ROOT/sysroot/usr/include/aarch64-linux-android -D__ANDROID_API__=21"
export XLDFLAGS="--sysroot=${NDK_ROOT}/platforms/android-21/arch-arm64"
export CROSS_COMPILE=$TOOLCHAIN/bin/aarch64-linux-android-

make clean
make -j16

make install SYS=android prefix=`pwd`/result/arm64-v8a CRYPTO=   SHARED= XDEF=-DNO_SSL
```
##### 4.2 FFmpeg编译脚本
```java
//x86平台
#!/bin/bash

#NDK_ROOT 变量指向ndk目录
NDK_ROOT=/Users/wanglong/MyProject/android_sdk/Android/sdk/ndk/android-ndk-r17c
#TOOLCHAIN 变量指向ndk中的交叉编译gcc所在的目录
TOOLCHAIN=$NDK_ROOT/toolchains/x86-4.9/prebuilt/darwin-x86_64

RTMP=/Users/wanglong/MyFile/ffmpeg/RTMP/librtmp-master/result/x86
#此变量用于编译完成之后的库与头文件存放在哪个目录
PREFIX=./android/x86-ffmpeg-rtmp-shared

#执行configure脚本，用于生成makefile
#--prefix : 安装目录
#--enable-small : 优化大小
#--disable-programs : 不编译ffmpeg程序(命令行工具)，我们是需要获得静态(动态)库。
#--disable-avdevice : 关闭avdevice模块，此模块在android中无用
#--disable-encoders : 关闭所有编码器 (播放不需要编码)
#--disable-muxers :  关闭所有复用器(封装器)，不需要生成mp4这样的文件，所以关闭
#--disable-filters :关闭视频滤镜
#--enable-cross-compile : 开启交叉编译
#--cross-prefix: gcc的前缀 xxx/xxx/xxx-gcc 则给xxx/xxx/xxx-
#disable-shared enable-static 不写也可以，默认就是这样的。
#--sysroot: 
#--extra-cflags: 会传给gcc的参数
#--arch --target-os : 必须要给

#  需要修改configure， 注释：enable-librtmp(6256行)
#
#
#--extra-cflags="-isysroot $NDK_ROOT/sysroot -isystem $NDK_ROOT/sysroot/usr/include/i686-linux-android -D__ANDROID_API__=21 -march=i686 -g -DANDROID -ffunction-sections -funwind-tables -fstack-protector-strong -no-canonical-prefixes -Wa,--noexecstack -Wformat -Werror=format-security -O0 -fPIC -I $RTMP/include" \
#
./configure \
--prefix=$PREFIX \
--enable-small \
--enable-gpl \
--enable-nonfree \
--disable-programs \
--disable-ffmpeg \
--disable-ffplay \
--disable-ffprobe \
--disable-doc \
--disable-symver \
--disable-asm \
--enable-cross-compile \
--enable-librtmp \
--cross-prefix=$TOOLCHAIN/bin/i686-linux-android- \
--disable-static \
--enable-shared \
--sysroot=$NDK_ROOT/platforms/android-21/arch-x86/ \
--extra-cflags="-isysroot $NDK_ROOT/sysroot -isystem $NDK_ROOT/sysroot/usr/include/i686-linux-android -D__ANDROID_API__=21 -g -DANDROID -ffunction-sections -funwind-tables -fstack-protector-strong -no-canonical-prefixes -mstackrealign -Wa,--noexecstack -Wformat -Werror=format-security -O0 -fPIC -I $RTMP/include" \
--arch=x86 \
--extra-ldflags="-L $RTMP/lib" \
--extra-libs="-l rtmp" \
--target-os=android

#上面运行脚本生成makefile之后，使用make执行脚本

make clean
make -j16
make install

//arm64-v8a平台
#!/bin/bash

#NDK_ROOT 变量指向ndk目录
NDK_ROOT=/Users/wanglong/MyProject/android_sdk/Android/sdk/ndk/android-ndk-r17c
#TOOLCHAIN 变量指向ndk中的交叉编译gcc所在的目录
TOOLCHAIN=$NDK_ROOT/toolchains/aarch64-linux-android-4.9/prebuilt/darwin-x86_64

#此变量用于编译完成之后的库与头文件存放在哪个目录
PREFIX=./android/arm64-v8a-ffmpeg-rtmp-shared
RTMP=/Users/wanglong/MyFile/ffmpeg/RTMP/librtmp-master/result/arm64-v8a

#执行configure脚本，用于生成makefile
#--prefix : 安装目录
#--enable-small : 优化大小
#--disable-programs : 不编译ffmpeg程序(命令行工具)，我们是需要获得静态(动态)库。
#--disable-avdevice : 关闭avdevice模块，此模块在android中无用
#--disable-encoders : 关闭所有编码器 (播放不需要编码)
#--disable-muxers :  关闭所有复用器(封装器)，不需要生成mp4这样的文件，所以关闭
#--disable-filters :关闭视频滤镜
#--enable-cross-compile : 开启交叉编译
#--cross-prefix: gcc的前缀 xxx/xxx/xxx-gcc 则给xxx/xxx/xxx-
#disable-shared enable-static 不写也可以，默认就是这样的。
#--sysroot: 
#--extra-cflags: 会传给gcc的参数
#--arch --target-os : 必须要给
./configure \
--prefix=$PREFIX \
--enable-small \
--enable-gpl \
--enable-nonfree \
--disable-programs \
--disable-ffmpeg \
--disable-ffplay \
--disable-ffprobe \
--disable-doc \
--disable-symver \
--disable-asm \
--enable-librtmp \
--enable-cross-compile \
--cross-prefix=$TOOLCHAIN/bin/aarch64-linux-android- \
--enable-shared \
--disable-static \
--sysroot=$NDK_ROOT/platforms/android-21/arch-arm64/ \
--extra-cflags="-isysroot $NDK_ROOT/sysroot -isystem $NDK_ROOT/sysroot/usr/include/aarch64-linux-android -D__ANDROID_API__=21 -march=armv8-a -g -DANDROID -ffunction-sections -funwind-tables -fstack-protector-strong -no-canonical-prefixes -Wa,--noexecstack -Wformat -Werror=format-security -O0 -fPIC -I $RTMP/include" \
--extra-ldflags="-L $RTMP/lib" \
--extra-libs="-l rtmp" \
--arch=aarch64 \
--target-os=android

#上面运行脚本生成makefile之后，使用make执行脚本

make clean
make -j16
make install

```
#### 5 FFmpeg 参数相关知识
##### 5.1 平台相关
```java
//armv7-a 平台
--arch=arm
--cpu=armv7-a
//armv8-a平台
--arch=aarch64
--cpu=armv8-a
//x86平台
--arch=x86
--cpu=i686
//x86_64平台
--arch=x86_64
--cpu=x86_64
```
##### 5.2 --extra-cflags 来源参考
AndroidStudio工程目录: **NDKProject/FFmpegLearn/app/.cxx/cmake/debug/arm64-v8a/build.ninja**, 具体使用的AndroidStudio版本有关
工程支持 **Native C++**
