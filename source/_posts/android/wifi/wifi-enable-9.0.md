---
title: Android 9.0 Wifi打开流程解析-Java
date: 2019/03/25
tags:
  - android
  - wifi
categories:
  - android
  - wifi
---
>前言:本文是针对与Android 9.0 wifi enable的流程解析,由于Android 8.0和Android 9.0的frameworks的差异比较大,所以重新写一遍流程
# UML图
由于UML图比较复杂,这里只是使用了简化的UML图
## 时序图
![sequence.png](sequence.png)
## WifiStateMachine状态图
![statemachine-structor.png](wifi-statemachine-structor.png)

