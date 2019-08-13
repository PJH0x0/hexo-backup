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

Wifi扫描可以分为三个个阶段进行解析:
1. 第一阶段是扫描阶段, 这个阶段会发送扫描指令给驱动进行扫描
2. 第二阶段是回调阶段, 这个阶段主要是接收到内核扫描成功的回复后触发的回调
2. 第三阶段是获取扫描结果,这个阶段是接收到回调之后发送获取扫描结果指令给内核获取扫描结果

# 时序图
这个是时序图是Wifi打开完成之后触发的扫描流程, 包括触发扫描, 注册回调流程, 获取及获取扫描结果三个部分
这里说明一下简写, 如果用全写的话太长了
1. WSSI : WifiScanningServiceImpl
2. CH : ClientHandler
3. WSSSM : WifiSingleScanStateMachine
4. DSS : DriverStartedState
![Wifi扫描流程](wifi-scan.png)

# 扫描阶段

这个阶段触发的方式很多, 详情见*Android P Wifi扫描场景*, 但最终都是调用`WifiScanner`的`startScan()`方法
```java
//WifiScanner
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
//WifiScanner
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
//WifiScanningServiceImpl
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
//WifiScanningServiceImpl$WifiSingleScanStateMachine
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
这里有一个比较有趣的是正在扫描的状况, 如果正在扫描的扫描参数相同的话, 那么直接加入到mActiveScans中, 具体我们留到获取扫描结果的时候分析, 如果不相同则留到下次扫描的时候作为参数
```java
//WifiScanningServiceImpl$WifiSingleScanStateMachine
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
//WifiScanningServiceImpl$WifiSingleScanStateMachine
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
可以看到Wifi扫描的参数都是从`mPendingScans`中获取, 扫描完成之后会交换`mActiveScans`和`mPendingScans`, 并清除`mPendingScans`, `mActiveScans`用于标记是谁触发的扫描, 需要向它返回扫描结果

往下是`WifiScannerImpl`, 这个类有两个实现`HalWifiScannerImpl`和`WificondScannerImpl`, 但是`HalWifiScannerImpl`是委托`WificondScannerImpl`进行处理, `WificondScannerImpl`主要处理频率参数.
可以看到以上都是处理参数的过程, 这里有一份相关参数的类图, 大部分参数都不知道有什么用.
![class-param](class-param.png)
经过参数处理之后最后调用到`WificondControl.scan()`, `WificondControl`通过HIDL与C++进行通信
```java
//WificondControl
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

至于C++层的处理和Java层差不多, 也是处理参数, 最后是通过socket与内核进行通信, 发送`NL80211_CMD_TRIGGER_SCAN`命令触发扫描

# 回调阶段
## 注册回调
注册回调的分为两部分:
1. `WificondScannerImpl`向`WifiMonitor`注册回调, 这个是`WificondScannerImpl`初始化的时候完成`wifiMonitor.registerHandler(mIfaceName,WifiMonitor.PNO_SCAN_RESULTS_EVENT, mEventHandler);`
2. 在enable wifi的时候`WificondControl`向`IWifiScannerImpl`注册回调, 
```java
//WificondControl
try {
    IWifiScannerImpl wificondScanner = clientInterface.getWifiScannerImpl();
    if (wificondScanner == null) {
        Log.e(TAG, "Failed to get WificondScannerImpl");
        return null;
    }
    mWificondScanners.put(ifaceName, wificondScanner);
    Binder.allowBlocking(wificondScanner.asBinder());
    ScanEventHandler scanEventHandler = new ScanEventHandler(ifaceName);
    mScanEventHandlers.put(ifaceName,  scanEventHandler);
    wificondScanner.subscribeScanEvents(scanEventHandler);
    PnoScanEventHandler pnoScanEventHandler = new PnoScanEventHandler(ifaceName);
    mPnoScanEventHandlers.put(ifaceName,  pnoScanEventHandler);
    wificondScanner.subscribePnoScanEvents(pnoScanEventHandler);
} catch (RemoteException e) {
    Log.e(TAG, "Failed to refresh wificond scanner due to remote exception");
}
```
## 回调过程
回调过程具体流程如下, 最终是由`WificondScannerImpl`处理回调消息
![callback-flowchat](callback-flowchat.png)

# 获取扫描结果
获取wifi扫描结果也分两部分
## 应用层获取
由设置或者是开机向导界面获取wifi扫描结果显示给用户, 从`WifiManager.getScanResults()`开始到`ScanRequestProxy` 返回扫描结果, 这个流程是是比较简单的, 但是有个问题, 这个扫描是从哪来的?

## 系统层获取
由`WificondScannerImpl`处理回调消息之后获取扫描结果. 先从`WifiNative`获取扫描结果
```java
//WificondScannerImpl
private void pollLatestScanData() {
    synchronized (mSettingsLock) {
        if (mLastScanSettings == null) {
             // got a scan before we started scanning or after scan was canceled
            return;
        }

        // 获取Scan results
        mNativeScanResults = mWifiNative.getScanResults(mIfaceName);
        //省略Scan results处理过程
            mLatestSingleScanResult = new WifiScanner.ScanData(0, 0, 0,
                    isAllChannelsScanned(mLastScanSettings.singleScanFreqs),
                    singleScanResults.toArray(new ScanResult[singleScanResults.size()]));
        //发送消息给WifiSingleScanStateMachine, 通知可以取返回结果了
            mLastScanSettings.singleScanEventHandler
                    .onScanStatus(WifiNative.WIFI_SCAN_RESULTS_AVAILABLE);

        mLastScanSettings = null;
    }
}
```
通过`WifiNative`获取扫描的过程不再继续分析了, 因为这个最终还是调用到`WificondCtrol`里面和`IWifiServiceImpl`进行通信获取, 接下来说一下回调结果
```java
//WifiScanningServiceImpl$WifiSingleScanStateMachine
@Override
public void onScanStatus(int event) {
    if (DBG) localLog("onScanStatus event received, event=" + event);
    switch(event) {
        case WifiNative.WIFI_SCAN_RESULTS_AVAILABLE:
        case WifiNative.WIFI_SCAN_THRESHOLD_NUM_SCANS:
        case WifiNative.WIFI_SCAN_THRESHOLD_PERCENT:
            sendMessage(CMD_SCAN_RESULTS_AVAILABLE);
            break;
        case WifiNative.WIFI_SCAN_FAILED:
            sendMessage(CMD_SCAN_FAILED);
            break;
        default:
            Log.e(TAG, "Unknown scan status event: " + event);
            break;
    }
}
//WifiScanningServiceImpl$WifiSingleScanStateMachine$ScanningState
@Override
public boolean processMessage(Message msg) {
    switch (msg.what) {
        case CMD_SCAN_RESULTS_AVAILABLE:
            mWifiMetrics.incrementScanReturnEntry(
                    WifiMetricsProto.WifiLog.SCAN_SUCCESS,
                    mActiveScans.size());
            //通知WifiScanner
            reportScanResults(mScannerImpl.getLatestSingleScanResults());
            mActiveScans.clear();
            transitionTo(mIdleState);
            return HANDLED;
        case CMD_FULL_SCAN_RESULTS:
            reportFullScanResult((ScanResult) msg.obj, /* bucketsScanned */ msg.arg2);
            return HANDLED;
        case CMD_SCAN_FAILED:
            mWifiMetrics.incrementScanReturnEntry(
                    WifiMetricsProto.WifiLog.SCAN_UNKNOWN, mActiveScans.size());
            sendOpFailedToAllAndClear(mActiveScans, WifiScanner.REASON_UNSPECIFIED,
                    "Scan failed");
            transitionTo(mIdleState);
            return HANDLED;
        default:
            return NOT_HANDLED;
    }
```
这个`mScannerImpl.getLatestSingleScanResults()`返回的是`mLatestSingleScanResult`, `reportScanResults()`方法就是通过Messenger来通知WifiScanner处理扫描结果, WifiScanner就会通知ScanRequestProxy处理对应扫描结果,并保存
```java
//WifiScanningServiceImpl$WifiSingleScanStateMachine
void reportScanResults(ScanData results) {
    if (results != null && results.getResults() != null) {
        if (results.getResults().length > 0) {
            mWifiMetrics.incrementNonEmptyScanResultCount();
        } else {
            mWifiMetrics.incrementEmptyScanResultCount();
        }
    }
    ScanData[] allResults = new ScanData[] {results};
    //返回扫描结果
    for (RequestInfo<ScanSettings> entry : mActiveScans) {
        ScanData[] resultsToDeliver = ScanScheduleUtil.filterResultsForSettings(
                mChannelHelper, allResults, entry.settings, -1);
        WifiScanner.ParcelableScanData parcelableResultsToDeliver =
                new WifiScanner.ParcelableScanData(resultsToDeliver);
        logCallback("singleScanResults",  entry.clientInfo, entry.handlerId,
                describeForLog(resultsToDeliver));
        entry.reportEvent(WifiScanner.CMD_SCAN_RESULT, 0, parcelableResultsToDeliver);
        // make sure the handler is removed
        entry.reportEvent(WifiScanner.CMD_SINGLE_SCAN_COMPLETED, 0, null);
    }

    WifiScanner.ParcelableScanData parcelableAllResults =
            new WifiScanner.ParcelableScanData(allResults);
    for (RequestInfo<Void> entry : mSingleScanListeners) {
        logCallback("singleScanResults",  entry.clientInfo, entry.handlerId,
                describeForLog(allResults));
        entry.reportEvent(WifiScanner.CMD_SCAN_RESULT, 0, parcelableAllResults);
    }

    if (results.isAllChannelsScanned()) {
        mCachedScanResults.clear();
        mCachedScanResults.addAll(Arrays.asList(results.getResults()));
    }
}
```
这里就用到mActiveScans, 把扫描结果返回到到对应的调用者中, 最终是会回调到`WifiScanner`中
```java
//WifiScanner$ServiceHandler
@Override
public void handleMessage(Message msg) {
    switch (msg.what) {
        case AsyncChannel.CMD_CHANNEL_FULLY_CONNECTED:
            return;
        case AsyncChannel.CMD_CHANNEL_DISCONNECTED:
            Log.e(TAG, "Channel connection lost");
            // This will cause all further async API calls on the WifiManager
            // to fail and throw an exception
            mAsyncChannel = null;
            getLooper().quit();
            return;
    }

    Object listener = getListener(msg.arg2);

    if (listener == null) {
        if (DBG) Log.d(TAG, "invalid listener key = " + msg.arg2);
        return;
    } else {
        if (DBG) Log.d(TAG, "listener key = " + msg.arg2);
    }

    switch (msg.what) {
        case CMD_SCAN_RESULT :
            ((ScanListener) listener).onResults(
                    ((ParcelableScanData) msg.obj).getResults());
            return;

```
这里继续回调到`ScanRequestProxy`中
```java
//ScanRequestProxy$ScanRequestProxyScanListener 
@Override
public void onResults(WifiScanner.ScanData[] scanDatas) {
    if (mVerboseLoggingEnabled) {
        Log.d(TAG, "Scan results received");
    }
    // For single scans, the array size should always be 1.
    if (scanDatas.length != 1) {
        Log.wtf(TAG, "Found more than 1 batch of scan results, Failing...");
        sendScanResultBroadcastIfScanProcessingNotComplete(false);
        return;
    }
    WifiScanner.ScanData scanData = scanDatas[0];
    ScanResult[] scanResults = scanData.getResults();
    if (mVerboseLoggingEnabled) {
        Log.d(TAG, "Received " + scanResults.length + " scan results");
    }
    // Store the last scan results & send out the scan completion broadcast.
    mLastScanResults.clear();
    mLastScanResults.addAll(Arrays.asList(scanResults));
    sendScanResultBroadcastIfScanProcessingNotComplete(true);
}
```
以上, 没得了
