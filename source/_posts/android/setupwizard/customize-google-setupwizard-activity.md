---
title: Android Q Google SetupWizard新增页面
abbrlink: 9daebc3e
---
>前言:本文基于Android Q进行的实现,其他版本并不能成功, 因为只有Android Q才有setupcompat和setupdesign库
在Google Setup Wizard中新增页面需要先集成自定义开机向导, 详情请看[Android集成自定义Google Setup Wizard](https://juejin.im/post/5df83101e51d45584476eff9), Android Q新增页面的具体步骤:
1. 定义layout布局,需要使用GlifLayout或者PartenerCustomizationLayout作为根布局
2. 定义Activity文件
3. 在AndroidManifest.xml增加Activity配置, 并配置对应的action
4. 在wizar_script.xml中配置对应的uri, 推荐是oem_post_setup之后添加
以下是具体的代码,相对简单
# layout布局
```xml
<!--activity_demo.xml-->
<?xml version="1.0" encoding="utf-8"?>

<com.google.android.setupdesign.GlifLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/setup_wizard_layout"
    app:sucHeaderText="@string/demo_activity_title"
    android:icon="@mipmap/ic_launcher_round"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:gravity="center_horizontal"
        android:orientation="vertical">
        <TextView
            android:id="@+id/demo_description"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="@string/demo_activity_content"
            android:textColor="#DD000000"
            android:textSize="18dp"/>

    </LinearLayout>

</com.google.android.setupdesign.GlifLayout>
```
GlifLayout是setupdesign库中的自定义布局, 继承自setupcompat库的PartnerCustomizationLayout, Android Q之后GMS要求根布局必须使用PartnerCustomizationLayout或者继承自PartnerCustomizationLayout的布局

# Activity
```java
public class DemoActivity extends Activity {
    public static final String TAG = "DemoActivity";

    private FooterButton mNextButton;    
    private FooterBarMixin mFooterBarMixin;    
    private GlifLayout mViewRoot;
    @Override
    protected void onCreate(Bundle onSavedState) {
        super.onCreate(onSavedState);
        setContentView(R.layout.activity_demo);
        mViewRoot = findViewById(R.id.setup_wizard_layout);
        mFooterBarMixin = mViewRoot.getMixin(FooterBarMixin.class);
        mNextButton = new FooterButton.Builder(this)
            .setListener(this::onNextClick)
            .setButtonType(ButtonType.NEXT)
            .setTheme(R.style.SudGlifButton_Primary)
            .setText("Next")
            .build();
        mFooterBarMixin.setPrimaryButton(mNextButton);
    }

    private void onNextClick(View view) {
        Log.d(TAG, "On next button click");
    }
}
```
在Android Q中, Google使用FooterBarMixin来控制前进后退的跳转, 这个也GMS所推荐的.

# AndroidManifest
```xml
    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:theme="@style/sudLayoutTheme"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true">
        <receiver android:name=".SuwCustomizationReceiver">
            <intent-filter>
                <action android:name="com.android.setupwizard.action.PARTNER_CUSTOMIZATION" />
            </intent-filter>
        </receiver>
        <activity android:name=".DemoActivity">
            <intent-filter>
                <action android:name="com.setupwizard.demo.NEW_DEMO" />
                <category android:name="android.intent.category.DEFAULT" />
            </intent-filter>
        </activity>

    </application>
```
这里主要关注的theme, theme需要使用Google指定的theme, Google给了两种theme分别是Light Theme和Heavy Theme, 只有GlifLayout才支持Heavy Theme, 并且使用GlifLayout的时候theme必须是继承自SudThemeGlif或者SudThemeGlif.Light, 否则可能会报错

以下是我使用的theme
```xml
    <style name="sudLayoutTheme" parent="SudThemeGlif.Light">
        <item name="android:screenOrientation">portrait</item>
        <item name="sudUsePartnerHeavyTheme">true</item>
        <item name="android:navigationBarDividerColor">#e0e0e0</item>
        <item name="android:listDivider">@*android:drawable/list_divider_material</item>
    </style>
```
# wizard_script
最后将对应的uri加入到wizard_script.xml中即可生效
```xml
    <WizardAction id="demo" wizard:uri="intent:#Intent;action=com.setupwizard.demo.NEW_DEMO;end"/>
```
# 效果
最后实现的效果图
