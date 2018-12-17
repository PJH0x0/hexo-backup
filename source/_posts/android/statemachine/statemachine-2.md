---
title: Android状态机模式解析(二)
date: 2018/12/16
tags:
  - android
  - stateMachine
categories:
  - android
abbrlink: 735aa624
---
> 前言：在[Android状态机模式解析(一)](http://www.godteen.com/posts/2262efea/#more)中讨论了状态机的初始化以及启动，本篇将要解析状态机如何处理消息以及状态的切换

# StateMachine处理消息
## 发送消息
```Java
//MainActivity.java
helloState.sendMessage(helloState.obtainMessage(HelloState.CMD_1));
//StateMachine.java
public final void sendMessage(Message msg) {
    // mSmHandler can be null if the state machine has quit.
    SmHandler smh = mSmHandler;
    if (smh == null) return;

    smh.sendMessage(msg);
}
```
发送消息的过程比较简单，就是直接向mSmHandler发送消息

## 处理消息
```Java
//StateMachine$SmHandler
@Override
public final void handleMessage(Message msg) {
    if (!mHasQuit) {
        //mSm的类型是StateMachine,这个专门是给子StateMachine处理的
        if (mSm != null && msg.what != SM_INIT_CMD && msg.what != SM_QUIT_CMD) {
            mSm.onPreHandleMessage(msg);
        }

        if (mDbg) mSm.log("handleMessage: E msg.what=" + msg.what);

        /** Save the current message */
        mMsg = msg;

        /** State that processed the message */
        State msgProcessedState = null;
        //初始化完成之后，条件成立
        if (mIsConstructionCompleted) {
            /** Normal path */
            //1. 处理消息
            msgProcessedState = processMsg(msg);
        } else if (!mIsConstructionCompleted && (mMsg.what == SM_INIT_CMD)
                && (mMsg.obj == mSmHandlerObj)) {
            /** Initial one time path. */
            mIsConstructionCompleted = true;
            invokeEnterMethods(0);
        } else {
            throw new RuntimeException("StateMachine.handleMessage: "
                    + "The start method not called, received msg: " + msg);
        }
        //2. 处理完消息之后切换状态
        performTransitions(msgProcessedState, msg);

        // We need to check if mSm == null here as we could be quitting.
        if (mDbg && mSm != null) mSm.log("handleMessage: X");
        //给子StateMachine处理
        if (mSm != null && msg.what != SM_INIT_CMD && msg.what != SM_QUIT_CMD) {
            mSm.onPostHandleMessage(msg);
        }
    }
}

private final State processMsg(Message msg) {
    //3. 获取栈顶的State，如果是第一次发送消息，则是InitialState
    StateInfo curStateInfo = mStateStack[mStateStackTopIndex];
    if (mDbg) {
        mSm.log("processMsg: " + curStateInfo.state.getName());
    }
    //暂时不用管
    if (isQuit(msg)) {
        transitionTo(mQuittingState);
    } else {
        //4. 从栈顶节点State到root节点State依次处理，如果有节点State处理了该消息返回true，则退出循环
        while (!curStateInfo.state.processMessage(msg)) {
            /**
             * Not processed
             */
            curStateInfo = curStateInfo.parentStateInfo;
            if (curStateInfo == null) {
                /**
                 * No parents left so it's not handled
                 */
                //如果root节点依然没有处理，则丢弃到子StateMachine处理或者不处理
                mSm.unhandledMessage(msg);
                break;
            }
            if (mDbg) {
                mSm.log("processMsg: " + curStateInfo.state.getName());
            }
        }
    }
    //5. 若有节点处理，则返回该节点State，若root节点还没处理，则返回null
    return (curStateInfo != null) ? curStateInfo.state : null;
}
```
从注释1、3、4、5的流程可以看出，如果某个节点State已经处理完毕该消息那么消息就不会往root节点State方向传递。
值得注意的一点是`mStateStack[mStateStackTopIndex]`取的不一定是叶子节点State，因为任何节点都可以作为栈顶节点State
接下来我们讨论状态切换

# StateMachine状态切换
由于StateMachine状态切换比较复杂，所以与StateMachine消息处理分开讨论

## 设置目标状态
以HelloState的S1状态为例
```Java
//HelloState.java
class S1 extends State {
    @Override
    public void enter() {
        log("mS1.enter");
    }
    @Override
    public boolean processMessage(Message message) {
        log("S1.processMessage what=" + message.what);
        if (message.what == CMD_1) {
            // Transition to ourself to show that enter/exit is called
            transitionTo(mS1);
            //IState.HANDLED为true
            return IState.HANDLED;
        } else {
            // Let parent process all other messages
            //IState.NOT_HANDLED为false
            return IState.NOT_HANDLED;
        }
    }
    @Override
    public void exit() {
        log("mS1.exit");
    }
}
protected final void transitionTo(IState destState) {
    mSmHandler.transitionTo(destState);
}
private final void transitionTo(IState destState) {
    mDestState = (State) destState;
    if (mDbg) mSm.log("transitionTo: destState=" + mDestState.getName());
}
```
从上面的代码可以看出transitionTo这个方法并没有切换状态，而只是设置了一个目标State,真正的状态切换是在`return IState.HANDLED`之后。
## 切换状态
延续之前的处理消息的代码
```Java
//StateMachine$SmHandler
@Override
public final void handleMessage(Message msg) {
    //省略其他代码
    //2. 处理完消息之后切换状态
    //msgProcessedState这个参数没啥用
    performTransitions(msgProcessedState, msg);
}
private void performTransitions(State msgProcessedState, Message msg) {
    State orgState = mStateStack[mStateStackTopIndex].state;

    State destState = mDestState;
    if (destState != null) {
        while (true) {

            //1. 获取orgState和destState的共同父节点State,如果没有则commonStateInfo则为null
            StateInfo commonStateInfo = setupTempStateStackWithStatesToEnter(destState);
            //2. 从栈顶State到commonStateInfo的State依次调用exit()方法
            invokeExitMethods(commonStateInfo);
            //3. 建立新的StateStack，也是反转mTempStateStack
            int stateStackEnteringIndex = moveTempStateStackToStateStack();
            //4. 调用新栈的enter方法,注意
            invokeEnterMethods(stateStackEnteringIndex);

            //deferMessage机制，后续再讨论
            moveDeferredMessageAtFrontOfQueue();
            //如果在State的enter()或是exit()方法调用了transitionTo()
            //要继续切换到目标状态
            if (destState != mDestState) {
                // A new mDestState so continue looping
                destState = mDestState;
            } else {
                // No change in mDestState so we're done
                break;
            }
        }
        mDestState = null;
    }

    if (destState != null) {
        if (destState == mQuittingState) {
            mSm.onQuitting();
            cleanupAfterQuitting();
        } else if (destState == mHaltingState) {
            mSm.onHalting();
        }
    }
}
private final StateInfo setupTempStateStackWithStatesToEnter(State destState) {
    //5. 清空mTempStateStack
    mTempStateStackCount = 0;
    StateInfo curStateInfo = mStateInfo.get(destState);
    //6. 建立新的mTemStateStack, 结束之后mTempStateStack == mTempStateStack.length
    //注意：此时mTempStateStack中没有commonStateInfo
    do {
        mTempStateStack[mTempStateStackCount++] = curStateInfo;
        curStateInfo = curStateInfo.parentStateInfo;
    } while ((curStateInfo != null) && !curStateInfo.active);

    //7. 返回commonStateInfo,可能为null
    return curStateInfo;
}

private final void invokeExitMethods(StateInfo commonStateInfo) {
    //8. 从栈顶State到commStateInfo的State依次调用exit()方法,
    //如果commonStateInfo不为null,即orgState和destState有公共父节点State, 则
    //结束后mStateStackTopIndex指向commonStateInfo，并且commonStateInfo.State也不会调用exit()
    //反之，如果commStateInfo=null, 则mStateStackTopIndex=-1
    while ((mStateStackTopIndex >= 0)
            && (mStateStack[mStateStackTopIndex] != commonStateInfo)) {
        State curState = mStateStack[mStateStackTopIndex].state;
        if (mDbg) mSm.log("invokeExitMethods: " + curState.getName());
        curState.exit();
        mStateStack[mStateStackTopIndex].active = false;
        mStateStackTopIndex -= 1;
    }
}

//这个方法我们在初始化也讲过，如果commonStateInfo==null, 那么和初始化是一样的方式
//如果commonStateInfo!=null, 那么就不一样
//注释中我们只讨论commonStateInfo!=null的情况
private final int moveTempStateStackToStateStack() {
    //1. mStateStackTopIndex指向的是commonStateInfo, 
    //commonStateInfo.active=true
    int startingIndex = mStateStackTopIndex + 1;
    //2. mTempStateStack只存储了从commonStateInfo到destState所在的StateInfo
    //并且不包括commStateInfo
    int i = mTempStateStackCount - 1;
    int j = startingIndex;
    //3. 反转mTempStateStack的结构
    while (i >= 0) {
        if (mDbg) mSm.log("moveTempStackToStateStack: i=" + i + ",j=" + j);
        mStateStack[j] = mTempStateStack[i];
        j += 1;
        i -= 1;
    }
    //4. 此时mStateStackTopIndex指向了destState所在的StateInfo
    mStateStackTopIndex = j - 1;
    if (mDbg) {
        mSm.log("moveTempStackToStateStack: X mStateStackTop=" + mStateStackTopIndex
                + ",startingIndex=" + startingIndex + ",Top="
                + mStateStack[mStateStackTopIndex].state.getName());
    }
    return startingIndex;
}
```
上面的代码比较多我去除了一些log和注释，这段的逻辑其实倒是还好，但是细节的地方比较多。
这里主要分为两种情况：
1. orgState和destState拥有共同的commonState, 那么需要注意三点:
* State.enter()和State.exit()的调用细节
* mTempStateStack清空、新建以及mStateStack的建立
* mStateStackTopIndex指向哪个StateInfo
2. orgState和destState没有共同的commonState,即commonStateInfo==null。这种情况比较简单，即源栈的State全部退出，然后建立新的栈。建立新的栈就和初始化的流程是一样的

## deferMessage()机制
这种机制通常用于一个消息要通过多个State进行处理的情况，而这些State不在同一个mStateStack栈中。以HelloState为例，`CMD_2`、`CMD_3`都是这种。
```Java
//HelloState.java
class P1 extends State {
    @Override
    public void enter() {
        log("mP1.enter");
    }
    @Override
    public boolean processMessage(Message message) {
        boolean retVal;
        log("mP1.processMessage what=" + message.what);
        switch(message.what) {
            case CMD_2:
                // CMD_2 will arrive in mS2 before CMD_3
                //这里利用了handler的机制，此时CMD_3并不会立即发送出去
                sendMessage(obtainMessage(CMD_3));
                //1. 会将CMD2延迟到切换状态后发送，并且是在MessageQueue的最前面
                deferMessage(message);
                //切换状态
                transitionTo(mS2);
                retVal = IState.HANDLED;
                break;
            default:
                // Any message we don't understand in this state invokes unhandledMessage
                retVal = IState.NOT_HANDLED;
                break;
        }
        return retVal;
    }
    @Override
    public void exit() {
        log("mP1.exit");
    }
}
//StateMachine.java
protected final void deferMessage(Message msg) {
    mSmHandler.deferMessage(msg);
}
//StateMachine$SmHandler
private final void deferMessage(Message msg) {
    if (mDbg) mSm.log("deferMessage: msg=" + msg.what);

    /* Copy the "msg" to "newMsg" as "msg" will be recycled */
    Message newMsg = obtainMessage();
    newMsg.copyFrom(msg);
    //mDeferredMessages就是一个ArrayList
    mDeferredMessages.add(newMsg);
}
```
在执行完`return retVal;`之后，此时还在`handleMessage()`里面,所以`CMD_3`并没有发送出去，接下来我们重新讨论一下`performTransition()`这个方法。
```Java
//StateMachine$SmHandler
private void performTransitions(State msgProcessedState, Message msg) {
    State orgState = mStateStack[mStateStackTopIndex].state;

    State destState = mDestState;
    if (destState != null) {
        //其他线程有可能又重新设置了新的mDestState
        while (true) {

            StateInfo commonStateInfo = setupTempStateStackWithStatesToEnter(destState);
            invokeExitMethods(commonStateInfo);
            int stateStackEnteringIndex = moveTempStateStackToStateStack();
            invokeEnterMethods(stateStackEnteringIndex);

            //1. 此时已经切换到destState了，以上面的例子，destState已经变为mS2
            moveDeferredMessageAtFrontOfQueue();
            //如果在State的enter()或是exit()方法调用了transitionTo()
            //要继续切换到目标状态
            if (destState != mDestState) {
                // A new mDestState so continue looping
                destState = mDestState;
            } else {
                // No change in mDestState so we're done
                break;
            }
        }
        mDestState = null;
    }
}
private final void moveDeferredMessageAtFrontOfQueue() {
    //2. 发送所有的延迟消息
    for (int i = mDeferredMessages.size() - 1; i >= 0; i--) {
        Message curMsg = mDeferredMessages.get(i);
        if (mDbg) mSm.log("moveDeferredMessageAtFrontOfQueue; what=" + curMsg.what);
        sendMessageAtFrontOfQueue(curMsg);
    }
    mDeferredMessages.clear();
}
```
执行完`performTransitions()`之后才真正的将`CMD_2`、`CMD_3`发送出去了，有两点需要注意：
1. 处理CMD_2和CMD_3的是mS2
2. CMD_2比CMD_3到的要早，因为`sendMessageAtFrontOfQueue(curMsg);`会将CMD_2移到MessageQueue的头部发送。

官方给予的例子比较复杂，在这里的就不讨论了，有兴趣可以去看HelloState.java的注释
# 总结
以上就是就是Android状态机的原理了，如果有问题的话可以发送邮件到godteen.peng@gmail.com和我一起讨论


