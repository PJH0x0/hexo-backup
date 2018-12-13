---
title: WifiService启动流程
date: 2018/12/12
tags:
  - android
  - wifi
categories:
  - android
  - wifi
abbrlink: d1635d94
---
> 前言:一篇关于WifiService启动的文章,本人也是刚刚接触wifi,把阅读源码的心得记录下来,供大家一起学习与交流.阅读本文需要的条件: 对于源码有一定的认知, 了解Java语言

# 操作系统
Android O

# 相关源码
```Java
frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiServiceImpl.java
frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiService.java
frameworks/base/services/java/com/android/server/SystemServer.java
frameworks/base/services/core/java/com/android/server/SystemService.java
frameworks/base/services/core/java/com/android/server/SystemServiceManager.java
frameworks/base/core/java/android/app/SystemServiceRegistry.java
```
# 时序图
时序图不是很规范,见谅
![时序图](start.png)
# 启动WifiService
WifiService在开机时就启动了. 
```Java
//SystemServer.java
// Wifi Service must be started first for wifi-related services.
mSystemServiceManager.startService(WIFI_SERVICE_CLASS);

//SystemServiceManager
public SystemService startService(String className) {
    final Class<SystemService> serviceClass;
    try {
        //1. 获取SystemService的Class对象
        serviceClass = (Class<SystemService>)Class.forName(className);
    } catch (ClassNotFoundException ex) {
        Slog.i(TAG, "Starting " + className);
        throw new RuntimeException("Failed to create service " + className
                + ": service class not found, usually indicates that the caller should "
                + "have called PackageManager.hasSystemFeature() to check whether the "
                + "feature is available on this device before trying to start the "
                + "services that implement it", ex);
    }
    return startService(serviceClass);
}

public <T extends SystemService> T startService(Class<T> serviceClass) {
    try {
        final String name = serviceClass.getName();
        Slog.i(TAG, "Starting " + name);
        Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "StartService " + name);

        // Create the service.
        if (!SystemService.class.isAssignableFrom(serviceClass)) {
            throw new RuntimeException("Failed to create " + name
                    + ": service must extend " + SystemService.class.getName());
        }
        final T service;
        try {
            //2. 调用WifiService的含有Context参数的构造方法
            Constructor<T> constructor = serviceClass.getConstructor(Context.class);
            service = constructor.newInstance(mContext);
        } catch (InstantiationException ex) {
            throw new RuntimeException("Failed to create service " + name
                    + ": service could not be instantiated", ex);
        } catch (IllegalAccessException ex) {
            throw new RuntimeException("Failed to create service " + name
                    + ": service must have a public constructor with a Context argument", ex);
        } catch (NoSuchMethodException ex) {
            throw new RuntimeException("Failed to create service " + name
                    + ": service must have a public constructor with a Context argument", ex);
        } catch (InvocationTargetException ex) {
            throw new RuntimeException("Failed to create service " + name
                    + ": service constructor threw an exception", ex);
        }

        // Register it.
        mServices.add(service);

        // Start it.
        try {
            //3. 调用WifiService的onStart()方法
            service.onStart();
        } catch (RuntimeException ex) {
            throw new RuntimeException("Failed to start service " + name
                    + ": onStart threw an exception", ex);
        }
        return service;
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
    }
}
```
根据注释1,2中可以得知, SystemServiceManager是通过反射进行创建WifiService对象的,接下来看一下WifiService的构造方法是怎样的
```Java
//WifiService.java
public WifiService(Context context) {
    super(context);
    //创建WifiServiceImpl对象
    mImpl = new WifiServiceImpl(context, new WifiInjector(context), new WifiAsyncChannel(TAG));
}
```
WifiServiceImpl继承了IWifiManager.Stub,是Binder通信的服务端.它实现了与Binder客户端WifiManager进行通信.之后的Wifi的扫描,保存,连接等动作都是由它完成的.
接下来是将WifiServiceImpl保存到ServiceManager中
```Java
//WifiService.java
@Override
public void onStart() {
    Log.i(TAG, "Registering " + Context.WIFI_SERVICE);
    publishBinderService(Context.WIFI_SERVICE, mImpl);
}
//SystemService.java
protected final void publishBinderService(String name, IBinder service) {
    publishBinderService(name, service, false);
}

protected final void publishBinderService(String name, IBinder service,
        boolean allowIsolated) {
    //此处不再讨论,因为ServiceManager再往下就是NDK的Binder服务注册流程
    ServiceManager.addService(name, service, allowIsolated);
}
```
这块主要功能就是向ServiceManager中注册服务的,然后提供给客户端app,客户端app则通过Context创建WifiManager与WifiServiceImpl通信.
以上就是WifiService的所有流程, 有问题或不足之处可以向我发送Email: godteen.peng@gmail.com
还有一个就是自动开启Wifi的功能,用户如果关机前打开了WiFi,那么开机后也应该打开WiFi.
```Java
@Override
public void onBootPhase(int phase) {
    if (phase == SystemService.PHASE_SYSTEM_SERVICES_READY) {
        mImpl.checkAndStartWifi();
    }
}
```
onBootPhase方法是在SystemServer.startOtherServices基本完成之后调用的,接下来进入WifiServiceImpl的操作
```Java
//WifiServiceImpl.java
//获取关机前的wifi状态
boolean wifiEnabled = mSettingsStore.isWifiToggleEnabled();
//如果关机前开启了Wifi,那就也开启Wifi
if (wifiEnabled) {
    try {
        setWifiEnabled(mContext.getPackageName(), wifiEnabled);
    } catch (RemoteException e) {
        /* ignore - local call */
    }
}
//WifiSettingsStore.java
//获取关机前的wifi状态
public synchronized boolean isWifiToggleEnabled() {
    //mCheckSavedStateAtBoot默认是false所以,先走这
    if (!mCheckSavedStateAtBoot) {
        mCheckSavedStateAtBoot = true;
        //清除已保存的状态,永远返回false
        if (testAndClearWifiSavedState()) return true;
    }

    //是否是飞行模式
    if (mAirplaneModeOn) {
        return mPersistWifiState == WIFI_ENABLED_AIRPLANE_OVERRIDE;
    } else {
        return mPersistWifiState != WIFI_DISABLED;
    }
}

//永远都是返回false
private boolean testAndClearWifiSavedState() {
    int wifiSavedState = getWifiSavedState();
    if (wifiSavedState == WIFI_ENABLED) {
        setWifiSavedState(WIFI_DISABLED);
    }
    return (wifiSavedState == WIFI_ENABLED);
}

//mPersistWifiState在WifiSettingsStore的构造方法中获取
WifiSettingsStore(Context context) {
    mContext = context;
    mAirplaneModeOn = getPersistedAirplaneModeOn();
    mPersistWifiState = getPersistedWifiState();
    mScanAlwaysAvailable = getPersistedScanAlwaysAvailable();
}

//获取数据库的值,
private int getPersistedWifiState() {
    final ContentResolver cr = mContext.getContentResolver();
    try {
        return Settings.Global.getInt(cr, Settings.Global.WIFI_ON);
    } catch (Settings.SettingNotFoundException e) {
        Settings.Global.putInt(cr, Settings.Global.WIFI_ON, WIFI_DISABLED);
        return WIFI_DISABLED;
    }
}
```
上面的注释应该是比较详细了,逻辑也很简单.之后就是正常的Wifi打开流程,这个后续再进行讨论

