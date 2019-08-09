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
这个是时序图是Wifi打开完成之后触发的扫描流程
![Wifi扫描流程](sequence.png)

# 扫描阶段

这个阶段触发的方式很多, 详情见*Android P Wifi扫描场景*, 但最终都是调用`WifiScanner`的`startScan()`方法
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
这里使用了`AsyncChannel`进行传递消息, `AsyncChannel`的原理就是利用了Android`Messenger`来在进程间通信, `Messenger`的介绍可以看一下[Android 基于Message的进程间通信 Messenger完全解析](https://blog.csdn.net/lmj623565791/article/details/47017485).
理解`AsyncChannel`传递消息的关键在于`Messenger`的来源, 目标`Messenger`的获取一般都是在初始化的时候
```java
public WifiScanner(Context context, IWifiScanner service, Looper looper) {
    Messenger messenger = null;
    try {
        messenger = mService.getMessenger();
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();

}
```
这里就要看是谁继承了这个`IWifiScanner`, 这个命名方式明显是一个AIDL的命名, 最后查到出处是`public class WifiScanningServiceImpl extends IWifiScanner.Stub`, 接下来是找到`Handler`处理消息的地方.
```java
@Override
public Messenger getMessenger() {
    if (mClientHandler != null) {
        mLog.trace("getMessenger() uid=%").c(Binder.getCallingUid()).flush();
        return new Messenger(mClientHandler);
    }
    loge("WifiScanningServiceImpl trying to get messenger w/o initialization");
    return null;
}
private class ClientHandler extends WifiHandler {
    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case WifiScanner.CMD_START_SINGLE_SCAN:
            case WifiScanner.CMD_STOP_SINGLE_SCAN:
                mSingleScanStateMachine.sendMessage(Message.obtain(msg));
                break;
 
        }
    }
}
```
接下来又会进入到StateMachine当中, 正常处理这条消息的是`DriverStartedState`.
```java
class DriverStartedState extends State {
    @Override
    public boolean processMessage(Message msg) {
        switch (msg.what) {
            case WifiScanner.CMD_START_SINGLE_SCAN:
                scanParams.setDefusable(true);
                ScanSettings scanSettings =
                        scanParams.getParcelable(WifiScanner.SCAN_PARAMS_SCAN_SETTINGS_KEY);
                WorkSource workSource =
                        scanParams.getParcelable(WifiScanner.SCAN_PARAMS_WORK_SOURCE_KEY);
                if (validateScanRequest(ci, handler, scanSettings)) {
                    logScanRequest("addSingleScanRequest", ci, handler, workSource,
                            scanSettings, null);
                    replySucceeded(msg);

                    // If there is an active scan that will fulfill the scan request then
                    // mark this request as an active scan, otherwise mark it pending.
                    // If were not currently scanning then try to start a scan. Otherwise
                    // this scan will be scheduled when transitioning back to IdleState
                    // after finishing the current scan.
                    if (getCurrentState() == mScanningState) {
                        if (activeScanSatisfies(scanSettings)) {
                            //下下次扫描
                            mActiveScans.addRequest(ci, handler, workSource, scanSettings);
                        } else {
                            //下一次扫描
                            mPendingScans.addRequest(ci, handler, workSource, scanSettings);
                        }
                    } else {
                        //立即扫描
                        mPendingScans.addRequest(ci, handler, workSource, scanSettings);
                        tryToStartNewScan();
                    }
                } else {
                    logCallback("singleScanInvalidRequest",  ci, handler, "bad request");
                    replyFailed(msg, WifiScanner.REASON_INVALID_REQUEST, "bad request");
                    mWifiMetrics.incrementScanReturnEntry(
                            WifiMetricsProto.WifiLog.SCAN_FAILURE_INVALID_CONFIGURATION, 1);
                }
                return HANDLED;
        }
    }
}
```
这里需要区分是否正在扫描,如果正在扫描的话会延迟到下一次的扫描或者进入到`IdleState`
```java
class IdleState extends State {
    @Override
    public void enter() {
        //IdleState也会触发扫描
        tryToStartNewScan();
    }

    @Override
    public boolean processMessage(Message msg) {
        return NOT_HANDLED;
    }
}
```
接下来是`tryToStartNewScan()`
```java
void tryToStartNewScan() {
    //省略了很多参数处理,但是这些参数处理也很重要
    WifiNative.ScanSettings settings = new WifiNative.ScanSettings();

    for (RequestInfo<ScanSettings> entry : mPendingScans) {
        settings.scanType =
            mergeScanTypes(settings.scanType, getNativeScanType(entry.settings.type));
        channels.addChannels(entry.settings);
        if (entry.settings.hiddenNetworks != null) {
            for (int i = 0; i < entry.settings.hiddenNetworks.length; i++) {
                WifiNative.HiddenNetwork hiddenNetwork = new WifiNative.HiddenNetwork();
                hiddenNetwork.ssid = entry.settings.hiddenNetworks[i].ssid;
                hiddenNetworkList.add(hiddenNetwork);
            }
        }
        if ((entry.settings.reportEvents & WifiScanner.REPORT_EVENT_FULL_SCAN_RESULT)
                != 0) {
            bucketSettings.report_events |= WifiScanner.REPORT_EVENT_FULL_SCAN_RESULT;
        }
    }

    if (mScannerImpl.startSingleScan(settings, this)) {
        // store the active scan settings
        mActiveScanSettings = settings;
        // swap pending and active scan requests
        RequestList<ScanSettings> tmp = mActiveScans;
        mActiveScans = mPendingScans;
        mPendingScans = tmp;
        // make sure that the pending list is clear
        mPendingScans.clear();
        transitionTo(mScanningState);
    } else {
        mWifiMetrics.incrementScanReturnEntry(
                WifiMetricsProto.WifiLog.SCAN_UNKNOWN, mPendingScans.size());
        // notify and cancel failed scans
        sendOpFailedToAllAndClear(mPendingScans, WifiScanner.REASON_UNSPECIFIED,
                "Failed to start single scan");
    }
}
```
可以看到Wifi扫描的参数都是从`mPendingScans`中获取, 这也是为何加入到`mActiveScans`里面的参数需要两次之后才能扫描到,扫描完成之后会交换`mActiveScans`和`mPendingScans`, 这波操作目前还没弄清是什么意思

往下是`WifiScannerImpl`, 这个类有两个实现`HalWifiScannerImpl`和`WificondScannerImpl`, 但是`HalWifiScannerImpl`是委托`WificondScannerImpl`进行处理, `WificondScannerImpl`主要处理频率参数.
可以看到以上都是处理参数的过程, 这里有一份相关参数的类图, 大部分参数都不知道有什么用.
![class-param](class-param.png)
经过参数处理之后最后调用到`WificondControl.scan()`, `WificondControl`通过HIDL与C++进行通信
```java
public boolean scan(@NonNull String ifaceName,
                    int scanType,
                    Set<Integer> freqs,
                    Set<String> hiddenNetworkSSIDs) {
    IWifiScannerImpl scannerImpl = getScannerImpl(ifaceName);
    if (scannerImpl == null) {
        Log.e(TAG, "No valid wificond scanner interface handler");
        return false;
    }
    SingleScanSettings settings = new SingleScanSettings();
    try {
        settings.scanType = getScanType(scanType);
    } catch (IllegalArgumentException e) {
        Log.e(TAG, "Invalid scan type ", e);
        return false;
    }
    settings.channelSettings  = new ArrayList<>();
    settings.hiddenNetworks  = new ArrayList<>();

    if (freqs != null) {
        for (Integer freq : freqs) {
            Log.i(TAG, "freq>>>" + freq);
            ChannelSettings channel = new ChannelSettings();
            channel.frequency = freq;
            settings.channelSettings.add(channel);
        }
    }
    if (hiddenNetworkSSIDs != null) {
        for (String ssid : hiddenNetworkSSIDs) {
            HiddenNetwork network = new HiddenNetwork();
            Log.i(TAG, "hidden network ssid>>>" + ssid);
            try {
                network.ssid = NativeUtil.byteArrayFromArrayList(NativeUtil.decodeSsid(ssid));
            } catch (IllegalArgumentException e) {
                Log.e(TAG, "Illegal argument " + ssid, e);
                continue;
            }
            settings.hiddenNetworks.add(network);
        }
    }
```
可以看到只需要4个参数就可以进行扫描:
1. ifaceName, 这个硬件借口名称, 一般只有一个无线网卡的话就是wlan0, 由enable wifi 的时候获取
2. scanType, 这个有三种类型`TYPE_LOW_LATENCY`,`TYPE_LOW_POWER`, `TYPE_HIGH_ACCURACY`, 一般都是`TYPE_HIGH_ACCURACY`这种类型
3. feqs, 这个是扫描信道的频率, 分为2.4GHz和5GHz
4. hiddenNetworkSSIDs, 这个扫描隐藏热点的SSID, 这个一般为空, 只有当用户在添加网络的高级选项选择了是隐藏网络选项, 这个才会添加对应ssid, 硬件也会主动去扫描这个ssid的热点
至于C++层的处理感觉和Java层差不多, 也是处理参数, 最后是通过socket与内核进行通信, 发送`NL80211_CMD_TRIGGER_SCAN`触发扫描


