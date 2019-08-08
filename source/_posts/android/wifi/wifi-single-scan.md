---
title: Android P Wifi扫描流程解析
date: 2019/8/8
categories:
  - android
  - wifi
tags:
  - android
  - source
abbrlink: 8e7e943f
---

Wifi扫描需要分为三个阶段进行解析:
1. 第一阶段是扫描阶段, 这个阶段会发送扫描指令给驱动进行扫描
2. 第二阶段是获取扫描结果, 这个阶段也是发送获取扫描结果指令给驱动获取扫描结果
3. 预阶段, 这个阶段主要是用于注册回调用于通知何时获取扫描结果的

# 时序图
这个是时序图是Wifi打开完成之后触发的扫描流程
![Wifi扫描流程](sequence.png)

# 扫描阶段
这个阶段触发的方式很多, 详情见Android P Wifi扫描场景, 但最终都是调用`WifiScanner`的`startScan()`方法
```java
public void startScan(ScanSettings settings, ScanListener listener, WorkSource workSource) {
    Preconditions.checkNotNull(listener, "listener cannot be null");
    int key = addListener(listener);
    if (key == INVALID_KEY) return;
    validateChannel();
    Bundle scanParams = new Bundle();
    scanParams.putParcelable(SCAN_PARAMS_SCAN_SETTINGS_KEY, settings);
    scanParams.putParcelable(SCAN_PARAMS_WORK_SOURCE_KEY, workSource);
    mAsyncChannel.sendMessage(CMD_START_SINGLE_SCAN, 0, key, scanParams);
}
```
这里使用了`AsyncChannel`进行传递消息, `AsyncChannel`的原理就是利用了Android`Messenger`来在进程间通信, `Messenger`的介绍可以看一下[Android 基于Message的进程间通信 Messenger完全解析](https://blog.csdn.net/lmj623565791/article/details/47017485)

