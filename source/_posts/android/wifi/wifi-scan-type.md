---
title: Android P Wifi扫描场景
date: 2019/8/2
categories:
  - android
  - wifi
tags:
  - android
  - source
---
# Android O Wifi扫描场景
抄自[Android wifi扫描机制(Android O)](https://blog.csdn.net/h784707460/article/details/79658950), 有想研究的可以直接看原文, 就是排版有点不好
1. 亮屏情况下，在Wifi settings界面，固定扫描，时间间隔为10s。
2. 亮屏情况下，非Wifi settings界面，二进制指数退避扫描，退避算法：interval*(2^n), 最小间隔min=20s, 最大间隔max=160s.
3. 灭屏情况下，有保存网络时，若已连接，不扫描，否则，PNO扫描，即只扫描已保存的网络。最小间隔min=20s，最大间隔max=60s. 
4. 无保存网络情况下，固定扫描，间隔为5分钟，用于通知用户周围存在可用开放网络。


# Android P Wifi扫描场景
Android P取消了第4情况固定扫描, 其实第4情况在Android O中没有亮屏和灭屏之分. 第1, 2种情况属于Single Scan, 第3种情况属于PNO Scan

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
        clearAccessPointsAndConditionallyUpdate();
        mLastInfo = null;
        mLastNetworkInfo = null;
        if (mScanner != null) {
            //停止扫描
            mScanner.pause();
        }
        mStaleScanResults = true;
    }
    mListener.onWifiStateChanged(state);
}

```
2. 在onStart生命周期中触发扫描
```java
@Override
@MainThread
public void onStart() {
    // fetch current ScanResults instead of waiting for broadcast of fresh results
    forceUpdate();

    registerScoreCache();

    mNetworkScoringUiEnabled =
            Settings.Global.getInt(
                    mContext.getContentResolver(),
                    Settings.Global.NETWORK_SCORING_UI_ENABLED, 0) == 1;

    mMaxSpeedLabelScoreCacheAge =
            Settings.Global.getLong(
                    mContext.getContentResolver(),
                    Settings.Global.SPEED_LABEL_CACHE_EVICTION_AGE_MILLIS,
                    DEFAULT_MAX_CACHED_SCORE_AGE_MILLIS);

    //触发扫描
    resumeScanning();
    if (!mRegistered) {
        mContext.registerReceiver(mReceiver, mFilter, null /* permission */, mWorkHandler);
        // NetworkCallback objects cannot be reused. http://b/20701525 .
        mNetworkCallback = new WifiTrackerNetworkCallback();
        mConnectivityManager.registerNetworkCallback(
                mNetworkRequest, mNetworkCallback, mWorkHandler);
        mRegistered = true;
    }
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

## 第二种情况
这种情况下也有两种方式进行触发:
1. 通过



