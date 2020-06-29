---
abbrlink: '0'
---
# UsbManager模块总结

## UsbDeviceManager初始化
初始化最主要的目的之一是决定使用哪一个Handler去处理usb消息, 这里假定是用的UsbHandlerLegacy, UsbGadget是Linux 2.6之后的支持的一种USB协议，但是不知道为何目前我们不支持
```java
//UsbDeviceManager.java
boolean halNotPresent = false;
try {
    IUsbGadget.getService(true);
} catch (RemoteException e) {
    Slog.e(TAG, "USB GADGET HAL present but exception thrown", e);
} catch (NoSuchElementException e) {
    halNotPresent = true;
    Slog.i(TAG, "USB GADGET HAL not present in the device", e);
}

if (halNotPresent) {
    /**
     * Initialze the legacy UsbHandler
     */
    mHandler = new UsbHandlerLegacy(FgThread.get().getLooper(), mContext, this,
            alsaManager, settingsManager);
} else {
    /**
     * Initialize HAL based UsbHandler
     */
    mHandler = new UsbHandlerHal(FgThread.get().getLooper(), mContext, this,
            alsaManager, settingsManager);
}
```

## 设置USB模式
流程如图

![usb-set-functions](./usb-set-function.png)

需要注意的是，setUsbConfig会执行两次，第一次是将之前设置的属性清除掉，第二次才是设置对应的属性。对应的属性键是`sys.usb.config`, 对应的属性值可能有`none, adb, rndis,
mtp, ptp, audio_source, midi, accessory`

