---
title: Android P Wifi扫描场景
date: 2019/8/2
categories:
  - android
  - wifi
tags:
  - android
  - source
abbrlink: f0e0a512
---
# Android O Wifi扫描场景
抄自[Android wifi扫描机制(Android O)](https://blog.csdn.net/h784707460/article/details/79658950), 有想研究的可以直接看原文, 就是排版有点不好
1. 亮屏情况下，在Wifi settings界面，固定扫描，时间间隔为10s。
2. 亮屏情况下，非Wifi settings界面，二进制指数退避扫描，退避算法：interval*(2^n), 最小间隔min=20s, 最大间隔max=160s.
3. 灭屏情况下，有保存网络时，若已连接，不扫描，否则，PNO扫描，即只扫描已保存的网络。最小间隔min=20s，最大间隔max=60s. 
4. 无保存网络情况下，固定扫描，间隔为5分钟，用于通知用户周围存在可用开放网络。


# Android P Wifi扫描场景
Android P取消了第4种情况固定扫描, 其实第4种情况在Android O中没有亮屏和灭屏之分. 第1, 2种情况属于Single Scan, 第3种情况属于PNO Scan

# 如何触发
## 第一种情况
触发在`WifiTracker.java`中, 这种情况有两种触发方式
1. 接收到`WIFI_STATE_CHANGED_ACTION`广播, 触发
```java
final BroadcastReceiver mReceiver = new BroadcastReceiver() {
    @Override
    public void onReceive(Context context, Intent intent) {
        String action = intent.getAction();

        if (WifiManager.WIFI_STATE_CHANGED_ACTION.equals(action)) {
            updateWifiState(
                    intent.getIntExtra(WifiManager.EXTRA_WIFI_STATE,
        }
    }
}
private void updateWifiState(int state) {
    if (state == WifiManager.WIFI_STATE_ENABLED) {
        if (mScanner != null) {
            // We only need to resume if mScanner isn't null because
            // that means we want to be scanning.
            //触发扫描
            mScanner.resume();
        }
    } else {
        if (mScanner != null) {
            //停止扫描
            mScanner.pause();
        }
        mStaleScanResults = true;
    }
}

```
2. 在onStart生命周期中触发扫描
```java
@Override
@MainThread
public void onStart() {
    //省略无关代码
    //触发扫描
    resumeScanning();
}
public void resumeScanning() {
    if (mScanner == null) {
        mScanner = new Scanner();
    }

    if (mWifiManager.isWifiEnabled()) {
        mScanner.resume();
    }
}
```

上面就是触发的两种方式, 最后都是调用`mScanner.resume()`, 这个其实WifiTracker的内部类
```java
private static final int WIFI_RESCAN_INTERVAL_MS = 10 * 1000;

@VisibleForTesting
class Scanner extends Handler {
    static final int MSG_SCAN = 0;

    private int mRetry = 0;

    void resume() {
        if (!hasMessages(MSG_SCAN)) {
            sendEmptyMessage(MSG_SCAN);
        }
    }

    //停止扫描
    void pause() {
        mRetry = 0;
        removeMessages(MSG_SCAN);
    }

    @VisibleForTesting
    boolean isScanning() {
        return hasMessages(MSG_SCAN);
    }

    @Override
    public void handleMessage(Message message) {
        if (message.what != MSG_SCAN) return;
        //调用WifiManager的扫描方法
        if (mWifiManager.startScan()) {
            mRetry = 0;
        } else if (++mRetry >= 3) {
            mRetry = 0;
            if (mContext != null) {
                Toast.makeText(mContext, R.string.wifi_fail_to_scan, Toast.LENGTH_LONG).show();
            }
            return;
        }
        //每隔10*1000 ms触发一次
        sendEmptyMessageDelayed(MSG_SCAN, WIFI_RESCAN_INTERVAL_MS);
    }
}
```
上面说了如何触发扫描的方式, 那么如何停止扫描呢? 上面第一种方式中当wifi没打开的时候会触发, 另外就是`onStop`生命周期的时候也会进行触发, 这里不再细述

## 第二/三种情况
这两种种情况下有比较多触发方式, 但是主要的还是监听连接状态变化以及亮灭屏变化触发扫描:
1. `handleConnectionStateChanged()`, 这个主要由`WifiStateMachine`的状态进行调用传入不同参数
```java
public void handleConnectionStateChanged(int state) {
    localLog("handleConnectionStateChanged: state=" + stateToString(state));

    mWifiState = state;

    if (mWifiState == WIFI_STATE_CONNECTED) {
        mOpenNetworkNotifier.handleWifiConnected();
        mCarrierNetworkNotifier.handleWifiConnected();
    }

    // Reset BSSID of last connection attempt and kick off
    // the watchdog timer if entering disconnected state.
    if (mWifiState == WIFI_STATE_DISCONNECTED) {
        mLastConnectionAttemptBssid = null;
        scheduleWatchdogTimer();
        startConnectivityScan(SCAN_IMMEDIATELY);
    } else {
        startConnectivityScan(SCAN_ON_SCHEDULE);
    }
}

```
2. `handleScreenStateChanged()`, 这个也是由`WifiStateMachine`进行调用的
```java
public void handleScreenStateChanged(boolean screenOn) {
    localLog("handleScreenStateChanged: screenOn=" + screenOn);

    mScreenOn = screenOn;

    mOpenNetworkNotifier.handleScreenStateChanged(screenOn);
    mCarrierNetworkNotifier.handleScreenStateChanged(screenOn);

    startConnectivityScan(SCAN_ON_SCHEDULE);
}
```


```java
private void startConnectivityScan(boolean scanImmediately) {
    localLog("startConnectivityScan: screenOn=" + mScreenOn
            + " wifiState=" + stateToString(mWifiState)
            + " scanImmediately=" + scanImmediately
            + " wifiEnabled=" + mWifiEnabled
            + " wifiConnectivityManagerEnabled="
            + mWifiConnectivityManagerEnabled);

    if (!mWifiEnabled || !mWifiConnectivityManagerEnabled) {
        return;
    }

    // Always stop outstanding connecivity scan if there is any
    stopConnectivityScan();

    // Don't start a connectivity scan while Wifi is in the transition
    // between connected and disconnected states.
    if (mWifiState != WIFI_STATE_CONNECTED && mWifiState != WIFI_STATE_DISCONNECTED) {
        return;
    }

    if (mScreenOn) {
        //触发第二种情况扫描
        startPeriodicScan(scanImmediately);
    } else {
        //触发第三中情况扫描
        if (mWifiState == WIFI_STATE_DISCONNECTED && !mPnoScanStarted) {
            startDisconnectedPnoScan();
        }
    }
}
```
### 第二种情况扫描分析
```java
private void startPeriodicScan(boolean scanImmediately) {
    mPnoScanListener.resetLowRssiNetworkRetryDelay();

    // No connectivity scan if auto roaming is disabled.
    if (mWifiState == WIFI_STATE_CONNECTED && !mEnableAutoJoinWhenAssociated) {
        return;
    }

    // Due to b/28020168, timer based single scan will be scheduled
    // to provide periodic scan in an exponential backoff fashion.
    if (scanImmediately) {
        resetLastPeriodicSingleScanTimeStamp();
    }
    mPeriodicSingleScanInterval = PERIODIC_SCAN_INTERVAL_MS;
    startPeriodicSingleScan();
}
private void startPeriodicSingleScan() {
    long currentTimeStamp = mClock.getElapsedSinceBootMillis();

    //scanImmediately的情况下不会走这里
    if (mLastPeriodicSingleScanTimeStamp != RESET_TIME_STAMP) {
        long msSinceLastScan = currentTimeStamp - mLastPeriodicSingleScanTimeStamp;
        if (msSinceLastScan < PERIODIC_SCAN_INTERVAL_MS) {
            localLog("Last periodic single scan started " + msSinceLastScan
                    + "ms ago, defer this new scan request.");
            schedulePeriodicScanTimer(PERIODIC_SCAN_INTERVAL_MS - (int) msSinceLastScan);
            return;
        }
    }


        mLastPeriodicSingleScanTimeStamp = currentTimeStamp;
        startSingleScan(isFullBandScan, WIFI_WORK_SOURCE);
        schedulePeriodicScanTimer(mPeriodicSingleScanInterval);

        // Set up the next scan interval in an exponential backoff fashion.
        mPeriodicSingleScanInterval *= 2;
        if (mPeriodicSingleScanInterval >  MAX_PERIODIC_SCAN_INTERVAL_MS) {
            mPeriodicSingleScanInterval = MAX_PERIODIC_SCAN_INTERVAL_MS;
        }
}

```
如果`scanImmediately == true`的话, `mLastPeriodicSingleScanTimeStamp`会被重置为`RESET_TIME_STAMP`, 所以会直接扫描. 而这个算法大概的意思是:<br>
1. 第一次扫描算作没有间隔, 第一次和第二次之间的扫描的间隔是20s
2. 每扫描一次增加一倍的扫描时间, 直到最大的160s, 也就6次就会达到最大值
3. 6次之后的每次扫描时间都是160s, 直到再次触发屏幕亮屏

### 第三种情况扫描分析
第三种PNO 扫描只有在灭屏并且无连接的情况下触发, 这种情况会单独有一个章节进行分析



