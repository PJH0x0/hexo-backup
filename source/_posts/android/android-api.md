---
title: Android API整理
date: 2019/04/29
tags:
  - android
  - api
categories: android
abbrlink: 2527fbe6
---

# Android tips
1. Android数据库, 6.0在data/data/PackageName下，7.1以后在data/user_de/0/PackageName下
2. Android如果定义了EditText的id，当应用退出的时候可以保存状态
3. 当RadioButton多次置为true的时候，onCheckChanged只会执行一次
4. Android global、system、secure数据库的位置,7.1在/data/users/0目录下, 8.0以后在/data/system/users/0目录下

# Code fragment

### Android ScrollView不能全屏
```xml
android:fillViewport="true"
```
### Andriod英文大写
```xml
<!--首字母大写，其他小写-->
android:textAllCaps="false"
```
### Android单位转换
```java
//dp转px
float scale = getResources().getDisplayMetrics().density;
return (int) (dp * scale + 0.5f);

//px转dp
float scale = getResources().getDisplayMetrics().density;
return px / scale + 0.5f;

//sp转px
float scale = getResources().getDisplayMetrics().scaledDensity;
return (int) (sp * scale + 0.5f);

//px转sp
float scale = getResources().getDisplayMetrics().scaledDensity;
return (px / scale + 0.5f);

```

### 打印方法栈的log

```java
//第一种方法
Log.i("TAG", "stacktrace" + Log.getStackTraceString(new Throwable()));
//第二种方法
Exception e = new Exception("TAG");
e.printStackTrace();
```

## 获取虚拟键导航栏高度
```java
int resourceId = getContext().getResources().getIdentifier("navigation_bar_height", "dimen", "android");
    
int height = getContext().getResources().getDimensionPixelSize(resourceId);
```

### 获取StatusBar高度
将上面的"navigation_bar_height"改成"status_bar_height"即可

### 获取控件的绝对位置

```java
//获取控件在屏幕中的位置，location[0]是X值，location[1]是Y值
int[] location = new int[2];
getLocationOnScreen(location);
```

### 第三方APK跳转
**Tips:**这种方法不能设置`android:exported="false"`属性
```java
//包名和类名不一定相同，以AndroidManifest.xml里面的packgeName以及Activity标签中的类名为准
ComponentName componentName = new ComponentName("com.android.settings","com.android.settings.wifi.WifiSetupActivity");
Intent intent = new Intent(Intent.ACTION_MAIN);
intent.addCategory(Intent.CATEGORY_DEFAULT);
intent.setComponent(componentName);
startActivity(intent);
```
# Android define

### 关于ViewStub和Merge标签
ViewStub是用来在当前布局包含另一个布局，有点像是include标签，只是只可以在需要的时候进行加载<br>
Merge标签是用来延伸上一个inlude标签的布局，减少布局层次，从而达到优化布局的效果

### 关于Activity的转场动画
Activity的转场动画有两个，分别是源Activity的退场动画和目的Activity的进场动画

### Android过度绘制定义
过度绘制（Overdraw）描述的是屏幕上的某个像素在同一帧的时间内被绘制了多次。在多层次重叠的 UI 结构里面，如果不可见的 UI 也在做绘制的操作，会导致某些像素区域被绘制了多次，同时也会浪费大量的 CPU 以及 GPU 资源

### Java反射机制定义
反射机制就是在运行状态中，对于任意一个类，都能够知道这个类的所有属性和方法；对于任意一个对象，都能够调用它的任意一个方法和属性；这种动态获取的信息以及动态调用对象的方法的功能称为java语言的反射机制
