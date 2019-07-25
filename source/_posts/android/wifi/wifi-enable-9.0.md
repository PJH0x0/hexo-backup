---
title: Android P Wifi打开流程变化
date: 2019/07/25
tags:
  - android
  - wifi
categories:
  - android
  - wifi
abbrlink: 141bf3b4
---

本文主要的目的是理清Android O和Android P之间Wifi enable流程的变化, 这部分主要是在WifiStateMachine状态机的状态变化方面, 增加了WifiStateMachinePrime类, 减少了WifiStateMachine部分的状态. 接下来看一下具体的流程变化
这里约定一些缩写:
1. WSM: WifiStateMachine
2. WSMP: WifistateMachinePrime
3. CMSM: ClientModeStateMachine
# UML图
由于UML图比较复杂,这里只是使用了简化的UML图
## 时序图
![sequence.png](sequence.png)
## WSM状态图
![statemachine-structor.png](wifi-statemachine-structor.png)
# 涉及的源码
```java
frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiController.java
frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiStateMachinePrime.java
frameworks/opt/net/wifi/service/java/com/android/server/wifi/ClientModeManager.java
```

# WifiController
这里不再是向WSM发送消息了, 而是通过WSMP来进行操作, 这里分为WLAN扫描开/关两种情况
1. WLAN扫描关闭的情况下, 进入StaDisabledState 并且关闭Wifi, 这个时候同时会卸载Wifi驱动
```java
@Override
public void enter() {
    //disable wifi
    mWifiStateMachinePrime.disableWifi();
    // Supplicant can't restart right away, so note the time we switched off
    mDisabledTimestamp = SystemClock.elapsedRealtime();
    mDeferredEnableSerialNumber++;
    mHaveDeferredEnable = false;
    mWifiStateMachine.clearANQPCache();
}
```
2. WLAN扫描打开的情况下
```java
@Override
public void enter() {
    // now trigger the actual mode switch in WifiStateMachinePrime
    mWifiStateMachinePrime.enterScanOnlyMode();

    // TODO b/71559473: remove the defered enable after mode management changes are complete
    // Supplicant can't restart right away, so not the time we switched off
    mDisabledTimestamp = SystemClock.elapsedRealtime();
    mDeferredEnableSerialNumber++;
    mHaveDeferredEnable = false;
}
```

接下来都是进入DeviceActiveState
```java
@Override
public void enter() {
    mWifiStateMachinePrime.enterClientMode();
    mWifiStateMachine.setHighPerfModeEnabled(false);
}
```

# WifiSatateMachinePrime(WSMP)
WSMP里面关于Wifi打开与关闭的主要是就是ModeStateMachine的处理, 里面有三个状态WifiDisabledState, ScanOnlyModeActiveState, ClientModeActiveState, 对应的就是上面WifiController发送的三个消息.
## 消息发送
```java
public void disableWifi() {
    changeMode(ModeStateMachine.CMD_DISABLE_WIFI);
}
public void enterScanOnlyMode() {
    changeMode(ModeStateMachine.CMD_START_SCAN_ONLY_MODE);
}
public void enterClientMode() {
    changeMode(ModeStateMachine.CMD_START_CLIENT_MODE);
}
private void changeMode(int newMode) {
    mModeStateMachine.sendMessage(newMode);
}
```
可以看到WifiController最终的处理都是发送对应的消息给ModeStateMahcine进行处理
## ModeStateMachine初始化
```java
public static final int CMD_START_CLIENT_MODE    = 0;
public static final int CMD_START_SCAN_ONLY_MODE = 1;
public static final int CMD_DISABLE_WIFI         = 3;

ModeStateMachine() {
    super(TAG, mLooper);

    addState(mClientModeActiveState);
    addState(mScanOnlyModeActiveState);
    addState(mWifiDisabledState);

    Log.d(TAG, "Starting Wifi in WifiDisabledState");
    setInitialState(mWifiDisabledState);
    start();
}
```
从初始化情况来看, 默认就是WifiDisabledState

### WifiDiabledState初始化
```java
@Override
public void enter() {
    Log.d(TAG, "Entering WifiDisabledState");
    mDefaultModeManager.sendScanAvailableBroadcast(mContext, false);
    mScanRequestProxy.enableScanningForHiddenNetworks(false);
    mScanRequestProxy.clearScanResults();
}
```
可以看到发送了ScanAvailable以及关闭了Scan

### ScanOnlyModeActiveState初始化
```java
@Override
public void enter() {
    Log.d(TAG, "Entering ScanOnlyModeActiveState");

    mListener = new ScanOnlyListener();
    mManager = mWifiInjector.makeScanOnlyModeManager(mListener);
    mManager.start();
    mActiveModeManagers.add(mManager);
    
    //电量相关, 暂时不管它
    updateBatteryStatsWifiState(true);
    updateBatteryStatsScanModeActive();
}
```
就做了两件事, 创建了一个监听器以及一个新的对象

### ClientModeActiveState初始化
```java
@Override
public void enter() {
    Log.d(TAG, "Entering ClientModeActiveState");

    mListener = new ClientListener();
    mManager = mWifiInjector.makeClientModeManager(mListener);
    mManager.start();
    mActiveModeManagers.add(mManager);

    updateBatteryStatsWifiState(true);
}
```
跟ScanOnlyModeActiveState差不多, 都是创建了一个监听器然后创建了一个Manager对象

## 处理消息
ModeStateMachine都是要走`checkForAndHandleModeChange()`的
```java
private boolean checkForAndHandleModeChange(Message message) {
    switch(message.what) {
        case ModeStateMachine.CMD_START_CLIENT_MODE:
            Log.d(TAG, "Switching from " + getCurrentMode() + " to ClientMode");
            mModeStateMachine.transitionTo(mClientModeActiveState);
            break;
        case ModeStateMachine.CMD_START_SCAN_ONLY_MODE:
            Log.d(TAG, "Switching from " + getCurrentMode() + " to ScanOnlyMode");
            mModeStateMachine.transitionTo(mScanOnlyModeActiveState);
            break;
        case ModeStateMachine.CMD_DISABLE_WIFI:
            Log.d(TAG, "Switching from " + getCurrentMode() + " to WifiDisabled");
            mModeStateMachine.transitionTo(mWifiDisabledState);
            break;
        default:
            return NOT_HANDLED;
    }
    return HANDLED;
}
```
就是MODE状态的相互切换, 这里我们关注`CMD_START_CLIENT_MODE`就行了, 因为这条才是enable wifi的消息, `CMD_START_SCAN_ONLY_MODE`消息只是允许硬件在底层扫描AP.

接收到`CMD_START_CLIENT_MODE`消息之后就会进入 ClientModeActiveState初始化, 接下来我们可以看一下这个ClientModeManager的操作

# ClientModeManager
ClientModeManager主要是ClientModeStateMachine(CMSM)这个状态机, 首先是入口`start()`方法
```java
public void start() {
    mStateMachine.sendMessage(ClientModeStateMachine.CMD_START);
}
```
## CMSM初始化
CMSM只有两个状态, IdleState和StartedState.
```java
ClientModeStateMachine(Looper looper) {
    super(TAG, looper);

    addState(mIdleState);
    addState(mStartedState);

    setInitialState(mIdleState);
    start();
}
```
## CMSM处理CMD_START消息
```java
@Override
public boolean processMessage(Message message) {
    switch (message.what) {
        case CMD_START:
            updateWifiState(WifiManager.WIFI_STATE_ENABLING,
                            WifiManager.WIFI_STATE_DISABLED);

            //1. 设置Wifi接口设置位客户端模式
            mClientInterfaceName = mWifiNative.setupInterfaceForClientMode(
                    false /* not low priority */, mWifiNativeInterfaceCallback);
            if (TextUtils.isEmpty(mClientInterfaceName)) {
                Log.e(TAG, "Failed to create ClientInterface. Sit in Idle");
                updateWifiState(WifiManager.WIFI_STATE_UNKNOWN,
                                WifiManager.WIFI_STATE_ENABLING);
                updateWifiState(WifiManager.WIFI_STATE_DISABLED,
                                WifiManager.WIFI_STATE_UNKNOWN);
                break;
            }
            sendScanAvailableBroadcast(false);
            mScanRequestProxy.enableScanningForHiddenNetworks(false);
            mScanRequestProxy.clearScanResults();
            //2. 切换到StartedState中
            transitionTo(mStartedState);
            break;
        default:
            Log.d(TAG, "received an invalid message: " + message);
            return NOT_HANDLED;
    }
    return HANDLED;
}
```
上面的代码的核心就是调用`setupInterfaceForClientMode()`来与硬件进行交互. 然后切换到StartedState当中

## CMSM后续处理
```java
@Override
public void enter() {
    Log.d(TAG, "entering StartedState");
    mIfaceIsUp = false;
    onUpChanged(mWifiNative.isInterfaceUp(mClientInterfaceName));
    mScanRequestProxy.enableScanningForHiddenNetworks(true);
}
private void onUpChanged(boolean isUp) {
    if (isUp == mIfaceIsUp) {
        return;  // no change
    }
    mIfaceIsUp = isUp;
    if (isUp) {
        Log.d(TAG, "Wifi is ready to use for client mode");
        sendScanAvailableBroadcast(true);
        
        mWifiStateMachine.setOperationalMode(WifiStateMachine.CONNECT_MODE,
                                             mClientInterfaceName);
        updateWifiState(WifiManager.WIFI_STATE_ENABLED,
                        WifiManager.WIFI_STATE_ENABLING);
    } else {
        if (mWifiStateMachine.isConnectedMacRandomizationEnabled()) {
            // Handle the error case where our underlying interface went down if we
            // do not have mac randomization enabled (b/72459123).
            return;
        }
        // if the interface goes down we should exit and go back to idle state.
        Log.d(TAG, "interface down!");
        updateWifiState(WifiManager.WIFI_STATE_UNKNOWN,
                        WifiManager.WIFI_STATE_ENABLED);
        mStateMachine.sendMessage(CMD_INTERFACE_DOWN);
    }
}
```
这段代码就是检测一下是不是打开了wifi, 并且发送一下广播以及切换WSM的状态 

## 小结
使用WSMP的好处就是减少了WSM的工作, 以及让本来WSM相当庞大的状态减少了许多, 算是架构的优化吧


