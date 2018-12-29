---
title: Android 9.0内核编译
categories:
  - android
  - build
tags:
  - android
  - kernel
  - build
abbrlink: 7a725fc8
---
> 前言:本文是在[Android 9.0编译](http://www.godteen.com/posts/73919dd2/)基础上操作的, 请保证Android编译成功之后来阅读本文.最近抱着学习的心态想要看一下驱动方面相关的知识,所以把内核下载编译记录一下.阅读本文需要的知识:Shell, Git基本操作

# 下载内核
```Shell
cd kernel
git clone https://aosp.tuna.tsinghua.edu.cn/android/kernel/goldfish.git
```
执行完这两条命令后就可以看到kernel目录下有一个goldfish目录了,goldfish内核专门是提供给emulator用的.
```Shell
cd goldfish
git branch -a
git checkout remotes/origin/android-goldfish-3.18
```
这里涉及到要下载哪个版本的内核,`emulator`命令默认用的是qemu内核, 一般跟这个版本的内核一样就可以了,可以直接启用`emulator`然后进入到`Settings->System->About phone->点击Android version`,里面就有内核版本号.
# 编译内核
编译内核之前同样要执行`source`和`lunch`操作,一定要和Android编译时选的一样, 可以进入到`out/target/product`目录下查看自己编的是什么版本, 如果是`generic_x86_64`,那lunch的就是`aosp_x86_64_eng`的,其他版本依此类推.
接下来就可以编译了,下面我提供了几个编译的脚本大家可以进行对比和参照,`lunch`不同的版本对应的脚本是不一样的,即使编译通过了,也不能运行,我因为这个浪费了很长的时间
使用方法:
1. 在goldfish目录下创建一个build.sh文件
2. 将脚本里面的内容复制到build.sh中,或者根据脚本自己写, 注意luch的版本
3. 执行`chmod a+rx build.sh`并且执行`./build.sh`.
## aosp_x86_64-eng
```Shell
#指定编译的内核类型
export ARCH=x86_64
#指定的gcc编译器的前缀, 就是下面PATH中的x86_64-linux-android-4.9的前缀
export CROSS_COMPILE=x86_64-linux-android-
export REAL_CROSS_COMPILE=x86_64-linux-android-
#这里android_root要写是android根目录的绝对地址例如: ~/google/android-9.0
PATH=$PATH:android_root/prebuilts/gcc/linux-x86/x86/x86_64-linux-android-4.9/bin
#编译的配置,在arch/x86/configs目录下,
make x86_64_ranchu_defconfig
#编译内核命令
make
```
编译内核大概10来分钟就可以完成了,之后会生成一个`arch/x86/boot/bzImage`的东西,不同kernel生成的是不一样的,要看清楚.
最后回到android根目录执行`emulator -kernel kernel/goldfish/arch/x86/boot/bzImage`启动虚拟机

