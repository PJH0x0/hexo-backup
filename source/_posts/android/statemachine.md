---
title: Android状态机模式解析
date: 2018/12/13
tags:
  - android
  - stateMachine
categories:
  - android
---
>前言:Android状态机模式在frameworks层用的还是比较广泛的,主要的是Wifi,Bluetooth,Network这几块用的比较多, 本文基于我自己改造的官方Demo,详情可参照[HelloState](https://github.com/TeenagerPeng/HelloState). 阅读本文需要的知识: Android的Handler使用, Java基础, 数据结构基础

# StateMachine类结构
![StateMachie类图](statemachine-class.png)

# StateMachine初始化
## 时序图
如图所示,是StateMachine初始化的时序图
![状态机初始化](initial-sequence.png)
## StateMachine创建
StateMachine的构造方法是`protected`的访问权限,所以一般要有一个子类去继承自它.
```Java
//MainActivity.java
HelloState helloState = new HelloState("Hello State");
//HelloState.java
public HelloState(String name) {
    super(name);
    log("ctor E");

    // Add states, use indentation to show hierarchy
    addState(mP1);
    addState(mS1, mP1);
    addState(mS2, mP1);
    addState(mP2);

    // Set the initial state
    setInitialState(mS1);
    log("ctor X");
}
//StateMachine.java
protected StateMachine(String name) {
    mSmThread = new HandlerThread(name);
    mSmThread.start();
    Looper looper = mSmThread.getLooper();
    //创建SmHandler
    initStateMachine(name, looper);
}
private void initStateMachine(String name, Looper looper) {
    mName = name;
    mSmHandler = new SmHandler(looper, this);
}
```
当然StateMachine也提供了让调用者自行设置Looper的构造方法,这里不再详细讨论.
创建SmHandler之后就需要添加状态,这里设计了4种状态.
```Java
//StateMachine.java
protected final void addState(State state) {
    mSmHandler.addState(state, null);
}
protected final void addState(State state, State parent) {
    mSmHandler.addState(state, parent);
}
//StateMachine$SmHandler
private final StateInfo addState(State state, State parent) {
    if (mDbg) {
        mSm.log("addStateInternal: E state=" + state.getName() + ",parent="
                + ((parent == null) ? "" : parent.getName()));
    }
    StateInfo parentStateInfo = null;
    if (parent != null) {
        //mStateInfo是HashMap<State, StateInfo>类型
        parentStateInfo = mStateInfo.get(parent);
        //1. 如果parentState不为空且没有添加到mStateInfo里面,则设置parentState为root节点
        if (parentStateInfo == null) {
            // Recursively add our parent as it's not been added yet.
            parentStateInfo = addState(parent, null);
        }
    }
    StateInfo stateInfo = mStateInfo.get(state);
    if (stateInfo == null) {
        stateInfo = new StateInfo();
        mStateInfo.put(state, stateInfo);
    }

    // Validate that we aren't adding the same state in two different hierarchies.
    //2. 是否添加了两个不同树结构下的State
    if ((stateInfo.parentStateInfo != null)
            && (stateInfo.parentStateInfo != parentStateInfo)) {
        throw new RuntimeException("state already added");
    }
    //3. 初始化StateInfo
    stateInfo.state = state;
    stateInfo.parentStateInfo = parentStateInfo;
    stateInfo.active = false;
    if (mDbg) mSm.log("addStateInternal: X stateInfo: " + stateInfo);
    return stateInfo;
}
```
StateMachine采用了一种树状的层次结构,但是可以有多个root节点,如注释1阐述了两种添加root节点的方式.
一种是`addState(mP1)`这种方式直接添加mP1作为root节点;另一种是```addState(mS3, mP3)```方式可以添加新的root节点mP3.
注释2是不能在不同root节点添加相同的State,这就造成了我一个疑惑,那为何要在注释3中初始化StateInfo呢?似乎直接在`stateInfo = new StateInfo()`下面初始化应该更好,也更加直观
最后是设置初始状态`setInitialState(mS1);`
```Java
//StateMachine.java
protected final void setInitialState(State initialState) {
    mSmHandler.setInitialState(initialState);
}
//StateMachine$SmHandler
private final void setInitialState(State initialState) {
    if (mDbg) mSm.log("setInitialState: initialState=" + initialState.getName());
    mInitialState = initialState;
}
```
这段代码就没啥好分析的,记住`mInitialState`就可以了,启动状态机的时候会用到

## 启动StateMachine
启动StateMachine相当于就是设置初始化的状态,然后等待消息的发送
```Java
//MainActivity.java
helloState.start();

//StateMachine.java

```
