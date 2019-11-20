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
Google SetupWizard中文翻译为设置向导, 是GMS应用的一部分, 不过Google提供了文档供OEM厂商或者运营商进行定制, 以下是我对于该文档的一个简单总结:
# Android.mk设置
```shell
#Android Q library
LOCAL_STATIC_ANDROID_LIBRARIES := \
	setupcompat \
	setupdesign
#Android P library
include frameworks/opt/setupwizard/library/common-gingerbread.mk
```
Android.mk中需要添加SetupWizard库,这个是为了使用Google SetupWizard的主题而添加的,如果使用自定义的主题的话,有可能测不过GMS.另外就是新增页面的根布局**必须**要使用库中的GlifLayout或者是PartenerCustomizationLayout
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
# 添加raw/目录
一般使用的是`GmsSampleIntegration`这个应用里面的raw目录,这个应用是Google提供的一段集成自定义SetupWizard的demo例子,一般是在`vendor/${partener_gms}/apps`目录下.
而raw/这个目录则包含了Google SetupWizard的流程, Google SetupWizard就会按照这些xml的流程跳转到对应的界面, 包括自定义的界面.所以这个raw目录是自定义Google SetupWizard的核心, 无论是添加还是修改页面,都很可能会涉及到
在移植了这个raw目录之后需要修改一下包名
1. 进入res/raw目录
2. 执行`sed -i "s/com.google.android.gmsintegration/com.setupwizard.demo/g" `grep com.google.android.gmsintegration -rl .``
3. 执行`ag com.setupwizard.demo`看是否与AndroidManifest.xml中的包名是否保持一致即可

# 添加启动时的uri
定制SetupWizard需要指定开机时的启动的流程, 否则会用Google SetupWizard.apk中的xml/目录下的流程
```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <string name="wizard_script_uri" translatable="false">android.resource://com.setupwizard.demo/raw/wizard_script</string>
</resources>
```
# 总结
以上步骤之后就可以编译自定义的SetupWizard然后push进去启动了, 这样得到的仍然是原生Android的页面,如何自定义SetupWizard留待后续文章, **拷贝代码的时候要注意修改包名**
