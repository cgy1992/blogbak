title: Android基础-Handler源码
date: 2018-12-12 00:00:00
categories :
- 源码解析
tags:
- android
---
最近深感对基础知识不够扎实, 所以回头看一下`Handler`的源码. 首先, 我们先从他的使用作为入口来
<!--more-->
## Handler的使用
先看下, 几种通过Handler发送消息的用法, 一共有四种, 当然最终调用的还是`Handler#sendMessageDelayed(Message msg, long delayMillis)`, 这个我们放在后面去解析.
``` java
private fun sendHandler(){
        // 1
        MyHandler().obtainMessage(1).sendToTarget()
        // 2
        val msg = Message()
        msg.what = 2
        MyHandler().sendMessage(msg)
        // 3
        MyHandler().post { Log.e(MainActivity::class.java.canonicalName, "3") }
        // 4
        val callback = Handler.Callback {
            Log.e(MainActivity::class.java.canonicalName, it.what.toString())
            true
        }
        val msg2 = Message()
        msg2.what=4
        MyHandler(callback).sendMessage(msg2)
    }

```
然后是他通用的消息接收处理
``` java
// kotlin内部类默认为静态类
private class MyHandler: Handler {
        constructor()
        constructor(callback: Callback)

        override fun handleMessage(msg: Message?) {
            super.handleMessage(msg)
            Log.e(MainActivity::class.java.canonicalName, msg?.what.toString())
        }
    }
```
## Handler构造
要了解`Handler`, 我们首先看他的构造函数, `Handler`的构造, 实际分为两种, 一种是传入`Looper`, 一种是直接使用当前线程的`Looper`, 可以看到, 主要就是通过默认当前线程Looper或者是提供的Looper获取`messageQueue`对象.
``` java

public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        // 使用当前线程的Looper对象
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        // 获取looper持有的messageQueue
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }

// 使用提供的Looper对象
public Handler(Looper looper, Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```
当我们在子线程内直接发送消息, 会抛出异常, 具体我们可以从`Looper#myLooper()`开始看
``` java
/**
     * Return the Looper object associated with the current thread.  Returns
     * null if the calling thread is not associated with a Looper.
     */
public static @Nullable Looper myLooper() {
       return sThreadLocal.get();
   }
```
`ThreadLocal`这里不细说, 反正知道他是每个线程独有的存储空间就行了.可以看到, looper是每个线程不共享的内存对象, 它存储在ThreadLocal内, 而在`Looper#prepare`会进行Looper的实例化, 和存储在threadLocal内
``` java
private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```
## 发送消息
Handler发送消息最终都会走到`Handler#sendMessageDelayed`方法, 会发现他最终调用的是`MessageQueue#enqueueMessage(Message msg, long when)`
``` java
public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }

public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }

private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```
`MessageQueue`虽然说是消息队列, 但是实际实现队列机制是依靠的`Message`的数据结构,每一个`Message`都会通过它内部的`next`字段指向另外一个`Message`, 最终实现了单向列表的数据结构, 而`MessageQueue`内部的`mMessages`就是队列头的消息对象, 具体可以看下, 插入消息时候的方法体
``` java
boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuittying) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
            // 延迟时间
            msg.when = when;
            // 队列中上一条消息
            Message p = mMessages;
            boolean needWake;
            // 无延迟 或者 是第一条消息// 或者需要比上一条消息早发送
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                // 将当前消息置于列表顶
                // 当前消息的下一条消息指向之前的mMessages
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                // 中间插入当前的消息对象
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            // 线程唤醒
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
```
## 消息处理
关于消息的处理, 我们又要回头去看`Looper`, 关于`MessageQueue`它主要做了两个事情, 从`Handler`那边接收`Message`做队列托管, 然后将符合条件的`Message`返回给`Looper`, 具体我们可以看下`Looper#loop()`
``` java
public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        // 校验当前线程在当前进程内
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
          // 1. 获取message
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // 2. handler的分发处理
            msg.target.dispatchMessage(msg);
            ...

            // 3. 消息的回收
            msg.recycleUnchecked();
        }
    }
```
删除了一些无关的代码, 我们看下核心流程, `Looper#loop`就一直不断循环的在做三件事:
1. 拿取`MessageQueue`中的`Message`
2. 调用`Message`持有的handler的分发处理事件
3. 回收已处理掉的`Message`

我们来看下`MessageQueue`给到`Looper`怎么样的`Message`
``` java
Message next() {
        // Return here if the message loop has already quit and been disposed.
        // This can happen if the application tries to restart a looper after quit
        // which is not supported.
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }

        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                // 获取当前启动时间
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                // 当头部消息的target(携带handler)为空的情况下
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    // 遍历队列, 直到获取到的消息不为空并且是同步信息
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    // 延迟消息
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        // 指向下一条message
                        // 该条消息需要被执行的时候
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();
                        // 返回当前的message
                        return msg;
                    }
                } else {
                    // No more messages.
                    // 如果当前队列头部没有消息, 则说明队列为空
                    nextPollTimeoutMillis = -1;
                }

                // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {
                    dispose();
                    return null;
                }

                // If first time idle, then get the number of idlers to run.
                // Idle handles only run if the queue is empty or if the first message
                // in the queue (possibly a barrier) is due to be handled in the future.
                // 当队列空闲(头部消息为空或者头部消息还未到时候执行), 并且没有闲置handler的时候
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    // 当没有闲置handler需要执行的时候
                    mBlocked = true;
                    continue;
                }

                // 如果闲置handlers数组为空
                if (mPendingIdleHandlers == null) {
                    // 初始化, 闲置的handlers最大数为4
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            // Run the idle handlers.
            // We only ever reach this code block during the first iteration.
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                // 引用释放, 则这个循环只会执行mPendingIdleHandlers集合的第一个handler元素
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    // 执行空闲回调处理
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }

                // 假设空闲回调处理不保留这个闲置处理, 则移除对应的idleHandler
                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // Reset the idle handler count to 0 so we do not run them again.
            // 重置
            pendingIdleHandlerCount = 0;

            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            // 重置
            nextPollTimeoutMillis = 0;
        }
    }
```
相关的代码注释差不多都在上面了, 我们主要整理下, 这里在做的事情
1. 获取队列头部消息, 如果头部消息存在, 并且当前消息持有handler为空的情况下, 则去取下一个非空的同步消息
2. 当该条消息是当前正需要执行的时候(具体看`Message#when`的时间戳), 则返回这条消息, 否则继续查找下一条消息
3. 如果当前没有消息, 或者当前时间没有需要执行的消息, 则触发idleHandler的处理.
4. 然后在下个时间戳继续找消息

回头看`Handler#dispatchMessage(Message msg)`, 可以了解到handler处理事件的优先级
``` java
public void dispatchMessage(Message msg) {
        // msg.callback通过handler.post传入的runnable
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
              // mCallback是通过Handler的构造函数传入
                 if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            // 最后才是Message传入的处理, 一般是继承Handler, 重写handleMessage方法
            handleMessage(msg);
        }
    }
```
## 关于Message的重复利用
主流程基本分析完毕, 我们现在再回头看使用API, 一般的业务场景, 我们用到比较多的可能是自己新建包装一个`Message`, 通过Handler来发送. 那么`Handler#obtainMessage()`做了什么呢, 为什么通过这个自己可以不用再新建一个Message了. 我们看API调取, 会发现他内部调用到了`Message#obtain()`, 而最终message对象是从这里获取到.
``` java
public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }
```
可以看到, 当`sPool`对象不为空的时候, 是复用了这个对象的内存空间.那么`sPool`又是什么时候被赋值的呢
``` java
void recycleUnchecked() {
        // Mark the message as in use while it remains in the recycled object pool.
        // Clear out all other details.
        flags = FLAG_IN_USE;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = -1;
        when = 0;
        target = null;
        callback = null;
        data = null;

        synchronized (sPoolSync) {
          // MAX_POOL_SIZE == 50
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }
```
这个方法是不是有点眼熟, 就是前面分析`Looper#loop()`时候, 当消息处理完以后, 消息回收调用到的, 可以看到sPool作为需要被回收的对象, 当消息处理完后, 当前消息进入当前回收队列中, 这样可以达到`Message`的复用作用
