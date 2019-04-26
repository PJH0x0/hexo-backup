---
title: Wifi打开流程解析-Java
date: 2018/12/18
tags:
  - android
  - wifi
categories:
  - android
  - wifi
abbrlink: 60d088a2
---
> 前言: 本文主要是针对于frameworks层的流程解析,包括了一小部分的Settings的流程. 阅读本文需要的知识: frameworks的基本架构了解, [状态机](http://www.godteen.com/posts/2262efea/), Java基础, Handler基础, 线程基础

# 操作系统
Andorid 8.1

# 相关源码
```Java
packages/apps/Settings/src/com/android/settings/wifi/WifiEnabler.java
frameworks/base/wifi/java/android/net/wifi/WifiManager.java
frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiServiceImpl.java
frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiSettingsStore.java
frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiController.java
frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiStateMachine.java
```
# 时序图
这里没有给出完整的时序图,因为后面状态机切换流程比较复杂,不太好用时序图进行表示
![wifi-enable](sequence.png)

# 用户进程处理
用户进程是WifiEnabler和WifiManager的调用
```Java
//WifiEnabler.java
@Override
public void onSwitchChanged(Switch switchView, boolean isChecked) {
    // Show toast message if Wi-Fi is not allowed in airplane mode
    //1. 判断是不是开启了飞行模式
    if (isChecked && !WirelessUtils.isRadioAllowed(mContext, Settings.Global.RADIO_WIFI)) {
        Toast.makeText(mContext, R.string.wifi_in_airplane_mode, Toast.LENGTH_SHORT).show();
        // Reset switch to off. No infinite check/listenenr loop.
        mSwitchBar.setChecked(false);
        return;
    }

    // Disable tethering if enabling Wifi
    //2. 关闭手机作为Wifi热点
    if (mayDisableTethering(isChecked)) {
        mWifiManager.setWifiApEnabled(null, false);
    }

    /******省去无关代码*****/

    //3. 调用WifiManager的方法
    if (!mWifiManager.setWifiEnabled(isChecked)) {
        // Error
        mSwitchBar.setEnabled(true);
        Toast.makeText(mContext, R.string.wifi_error, Toast.LENGTH_SHORT).show();
    }
}
//WifiManager.java
public boolean setWifiEnabled(boolean enabled) {
    try {
        //4. 通过Binder调用WifiServiceImpl的方法
        return mService.setWifiEnabled(mContext.getOpPackageName(), enabled);
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```
上面的逻辑比较简单,就两步
1. 调用`WifiManager.setWifiEnabled()`方法, 本文讨论的是`isChecked=true`的情况
2. 通过Binder调用`mService.setWifiEnabled()`方法

# WifiServiceImpl的处理
```Java
//WifiServiceImpl.java
@Override
public synchronized boolean setWifiEnabled(String packageName, boolean enable)
    throws RemoteException {
    /******省去无关代码*****/

    long ident = Binder.clearCallingIdentity();
    try {
        //1. 将打开WiFi写入Settings.Global数据库中
        if (! mSettingsStore.handleWifiToggled(enable)) {
            // Nothing to do if wifi cannot be toggled
            return true;
        }
    } finally {
        Binder.restoreCallingIdentity(ident);
    }
    /******省去无关代码*****/
    //2. 给WifiController发送WIFI_TOGGLED的消息
    mWifiController.sendMessage(CMD_WIFI_TOGGLED);
}
//WifiSettingsStore.java
public synchronized boolean handleWifiToggled(boolean wifiEnabled) {
    // Can Wi-Fi be toggled in airplane mode ?
    if (mAirplaneModeOn && !isAirplaneToggleable()) {
        return false;
    }

    //3. 本篇讨论的是打开wifi的情况,所以wifiEnabled为true
    if (wifiEnabled) {
        if (mAirplaneModeOn) {
            //强制打开WIFI
            persistWifiState(WIFI_ENABLED_AIRPLANE_OVERRIDE);
        } else {
            persistWifiState(WIFI_ENABLED);
        }
    } else {
        // When wifi state is disabled, we do not care
        // if airplane mode is on or not. The scenario of
        // wifi being disabled due to airplane mode being turned on
        // is handled handleAirplaneModeToggled()
        persistWifiState(WIFI_DISABLED);
    }
    return true;
}
private void persistWifiState(int state) {
    final ContentResolver cr = mContext.getContentResolver();
    mPersistWifiState = state;
    Settings.Global.putInt(cr, Settings.Global.WIFI_ON, state);
}

```
总结起来,`WifiServiceImpl`也是做了两件事:
1. 向`Settings.Global`数据库设置`WIFI_ON`为`WIFI_ENABLED`
2. 向`WifiController`发送`CMD_WIFI_TOGGLED`信息

# WifiController的处理
`WifiController`是一个状态机(StateMachine), 我们研究状态机时要做的第一件事就是确定状态机当前的状态.
一种干扰最少的方式就是开机自动开启Wifi,这种情况在[WifiService启动流程](http://www.godteen.com/posts/d1635d94/)中提到过,这里我们只要知道开机也会开启Wifi就够了.
## WifiController初始化
`WifiController`在`WifiService`创建`WifiServiceImpl`就完成了,在`WifiInjector`的构造方法中完成,暂时不去讨论.
```Java
//WifiController.java
WifiController(Context context, WifiStateMachine wsm, WifiSettingsStore wss,
        WifiLockManager wifiLockManager, Looper looper, FrameworkFacade f) {
    //创建StateMachine的另一个方式
    super(TAG, looper);
    /******省去无关代码*****/

    //1. 添加状态机
    addState(mDefaultState);
        addState(mApStaDisabledState, mDefaultState);
        addState(mStaEnabledState, mDefaultState);
            addState(mDeviceActiveState, mStaEnabledState);
            addState(mDeviceInactiveState, mStaEnabledState);
                addState(mScanOnlyLockHeldState, mDeviceInactiveState);
                addState(mFullLockHeldState, mDeviceInactiveState);
                addState(mFullHighPerfLockHeldState, mDeviceInactiveState);
                addState(mNoLockHeldState, mDeviceInactiveState);
        addState(mStaDisabledWithScanState, mDefaultState);
        addState(mApEnabledState, mDefaultState);
        addState(mEcmState, mDefaultState);

    boolean isAirplaneModeOn = mSettingsStore.isAirplaneModeOn();
    boolean isWifiEnabled = mSettingsStore.isWifiToggleEnabled();
    //2. 获取是否打开了WLAN扫描功能
    boolean isScanningAlwaysAvailable = mSettingsStore.isScanAlwaysAvailable();

    log("isAirplaneModeOn = " + isAirplaneModeOn +
            ", isWifiEnabled = " + isWifiEnabled +
            ", isScanningAvailable = " + isScanningAlwaysAvailable);
    //3. 设置对应的初始状态
    if (isScanningAlwaysAvailable) {
        setInitialState(mStaDisabledWithScanState);
    } else {
        setInitialState(mApStaDisabledState);
    }
    /******省去无关代码*****/

}
```
第一步就是给状态机添加状态,这里给出WifiController的状态结构
![wifi-controller-state-structor](wifi-controller-statemachine-structor.png)

第三步设置初始状态则是根据是否开启**WLAN扫描**功能来进行设置.**WLAN扫描**功能可以直接在设置首页搜索"扫描"就可以进入查看.或者是通过`adb shell settings get global wifi_scan_always_enabled`查看,默认一般是打开的.
时序图中的情况是**WLAN扫描**关闭的情况,下面我们分别从这两种情况出发来进行分析

## WLAN扫描关闭的情况
首先是WLAN扫描关闭情况下,`CMD_WIFI_TOGGLED`是发送给了`ApStaDisabledState`
```Java
//WifiController$ApStaDisabledState
class ApStaDisabledState extends State {
    private int mDeferredEnableSerialNumber = 0;
    private boolean mHaveDeferredEnable = false;
    private long mDisabledTimestamp;

    @Override
    public void enter() {
        //关闭Supplicant,本身就是关闭的,不用管
        mWifiStateMachine.setSupplicantRunning(false);
        // Supplicant can't restart right away, so not the time we switched off
        mDisabledTimestamp = SystemClock.elapsedRealtime();
        mDeferredEnableSerialNumber++;
        mHaveDeferredEnable = false;
        mWifiStateMachine.clearANQPCache();
    }
    @Override
    public boolean processMessage(Message msg) {
        switch (msg.what) {
            case CMD_WIFI_TOGGLED:
            case CMD_AIRPLANE_TOGGLED:
                //1. 判断是否打开WiFi,就是在WifiSettingsStore中设置过的
                if (mSettingsStore.isWifiToggleEnabled()) {

                    /****省略其他无关代码****/
                    //2. 默认就是false
                    if (mDeviceIdle == false) {
                        // wifi is toggled, we need to explicitly tell WifiStateMachine that we
                        // are headed to connect mode before going to the DeviceActiveState
                        // since that will start supplicant and WifiStateMachine may not know
                        // what state to head to (it might go to scan mode).
                        //3. CONNECT_MODE的值是1, 设置模式
                        mWifiStateMachine.setOperationalMode(WifiStateMachine.CONNECT_MODE);
                        //4. 切换到DeviceActivate状态中
                        transitionTo(mDeviceActiveState);
                    } else {
                        checkLocksAndTransitionWhenDeviceIdle();
                    }
                } else if (mSettingsStore.isScanAlwaysAvailable()) {
                    transitionTo(mStaDisabledWithScanState);
                }
                break;
                /****省略其他无关代码****/
        }
    }
}
//WifiStateMachine.java
public void setOperationalMode(int mode) {
    if (mVerboseLoggingEnabled) log("setting operational mode to " + String.valueOf(mode));
    //这是一个异步消息
    sendMessage(CMD_SET_OPERATIONAL_MODE, mode, 0);
}

```
第3步中`setOperationalMode(WifiStateMachine.CONNECT_MODE)`就是给`WifiStateMachine`发了一个异步消息.
接下来从状态机的架构中可以得知从`ApStaDisabledState`切换到`DeviceActiveState`要经过`StaEnabledState`,这两个状态都会调用`enter()`方法,并且是`StaEnabledState`先调用.
```Java
//WifiController$StaEnabledState
class StaEnabledState extends State {
    @Override
    public void enter() {
        //1. 打开Supplicant
        mWifiStateMachine.setSupplicantRunning(true);
    }
}
class DeviceActiveState extends State {
    @Override
    public void enter() {
        //2. 再设置一次
        mWifiStateMachine.setOperationalMode(WifiStateMachine.CONNECT_MODE);
        //3. 这里我也是不清楚到底是干嘛的
        mWifiStateMachine.setHighPerfModeEnabled(false);
    }
}
//WifiStateMachine.java
public void setSupplicantRunning(boolean enable) {
    if (enable) {
        sendMessage(CMD_START_SUPPLICANT);
    } else {
        sendMessage(CMD_STOP_SUPPLICANT);
    }
}
```
需要注意的是, `CMD_START_SUPPLICANT`和`CMD_SET_OPERATIONAL_MODE`都是异步消息,但是因为`WifiStateMachine`是要先处理完`ApStaDisabledState`发送的`CMD_SET_OPERATIONAL_MODE`这个消息之后才能处理`StaEnabledState`和`DeviceActiveStae`发送的消息,在这里,**消息处理的顺序是很重要的,所以要对Handler消息发送机制有一定的理解**

## WLAN扫描打开情况
首先是要进入`StaDisabledWithScanState`的状态
```Java
//WifiController$StaDisabledWithScanState
class StaDisabledWithScanState extends State {
    @Override
    public void enter() {
        // need to set the mode before starting supplicant because WSM will assume we are going
        // in to client mode
        //1. 设置对应的模式,SCAN_ONLY_WITH_WIFI_OFF_MODE的值是3
        mWifiStateMachine.setOperationalMode(WifiStateMachine.SCAN_ONLY_WITH_WIFI_OFF_MODE);
        //2. 打开Supplicant
        mWifiStateMachine.setSupplicantRunning(true);
        // Supplicant can't restart right away, so not the time we switched off
        mDisabledTimestamp = SystemClock.elapsedRealtime();
        mDeferredEnableSerialNumber++;
        mHaveDeferredEnable = false;
        mWifiStateMachine.clearANQPCache();
    }
    @Override
    public boolean processMessage(Message msg) {
        switch (msg.what) {
            case CMD_WIFI_TOGGLED:
                if (mSettingsStore.isWifiToggleEnabled()) {
                    /****省略无关代码****/
                    if (mDeviceIdle == false) {
                        //3. 切换到DeviceActiveState
                        transitionTo(mDeviceActiveState);
                    } else {
                        checkLocksAndTransitionWhenDeviceIdle();
                    }
                }
                break;
                /****省略无关代码****/
        }
    }
}
```
可以看到流程还是很不相同的,而且顺序要注意:**StaDisabledWithScanState是在enter()方法中开启的,所以对WifiStateMachine的处理上有很大的不同**.虽然最终都是切换到`DeviceActiveState`中,但是WLAN扫描打开的情况下,此时`WifiStateMachine`所处的状态和WLAN扫描关闭的情况完全不同
## 小结
从**WLAN扫描**功能开/关两个不同情况来看,可以总结出以下消息发送的顺序
1. **WLAN扫描**功能关闭: 接收到消息->`CMD_SET_OPERATIONAL_MODE(1)`->`CMD_START_SUPPLICANT`->`CMD_SET_OPERATIONAL_MODE(1)`
2. **WLAN扫描**功能打开: `CMD_SET_OPERATIONAL_MODE(3)`->`CMD_START_SUPPLICANT`->接收到消息->`CMD_START_SUPPLICANT`->`CMD_SET_OPERATIONAL_MODE(1)`

# WifiStateMachine处理
## WifiStateMachine结构图
`WifiStateMachine`的初始状态是`InitialState`, 没有什么可以分析的
![WifiStateMahchine structor](wifi-statemachine-structor.png)
## WifiStateMachine$InitialState处理
先从`InitialState`接收第一个消息开始
```Java
//WifiStateMachine$InitialState
class InitialState extends State {
    @Override
    public boolean processMessage(Message message) {
        logStateAndMessage(message, this);
        switch (message.what) {
            case CMD_SET_OPERATIONAL_MODE:
                //1. 这个是区分WLAN扫描功能的地方,后续会用到
                mOperationalMode = message.arg1;
                if (mOperationalMode != DISABLED_MODE) {
                    //2. 发送CMD_START_SUPPLICANT,这里需要区分WLAN扫描开/关的情况
                    sendMessage(CMD_START_SUPPLICANT);
                }
                break;
        }
        return HANDLED;
    }
}
```
接下来是第二个消息`CMD_START_SUPPLICANT`,注意:
1. WLAN扫描关闭: `CMD_START_SUPPLICANT`来自于WifiController$StaEnabledState, 并且这个消息是放在队列最后
2. WLAN扫描打开: `CMD_START_SUPPLICANT`来自于WifiStateMachine$StaDisabledWithScanState的enter()方法, 这个消息放在队列最前面

```Java
//WifiStateMachine$InitialState
class InitialState extends State {
    @Override
    public boolean processMessage(Message message) {
        logStateAndMessage(message, this);
        switch (message.what) {
            case CMD_START_SUPPLICANT:
                //1. 核心代码之一,启动Wifi驱动
                mClientInterface = mWifiNative.setupForClientMode();
                if (mClientInterface == null
                        || !mDeathRecipient.linkToDeath(mClientInterface.asBinder())) {
                    setWifiState(WifiManager.WIFI_STATE_UNKNOWN);
                    cleanup();
                    break;
                }

                try {
                    // A runtime crash or shutting down AP mode can leave
                    // IP addresses configured, and this affects
                    // connectivity when supplicant starts up.
                    // Ensure we have no IP addresses before a supplicant start.
                    mNwService.clearInterfaceAddresses(mInterfaceName);

                    // Set privacy extensions
                    mNwService.setInterfaceIpv6PrivacyExtensions(mInterfaceName, true);

                    // IPv6 is enabled only as long as access point is connected since:
                    // - IPv6 addresses and routes stick around after disconnection
                    // - kernel is unaware when connected and fails to start IPv6 negotiation
                    // - kernel can start autoconfiguration when 802.1x is not complete
                    mNwService.disableIpv6(mInterfaceName);
                } catch (RemoteException re) {
                    loge("Unable to change interface settings: " + re);
                } catch (IllegalStateException ie) {
                    loge("Unable to change interface settings: " + ie);
                }
                //2. 核心代码之一,启动supplicant
                if (!mWifiNative.enableSupplicant()) {
                    loge("Failed to start supplicant!");
                    setWifiState(WifiManager.WIFI_STATE_UNKNOWN);
                    cleanup();
                    break;
                }
                setWifiState(WIFI_STATE_ENABLING);
                if (mVerboseLoggingEnabled) log("Supplicant start successful");
                //3. 管理Wifi
                mWifiMonitor.startMonitoring(mInterfaceName, true);
                setSupplicantLogLevel();
                //4. 切换到SupplicantStartingState
                transitionTo(mSupplicantStartingState);
                break;
        }
        return HANDLED;
    }
}
```
Wifi驱动的源码没有进行过深入研究,这里就不进行讨论了,切换完成之后才会处理下一个消息,所以`InitialState`只处理了`CMD_SET_OPERATIONAL_MODE`和`CMD_START_SUPPLICANT`两个消息,其他都是在`SupplicantStartingState`处理.下面是切换到`SupplicantStartingState`处理消息的情况

## SupplicantStartingState处理
```Java
//StateMachine$SupplicantStartingState 
class SupplicantStartingState extends State {
    @Override
    public boolean processMessage(Message message) {
        logStateAndMessage(message, this);
        switch(message.what) {
            case WifiMonitor.SUP_CONNECTION_EVENT:
                if (mVerboseLoggingEnabled) log("Supplicant connection established");

                mSupplicantRestartCount = 0;
                /* Reset the supplicant state to indicate the supplicant
                 * state is not known at this time */
                mSupplicantStateTracker.sendMessage(CMD_RESET_SUPPLICANT_STATE);
                /* Initialize data structures */
                mLastBssid = null;
                mLastNetworkId = WifiConfiguration.INVALID_NETWORK_ID;
                mLastSignalLevel = -1;

                mWifiInfo.setMacAddress(mWifiNative.getMacAddress());
                // Attempt to migrate data out of legacy store.
                if (!mWifiConfigManager.migrateFromLegacyStore()) {
                    Log.e(TAG, "Failed to migrate from legacy config store");
                }
                initializeWpsDetails();
                sendSupplicantConnectionChangedBroadcast(true);
                transitionTo(mSupplicantStartedState);
                break;

            case CMD_START_SUPPLICANT:
            case CMD_STOP_SUPPLICANT:
            case CMD_START_AP:
            case CMD_STOP_AP:
            case CMD_SET_OPERATIONAL_MODE:
                messageHandlingStatus = MESSAGE_HANDLING_STATUS_DEFERRED;
                deferMessage(message);
                break;
        }
        return HANDLED;
    }
}
```
可以看到`SupplicantStartingState`就是将这些消息都defer掉了,等着下个状态再进行调用,只有`CMD_SET_HIGH_PERF_MODE`这个是在`DefaultState`中处理掉了,这里不去讨论
1. WLAN扫描关闭: CMD_SET_OPERATIONAL_MODE->CMD_START_SUPPLICANT(来自InitialState)
2. WLAN扫描打开: CMD_START_SUPPLICANT(来自InitialState)->CMD_START_SUPPLICANT->CMD_SET_OPERATIONAL_MODE
对于WLAN扫描打开的情况,头部的`CMD_START_SUPPLICANT`是从接收消息前就切换到`SupplicantStartingState`处理的.
SupplicantStartingState会一直处于这个状态,直到接收到`SUP_CONNECTION_EVENT`这个消息之后才进行切换.

## SupplicantStartedState处理
```Java
class SupplicantStartedState extends State {
    @Override
    public void enter() {
        /****省略无关代码****/
        //1. 根据mOperationMode区分WLAN扫描开启和关闭的情况
        if (mOperationalMode == SCAN_ONLY_MODE ||
                mOperationalMode == SCAN_ONLY_WITH_WIFI_OFF_MODE) {
            //2. WLAN扫描开启的情况
            mWifiNative.disconnect();
            setWifiState(WIFI_STATE_DISABLED);
            transitionTo(mScanModeState);
        } else if (mOperationalMode == CONNECT_MODE) {
            // Transitioning to Disconnected state will trigger a scan and subsequently AutoJoin
            //3. WLAN扫描关闭的情况
            transitionTo(mDisconnectedState);
        } else if (mOperationalMode == DISABLED_MODE) {
            transitionTo(mSupplicantStoppingState);
        }
    }
}
```
这里有个状态机的细节,如果在`enter()`方法中直接调用`transitionTo(destState)`方法的时候,会直接切换到`destState`, 具体可看[Android状态机模式解析(二)](http://www.godteen.com/posts/735aa624/). 所以这个`SupplicantStaredState`并不会处理deferMessage,而是到对应的`destState`处理

## DisconnectedState处理deferMessage
DisconnectedState处理的是WLAN扫描关闭情况下的deferMessage
```Java
class DisconnectedState extends State {
    @Override
    public boolean processMessage(Message message) {
        switch (message.what) {
            case CMD_SET_OPERATIONAL_MODE:
                if (message.arg1 != CONNECT_MODE) {
                    mOperationalMode = message.arg1;
                    if (mOperationalMode == DISABLED_MODE) {
                        transitionTo(mSupplicantStoppingState);
                    } else if (mOperationalMode == SCAN_ONLY_MODE
                            || mOperationalMode == SCAN_ONLY_WITH_WIFI_OFF_MODE) {
                        p2pSendMessage(CMD_DISABLE_P2P_REQ);
                        setWifiState(WIFI_STATE_DISABLED);
                        transitionTo(mScanModeState);
                    }

                }
                break;

        }
    }
}
```
可以看到对于deferMessage,并没有`DisconnectedState`并没有做处理,尤其是`CMD_START_SUPPLICANT`这个消息直接就是丢掉了.这里做的主要操作发送广播的作用,切换状态的时候要经过`ConnectModeState`, 所以要调用`ConnectModeState`的`enter()`方法.
```Java
class ConnectModeState extends State {
    @Override
    public void enter() {
        if (!mWifiNative.removeAllNetworks()) {
            loge("Failed to remove networks on entering connect mode");
        }

        // Let the system know that wifi is available in client mode.
        //1. 通知客户端Wifi已经打开
        setWifiState(WIFI_STATE_ENABLED);

        mNetworkInfo.setIsAvailable(true);
        if (mNetworkAgent != null) mNetworkAgent.sendNetworkInfo(mNetworkInfo);

        // initialize network state
        setNetworkDetailedState(DetailedState.DISCONNECTED);

        // Inform WifiConnectivityManager that Wifi is enabled
        mWifiConnectivityManager.setWifiEnabled(true);
        // Inform metrics that Wifi is Enabled (but not yet connected)
        mWifiMetrics.setWifiState(WifiMetricsProto.WifiLog.WIFI_DISCONNECTED);
        // Inform p2p service that wifi is up and ready when applicable
        p2pSendMessage(WifiStateMachine.CMD_ENABLE_P2P);
    }
}
private void setWifiState(int wifiState) {
    final int previousWifiState = mWifiState.get();

    try {
        if (wifiState == WIFI_STATE_ENABLED) {
            mBatteryStats.noteWifiOn();
        } else if (wifiState == WIFI_STATE_DISABLED) {
            mBatteryStats.noteWifiOff();
        }
    } catch (RemoteException e) {
        loge("Failed to note battery stats in wifi");
    }

    mWifiState.set(wifiState);

    if (mVerboseLoggingEnabled) log("setWifiState: " + syncGetWifiStateByName());
    //2. 发送广播
    final Intent intent = new Intent(WifiManager.WIFI_STATE_CHANGED_ACTION);
    intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY_BEFORE_BOOT);
    intent.putExtra(WifiManager.EXTRA_WIFI_STATE, wifiState);
    intent.putExtra(WifiManager.EXTRA_PREVIOUS_WIFI_STATE, previousWifiState);
    mContext.sendStickyBroadcastAsUser(intent, UserHandle.ALL);
}
```
这是我唯一知道的功能,至于其他和NetworkInfo和ConnectivityManager相关的代码,没有进行过研究

## ScanModeState处理deferMessage
```Java
class ScanModeState extends State {
    private int mLastOperationMode;
    @Override
    public void enter() {
        mLastOperationMode = mOperationalMode;
        mWifiStateTracker.updateState(WifiStateTracker.SCAN_MODE);
    }
    @Override
    public boolean processMessage(Message message) {
        logStateAndMessage(message, this);

        switch(message.what) {
            case CMD_SET_OPERATIONAL_MODE:
                //1. 此时接收的是第二次的CMD_SET_OPERATIONAL_MODE, 此时条件为true
                if (message.arg1 == CONNECT_MODE) {
                    mOperationalMode = CONNECT_MODE;
                    transitionTo(mDisconnectedState);
                } else if (message.arg1 == DISABLED_MODE) {
                    transitionTo(mSupplicantStoppingState);
                }
                // Nothing to do
                break;
                // Handle scan. All the connection related commands are
                // handled only in ConnectModeState
            case CMD_START_SCAN:
                handleScanRequest(message);
                break;
            case WifiMonitor.SUPPLICANT_STATE_CHANGE_EVENT:
                SupplicantState state = handleSupplicantStateChange(message);
                if (mVerboseLoggingEnabled) log("SupplicantState= " + state);
                break;
            default:
                return NOT_HANDLED;
        }
        return HANDLED;
    }

```
可以看到, 对于两个`CMD_START_SUPPLICANT`这个消息也是丢弃掉了, 对于`CMD_SET_OPERATIONAL_MODE`是切换到`DisconnectedState`处理,就和WLAN扫描关闭的情况是相同的了.
# 总结
对于打开Wifi来说,真正逻辑和代码细节比较多的地方是在状态机中,尤其是对于状态机原理和Handler机制的了解,不然很容易迷失在WifiStateMachine将近7000行的代码当中.这里有一个小技巧可以分享一下,通过日志去研究WifiStateMachine的状态切换,打开日志的方式:**设置->系统->开发者选项->启用WLAN详细日志记录功能**这样就开启了, 然后通过`adb logcat -s WifiStateMachine`进行查看,但是WifiController的日志好像是没有的
