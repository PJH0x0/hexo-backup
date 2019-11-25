---
title: Flutter开发环境搭建(无AndroidStudio)
date: 2019/11/25
categories:
  - flutter
tags:
  - flutter
---
>本文介绍的是在没有下载AndroidStudio的情况下如何搭建flutter开发环境, 系统基于Ubuntu 18.04, Windows应该也是差不多, 主要是我想要用vim开发flutter, 习惯了vim

# 下载flutter
首先是要下载flutter, 官网最新的[stable版本](https://flutter.dev/docs/development/tools/sdk/releases?tab=linux)截止到目前是v1.9.1+hotfix.6, 什么版本无所谓, 不会影响环境搭建. 
但是有一点要特别注意:**解压之后不能将目录设置为.flutter, 否则会报错, 这个bug至今未解**, 然后就是要设置一下环境变量
```shell
#~/.bashrc文件
#~/.flutter-sdk是我的目录, 需要按照自己解压的路径重新设置
export PATH=~/.flutter-sdk/bin:$PATH
```
最后运行一下`flutter doctor`, 出现类似`[✓] Flutter (Channel stable, v1.9.1+hotfix.6, on Linux, locale en_US.UTF-8)`这种信息则说明成功了, 至于下面的一些有关于vscode,AndroidStudio这些工具的报错信息暂时不管
# 下载SDK tools
接下来, 我们要到[SDK官网](https://developer.android.com/studio)中下载SDK tools, 这里下载*Command line tools only*版本, 然后解压到对应的目录下, 里面只有一个tools目录,这时我们需要设置一下环境变量, 方便以后使用
```shell
#~/.bashrc文件
export ANDROID_HOME=~/.sdk
export PATH=$PATH:${ANDROID_HOME}/platform-tools
export PATH=$PATH:${ANDROID_HOME}/tools
#shell中执行
source ~/.bashrc
```
然后运行一下`sdkmanager --list`可以看到sdk的所有相关的工具, flutter需要的工具有
```shell
sdkmanager "platform-tools"
sdkmanager "platforms;android-29"
sdkmanager "build-tools;29.0.2"
```
之后运行`flutter doctor`会提示我们要安装licenses, 按照`flutter doctor --android-licenses`命令安装即可, 安装完成之后则可以看到sdk的目录下会多一个licenses目录, 之后运行`flutter doctor`则可以看到类似*[✓] Android toolchain - develop for Android devices (Android SDK version 29.0.2)*的信息, 以上就是配置的全过程

# 设置Android虚拟机
为了运行demo,我们需要下载一个Android虚拟机镜像, `sdkmanager "system-images;android-29;google_apis_playstore;x86_64"`, 接下来的步骤可能会有点麻烦, 所以我更推荐的是直接用真机, 这样更简单明了
## 打开CPU虚拟化
如果电脑使用的是Ubuntu真机, 可以在BIOS中找到,不同型号的主板在不一样的位置,英文大致都是含有KVM, 如果是Ubuntu虚拟机的话可以在虚拟机-设置-处理器-虚拟化引擎中找到
## 设置权限
下面这些命令来自Stack Overflow
```shell
sudo apt install qemu-kvm
sudo adduser <username> kvm
sudo chown <username> /dev/kvm
```
按照上面运行一遍, 否则就会出现类似*CPU acceleration status: This user doesn't have permissions to use KVM (/dev/kvm)*这种错误
## 创建并运行Android虚拟机
```shell
flutter emulators --create --name device_name
flutter emulators
flutter emulators --launch device_name
```
然后静静的等待开机就好了,虚拟机开机时间是相当长, 当然创建虚拟机的方式不是只有这一种, 也可以使用avdmanager进行创建
# 运行demo
在运行demo之前, 首先要确定设备已经连上了, 使用`flutter devices`确定设备已连接, 之后进入的flutter目录下的examples目录, 里面有创建好的应用, 进入对应的目录执行`flutter run`即可
例如:
```shell
cd ~/.flutter-sdk/examples/hello_world
flutter run
```
之后就会出现一个黑色的页面,并且中间有一个*Hello,world!*
