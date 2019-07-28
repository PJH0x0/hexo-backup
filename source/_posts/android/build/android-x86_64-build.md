---
title: Android编译x86_64选项
date: 2019/7/28
categories:
  - android
  - build
tags:
  - android
  - build
abbrlink: 64d4863f
---

# 原因
选择x86_64的原因就是开启虚拟机的速度很快，当我们需要调试Android系统，比如打印了一些LOG或者是改了一些文件，使用x86_64选项编译的虚拟机要比arm64的虚拟机快了很多，具体数据没有测试过，不同电脑的性能对于虚拟机启动时间会有影响, 如果没有固态硬盘的情况下，大致是一半的时间. 而且因为和宿主机的cpu架构相同，用的时候也不会很卡，我之前用arm64的时候有时甚至会卡直接重启虚拟机

# 编译
直接在`lunch`的时候选择`aosp_x86_64-eng`就可以了

![lunch](lunch-x86_64.png)

# 开启CPU虚拟化
进入BIOS开启CPU虚拟化，intel的是叫Intel Virtual Technology, AMD是叫SVM mode, 开启之后重启， 用`egrep -c '(svm|vmx)' /proc/cpuinfo`进行检测， 如果不为0就是正确开启了
![svm-check](svm-check.png)

# 安装KVM
`sudo apt-get install qemu qemu-kvm`, 安装之后用`kvm-ok`检测是否安装成功.

![kvm-ok](kvm-ok.png)

# 问题
有个问题出现了
```shell
emulator: ERROR: x86 emulation currently requires hardware acceleration!
Please ensure KVM is properly installed and usable.
CPU acceleration status: This user doesn't have permissions to use KVM (/dev/kvm)
```
这个问题主要是因为我们当前用户没有权限导致的，运行以下的命令进行解决
```shell
sudo addgroup kvm
sudo usermod -a -G kvm username
sudo chgrp kvm /dev/kvm
sudo vim /etc/udev/rules.d/60-qemu-kvm.rules #在里面写入KERNEL=="kvm", GROUP="kvm", MODE="0660"
```
[原文](https://stackoverflow.com/questions/30076015/running-android-emulator-during-jenkins-build)中有详细的解释，非常清楚

