---
title: Wifi打开流程解析-Java
date: 2018/12/13
tags:
  - android
  - wifi
categories:
  - android
  - wifi
abbrlink: 60d088a2
---
> 前言: 本文主要是针对于frameworks层的流程解析,包括了一小部分的Settings的流程. 阅读本文需要的知识: frameworks的基本架构了解, [状态机知识](http://www.godteen.com/posts/2262efea/), Java基础

# 操作系统
Andorid 8.1

# 相关源码
```Java
packages/apps/Settings/src/com/android/settings/wifi/WifiEnabler.java

```
# 
