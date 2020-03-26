---
title: Android集成自定义Google Setup Wizard
date: 2019/11/19
tags:
  - android
  - setupwizard
categories:
  - android
abbrlink: a125a8ec
---

>前言:本文基于Android P/Q进行的实验,其他版本并不能保证成功
Google SetupWizard中文翻译为设置向导, 是GMS应用的一部分, 不过Google提供了文档供OEM厂商或者运营商进行定制, 文档的名称是*Configuring the Setup Wizard*
# Android.mk设置
```shell
#Android Q library
LOCAL_STATIC_ANDROID_LIBRARIES := \
	setupcompat \
	setupdesign
#Android P library
include frameworks/opt/setupwizard/library/common-gingerbread.mk
```
Android.mk中需要添加SetupWizard库,这个是为了使用Google SetupWizard的主题而添加的,如果使用自定义的主题的话,有可能测不过GMS. Android P使用的是frameworks/opt/setupwizard, 而Android Q必须换成external下的setupcompat和setupdesign.另外就是新增页面的根布局**必须**要使用库中的GlifLayout(Android P)或者是PartenerCustomizationLayout(Android Q)
# 添加BroadcastReceiver
按照Google的文档,我们需要添加一个空的广播用于接收`com.android.setupwizard.action.PARTNER_CUSTOMIZATION`,这个广播可以不做任何动作

```xml
<!--AndroidManifest.xml-->
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.setupwizard.demo">

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true">
        <receiver android:name=".SuwCustomizationReceiver">
            <intent-filter>
                <action android:name="com.android.setupwizard.action.PARTNER_CUSTOMIZATION" />
            </intent-filter>
        </receiver>

    </application>

</manifest>
```
```java
//SuwCustomizationReceiver.java
package com.setupwizard.demo;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;

public class SuwCustomizationReceiver extends BroadcastReceiver {

    public SuwCustomizationReceiver() {
    }

    @Override
    public void onReceive(Context context, Intent intent) {
    }
}
```
# 添加raw目录
一般使用的是`GmsSampleIntegration`这个应用里面的raw目录,这个应用是Google提供的一段集成自定义SetupWizard的demo例子,一般是在`vendor/${partener_gms}/apps`目录下.<br>
而raw这个目录则包含了Google SetupWizard的流程, Google SetupWizard就会按照这些xml的流程跳转到对应的界面, 包括自定义的界面.所以这个raw目录是自定义Google SetupWizard的核心, 无论是添加还是修改页面,都很可能会涉及到<br>
移植了这个raw目录之后需要修改包名`com.google.android.gmsintegration`为自己的包名
1. 进入res/raw目录
2. 执行*sed -i "s/com.google.android.gmsintegration/com.setupwizard.demo/g" \`grep com.google.android.gmsintegration -rl .\`*
3. 执行`ag com.setupwizard.demo`看是否与AndroidManifest.xml中的包名是否保持一致即可

# 添加启动时的uri
定制SetupWizard需要指定开机时的启动的流程, 否则会用Google SetupWizard.apk中的xml/目录下的流程
```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <string name="wizard_script_uri" translatable="false">android.resource://com.setupwizard.demo/raw/wizard_script</string>
</resources>
```
# 简单的自定义
为了区分与原生Google SetupWizard的不同, 在`strings.xml`添加一行字符串用于overlay原生的字符串, 在Google文档中还规定了许多可以定制的资源,详情见文档中的*Providing resource assets*
```xml
<!--strings.xml-->
<!--sim_missing_text出现在设置向导第二页, 前提是设备没有插入sim卡-->
<string name="sim_missing_text">Demo for sim missing</string>
```
# 总结
以上步骤之后就可以编译自定义的SetupWizard然后push进去启动了, 不插卡的情况下在第二页中就可以看到效果了, **拷贝代码的时候要注意修改包名**
