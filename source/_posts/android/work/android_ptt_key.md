---
title: v500可编程键技术总结
date: 2019/12/13
tags: '-work -android'
categories: '-work -android'
abbrlink: e8c74e8
---

# 实现长按功能
当有对应的按键事件发生的时候,会传递过来一个KeyEvent对象, 其中getAction()方法可以获取到是DOWN,UP,CANCEL,MOVE等事件, 这里判断是否是down, 另外就是长按功能longPress功能则由getRepeatCount()提供.
Android认为是超过0次就是长按事件, 我则设置的是3次
```Java
final boolean down = event.getAction() == KeyEvent.ACTION_DOWN;
boolean longPressed = repeatCount > 2; 
if (down && repeatCount == 3) {
    //handleLongPress
    keyTpe = 1//Long pressed
}
```
这里使用repeatCount为3是因为长按会多次的触发这个方法, 导致多次启动应用
# 实现双击功能
主要是通过`KeyEvent.getEventTime()`来实现, 主要是两次短按UP事件时间间隔差距少于某个阈值就可以认为是双击事件
```Java
if(!down && !longPressed) {
    if (isDoubleTap(event)) {
        keyType = 2;//Double press
    } else {
        mProgrammablePreviousKeyEvent = KeyEvent.obtain(event);
    }
}
private boolean isDoubleTap(KeyEvent currentEvent) {
    if (null != mProgrammablePreviousKeyEvent) {
        Log.d(TAG, "mProgrammablePreviousKeyEvent>>" + mProgrammablePreviousKeyEvent + "; CurrentEvent>>" + currentEvent);
        long time = currentEvent.getEventTime() - mProgrammablePreviousKeyEvent.getEventTime();
        Log.d(TAG, "Double tap time>>" + time);
        Log.d(TAG, "Timeout>>" + ViewConfiguration.getMultiPressTimeout());
        if (time < ViewConfiguration.getMultiPressTimeout()) {
            return true;  
        }
    }
    return false;
}
```
要注意:**传过来的KeyEvent不能直接赋值, 因为这个是复用的, 直接赋值第二次传过来会发现是同一个对象, 需要使用obain()方法重新创建一个新的KeyEvent对象.**
