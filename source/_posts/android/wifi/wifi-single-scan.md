---
title: Android P Wifi扫描流程解析
date: 2019/8/8
categories:
  - android
  - wifi
tags:
  - android
  - source
---

Wifi扫描需要分为三个阶段进行解析:
1. 第一阶段是扫描阶段, 这个阶段会发送扫描指令给驱动进行扫描
2. 第二阶段是获取扫描结果, 这个阶段也是发送获取扫描结果指令给驱动获取扫描结果
3. 预阶段, 这个阶段主要是用于注册回调用于通知何时获取扫描结果的

# 时序图
![Wifi扫描流程](sequence.png)

# 扫描阶段
这个阶段触发的方式很多, 详情见, 先从`WifiManager`的`startScan()`开始

