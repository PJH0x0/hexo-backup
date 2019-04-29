---
title: Android 9.0编译
date: 2019/01/02
categories:
  - android
  - build
tags:
  - android
  - build
abbrlink: 73919dd2
---
> 前言: 本文是针对国内无法科学上网或者即使可以网速也很慢的情况下针对Google 2018年发布的Android P编译的一篇文章.如果能够科学上网的同学尽量参照[官方教程](https://source.android.com/setup/build/downloading),里面也是有中文页面的.阅读本文需要的知识:Shell命令有一定了解, Git基本操作, 对Android有一定的了解

# 准备工作
## 操作系统
Ubuntu 18.04或者是Ubuntu 16.04, android编译依赖极多,如果不是对于自己使用的操作系统特别熟悉的话还是按照官方指示来吧.
## JDK
必须使用`OpenJDK8`以上,我使用`OpenJDK9`编译也通过了, 使用`java -version`进行查看当前的java版本, 如果没有安装请使用以下命令安装:
```Shell
sudo apt-get update
sudo apt install openjdk-8-jdk
```
## 依赖安装
```Shell
#下面是参照的https://www.jianshu.com/p/716088b50332
sudo apt-get install -y git flex bison gperf build-essential libncurses5-dev:i386
sudo apt-get install libx11-dev:i386 libreadline6-dev:i386 libgl1-mesa-dev g++-multilib
sudo apt-get install tofrodos python-markdown libxml2-utils xsltproc zlib1g-dev:i386
sudo apt-get install dpkg-dev libsdl1.2-dev libesd0-dev
sudo apt-get install git-core gnupg flex bison gperf build-essential
sudo apt-get install zip curl zlib1g-dev gcc-multilib g++-multilib
sudo apt-get install libc6-dev-i386
sudo apt-get install lib32ncurses5-dev x11proto-core-dev libx11-dev
sudo apt-get install lib32z-dev ccache
sudo apt-get install libgl1-mesa-dev libxml2-utils xsltproc unzip m4
#下面两个是我自己加的
sudo apt-get install libswitch-perl
sudo apt-get install git-core gawk
```
上面的命令依次执行,可能会有不支持i386和32位的情况跳过即可, 如果已安装最新的版本也不用去管.
# 源码下载
## 安装repo
repo是一个git工具,专门给android使用的.
```Shell
mkdir ~/bin
cd ~/bin/
#这个环境变量我就不知道加的什么意思了
PATH=~/bin:$PATH
#只会在当前目录下生成repo文件
curl https://mirrors.tuna.tsinghua.edu.cn/git/git-repo -o repo
chmod a+x repo
```
然后一定要在`.bashrc`中添加
```Shell
export REPO_URL='https://mirrors.tuna.tsinghua.edu.cn/git/git-repo/'
```
并且执行`source .bashrc`
## 下载AndroidP源码
```Shell
cd ~
mkdir -p google/android-9.0
repo init -u https://aosp.tuna.tsinghua.edu.cn/platform/manifest -b android-9.0.0_r9
repo sync
```
我没有使用下载初始化包的方式,感觉没有必要,因为现在的网速还是蛮快的,没有必要多此一举.
# 源码编译
源码编译的步骤极为简单
```Shell
source build/envsetup.sh
lunch
```
执行完`lunch`命令后会有很多个版本供选择,选择其中一个即可,输入对应版本前面的编号或者是版本名称按回车.一般有三种选择:
1. aosp_arm-eng, 编译arm架构的32位的engineer版本.
2. aosp_arm64-eng, 编译arm架构下64位的engineer版本.
3. aosp_x86_64-eng, 编译x86架构下64位的engineer版本(推荐编译这个版本, 速度真的快很多, 不过要机器支持虚拟化,而且如果没有显卡的话还得加上-gpu off参数)

选定版本后执行`make -j4`, `-j4`取决于你的机器,取的大一点也无所谓. 如果编译结束后出现了`make compelete successfully`就表明编译成功了.  
# 启动虚拟机
使用`emulator`启动android虚拟机, `emulator -help`可以查看对应的参数.这里要注意两个问题:
1. 使用`emulator`一定要先执行`source和lunch`命令.
2. 如果使用aosp_x86_64-eng要启动虚拟机要开启硬件加速,具体如何启动需要自行搜索一下.

# 总结
如果大家编译报错,多上StackOverflow进行找找问题解决方案, 实在找不到解决方案就重新下一份其他分支上面的代码重新编(反正我是这么干的),祝大家编译顺利!!!

参考:
1. https://www.jianshu.com/p/716088b50332
2. https://mirror.tuna.tsinghua.edu.cn/help/AOSP/
