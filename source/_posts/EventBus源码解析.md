title: EventBus源码解析
date: 2018-03-30 00:00:00
categories:
- 源码解析
tags:
- android
- 源码解析
---
### 前言
前期加班加点赶项目, 趁着刚上线空两天,赶紧看下`EventBus`做个"思维复健"
### 使用
`EventBus`的使用非常简单, 如果使用默认的`EventBus`, 我们一般只会使用到以下三个API
1. 绑定
``` java
EventBus.getDefault().regisiter(this);
```
2. 发送信息
``` java
EventBus.getDefault().post(new Event());
```
3. 解绑
``` java
EventBus.getDefault().unregisiter(this);
```
<!-- more -->
### EventBus.getDefault()
`EventBus`内部维护了一个单例, 通过`getDefault`我们可以获取默认单例来进行绑定和发送动作, 但是当我们需要进行一些关于log, 是否未有订阅者情况的响应处理时, 我们可以通过`EventBusBuilder`通过构建者模式来进行配置处理,本篇解析仅分析默认情况下的流程代码
### 绑定
老规矩, 我们先上代码
``` java
public void register(Object subscriber) {
        Class<?> subscriberClass = subscriber.getClass();
        // 获取对应subscriber类的订阅方法
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            // 遍历执行订阅
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
        }
    }
```
我们根据传递的订阅者来获取相关的订阅方法, 然后遍历执行订阅的动作. 我们首先看下如果查找订阅者的所有订阅方法
``` java
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        // 从缓存中查找订阅方法
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
        // 缓存中有, 直接返回
        if (subscriberMethods != null) {
            return subscriberMethods;
        }
        // 查找注册方法, 默认false
        if (ignoreGeneratedIndex) {
            // 使用反射查找
            subscriberMethods = findUsingReflection(subscriberClass);
        } else {
            // 使用注解器生成的类查找
            subscriberMethods = findUsingInfo(subscriberClass);
        }
        // 如果没有订阅方法, 则抛出异常
        if (subscriberMethods.isEmpty()) {
            throw new EventBusException("Subscriber " + subscriberClass
                    + " and its super classes have no public methods with the @Subscribe annotation");
        } else {
            // 否则加入缓存中
            METHOD_CACHE.put(subscriberClass, subscriberMethods);
            return subscriberMethods;
        }
    }
```
我们知道`EventBus`3.0版本后通过`@Subscribe`注解来标注对应的订阅方法, 可以看到通过`findUsingInfo`方法查询订阅方法, 如果没有订阅方法, 会抛出异常, 而如果找到了, 则会加入缓存`METHOD_CACHE`进行内部维护, 这个方法可以优化部分性能, 减少反射带来的性能问题.
我们在往`findUsingInfo`里看, 会发现如果找不到相关订阅者信息的时候, 仍会通过反射来寻找.
``` java
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            findState.subscriberInfo = getSubscriberInfo(findState);
            if (findState.subscriberInfo != null) {
                SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
                // 遍历订阅者方法
                for (SubscriberMethod subscriberMethod : array) {
                    if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                        findState.subscriberMethods.add(subscriberMethod);
                    }
                }
            } else {
                // 没有订阅信息, 从反射来找
                findUsingReflectionInSingleClass(findState);
            }
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }
```
我们回头去看`subscribe`订阅动作的执行代码
``` java
/**
     * 订阅动作
     * @param subscriber         订阅者(类似订阅的Activity之类)
     * @param subscriberMethod   订阅事件方法, 比如加了@Subscribe注解的方法
     */
    private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
        // 订阅事件的类, 比如平常传递的自己写的EventLogin等等..
        Class<?> eventType = subscriberMethod.eventType;
        Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
        // 获取与eventType有关的订阅事件的队列
        CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        // 如果为空
        if (subscriptions == null) {
            // 初始队列
            subscriptions = new CopyOnWriteArrayList<>();
            subscriptionsByEventType.put(eventType, subscriptions);
        } else {
            // 如果管理的订阅队列存在新的订阅事件, 则抛出已注册事件的异常
            if (subscriptions.contains(newSubscription)) {
                throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                        + eventType);
            }
        }
        int size = subscriptions.size();
        // 遍历订阅的事件
        for (int i = 0; i <= size; i++) {
            // 根据优先级, 插入订阅事件
            if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
                subscriptions.add(i, newSubscription);
                break;
            }
        }
        // 以订阅者为key, value为订阅事件的类的队列
        List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
        if (subscribedEvents == null) {
            subscribedEvents = new ArrayList<>();
            typesBySubscriber.put(subscriber, subscribedEvents);
        }
        subscribedEvents.add(eventType);
        // 是否粘性事件
        if (subscriberMethod.sticky) {
            // 是否分发订阅了响应事件类父类事件的方法, 默认为true
            if (eventInheritance) {
                // Existing sticky events of all subclasses of eventType have to be considered.
                // Note: Iterating over all events may be inefficient with lots of sticky events,
                // thus data structure should be changed to allow a more efficient lookup
                // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
                // stickyEvents 粘性事件集合, key为eventType的类, value是eventType对象
                Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
                for (Map.Entry<Class<?>, Object> entry : entries) {
                    // 获取候选eventType
                    Class<?> candidateEventType = entry.getKey();
                    // native方法, 应该是判断当前注册eventType与候选缓存的eventType是否匹配
                    if (eventType.isAssignableFrom(candidateEventType)) {
                        // 如果匹配, 校验并发送订阅
                        Object stickyEvent = entry.getValue();
                        checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                    }
                }
            } else {
                Object stickyEvent = stickyEvents.get(eventType);
                checkPostStickyEventToSubscription(newSubscription, stickyEvent);
            }
        }
    }
```
相关注释都在代码里, 这块的流程我们可以梳理成以下步骤:
1. 获取我们订阅时间传输的类`EventType`, 初始化内部维护的两个集合, 分别是`subscriptionsByEventType`和`typesBySubscriber`, 根据命名我们也可以理解, 一个是根据`eventType`区分的订阅者队列, 一个是根据`subscriber`(订阅者)区分的`eventType`队列, 分别向对应的集合内添加对应新的订阅者和订阅事件
2. 根据是否粘性事件判断是否需要调用`checkPostStickyEventToSubscription`直接发送信息给订阅者
3. `checkPostStickyEventToSubscription`内部判断事件是否被中断来判断是否会调用到`postToSubscription`, 就是发送信息给订阅者

## 发送信息
``` java
/**
     * 事件发送
     */
    /** Posts the given event to the event bus. */
    public void post(Object event) {
        // currentPostingThreadState 为ThreadLocal对象
        // 获取当前线程的发送状态
        PostingThreadState postingState = currentPostingThreadState.get();
        // 获取当前线程的事件发送队列
        List<Object> eventQueue = postingState.eventQueue;
        // 添加事件
        eventQueue.add(event);
        // 如果不在发送中
        if (!postingState.isPosting) {
            // 判断是否在主线程
            postingState.isMainThread = isMainThread();
            // 修改发送中状态
            postingState.isPosting = true;
            if (postingState.canceled) {
                throw new EventBusException("Internal error. Abort state was not reset");
            }
            try {
                // 遍历发送队列事件
                while (!eventQueue.isEmpty()) {
                    // 从队头开始发送, 同时移除队列中的对应事件
                    postSingleEvent(eventQueue.remove(0), postingState);
                }
            } finally {
                // 修改发送中状态, 修改主线程判断
                postingState.isPosting = false;
                postingState.isMainThread = false;
            }
        }
    }
```
由于在业务场景中, 无法判断发送信息在什么线程下执行的, 所以内部维护的`currentPostingThreadState`是一个`ThreadLocal`对象, 它可以保证当前线程的数据不会被其他线程共享.在`post`中, 我们就能看到`EventBus`会根据当前线程, 将事件发送给当前线程的队列中, 然后遍历执行`postSingleEvent`进行单个事件的发送, 同时移除掉队列中已发送的事件
``` java
/**
     * 发送单个事件
     * @param event
     * @param postingState
     * @throws Error
     */
    private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
        // event 是对应eventType的实例
        Class<?> eventClass = event.getClass();
        // 默认没有找到订阅者
        boolean subscriptionFound = false;
        // 默认true, 判断是否触发eventType的父类或接口的订阅
        if (eventInheritance) {
            // 查找获取所有eventType的父类和接口
            List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
            int countTypes = eventTypes.size();
            // 循环发送
            for (int h = 0; h < countTypes; h++) {
                Class<?> clazz = eventTypes.get(h);
                subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
            }
        } else {
            subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
        }
        // 如果没有找到订阅者
        if (!subscriptionFound) {
            if (logNoSubscriberMessages) {
                logger.log(Level.FINE, "No subscribers registered for event " + eventClass);
            }
            // 如果我们的builder配置了sendNoSubscriberEvent(默认为true)
            if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                    eventClass != SubscriberExceptionEvent.class) {
                // 会发送一个NoSubscriberEvent的事件, 如果需要判断无订阅者时候的触发情况, 可以接收这个事件做处理
                post(new NoSubscriberEvent(this, event));
            }
        }
    }
```
这里的流程我们可以分成两步:
1. 通过`postSingleEventForEventType`根据`eventType`查找对应的订阅者, 如果找到, 则发送事件
2. 如果没有找到订阅者, 根据构造器内我们通过`sendNoSubscriberEvent`的配置, 来判断是否需要发送一个无订阅者响应事件

``` java
private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
        CopyOnWriteArrayList<Subscription> subscriptions;
        synchronized (this) {
            // 根据eventType获取订阅者
            subscriptions = subscriptionsByEventType.get(eventClass);
        }
        if (subscriptions != null && !subscriptions.isEmpty()) {
            for (Subscription subscription : subscriptions) {
                postingState.event = event;
                postingState.subscription = subscription;
                boolean aborted = false;
                try {
                    // 发送给订阅者
                    postToSubscription(subscription, event, postingState.isMainThread);
                    aborted = postingState.canceled;
                } finally {
                    postingState.event = null;
                    postingState.subscription = null;
                    postingState.canceled = false;
                }
                if (aborted) {
                    break;
                }
            }
            return true;
        }
        return false;
    }
```
`postSingleEventForEventType`方法就执行了上面的第一步动作, 如果找到了订阅者, 就会返回true; 否则, 返回false.最终我们通过调用`postToSubscription`将事件发送给订阅者.
咱们继续往下走.
``` java
/**
     * 订阅发布
     * @param subscription 新注册的订阅者
     * @param event eventType
     * @param isMainThread 是否主线程
     */
    private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
        // 订阅方法的指定线程
        switch (subscription.subscriberMethod.threadMode) {
            // 相同线程内
            case POSTING:
                invokeSubscriber(subscription, event);
                break;
            // 主线程内, 不阻塞
            case MAIN:
                if (isMainThread) {
                    // 订阅者的调用
                    invokeSubscriber(subscription, event);
                } else {
                    // 通过handler处理
                    mainThreadPoster.enqueue(subscription, event);
                }
                break;
            // 主线程, 阻塞
            case MAIN_ORDERED:
                if (mainThreadPoster != null) {
                    mainThreadPoster.enqueue(subscription, event);
                } else {
                    // temporary: technically not correct as poster not decoupled from subscriber
                    invokeSubscriber(subscription, event);
                }
                break;
            // 后台线程,
            case BACKGROUND:
                if (isMainThread) {
                    // 实现了Runnable
                    backgroundPoster.enqueue(subscription, event);
                } else {
                    invokeSubscriber(subscription, event);
                }
                break;
            // 异步线程
            case ASYNC:
                asyncPoster.enqueue(subscription, event);
                break;
            default:
                throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
        }
    }
```
默认的线程模式一般是`POSTING`会走发送信息时所在的线程, 这样避免了线程切换所存在的可能开销.我们首先看下`invokeSubscriber`方法, 它的作用就是做到了订阅者的调用
``` java
void invokeSubscriber(Subscription subscription, Object event) {
        try {
            // 订阅方法的调用
            subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
        } catch (InvocationTargetException e) {
            handleSubscriberException(subscription, event, e.getCause());
        } catch (IllegalAccessException e) {
            throw new IllegalStateException("Unexpected exception", e);
        }
    }
```
其实可以看出, 这里就是获取订阅方法, 通过反射将事件作为参数调用.

我们看下走UI线程的流程, 在判断当前线程非主线程的情况下, 我们会调用到`mainThreadPoster.enqueue(subscription, event);`,
首先, 我们回到`EventBus`的构造函数中, 找到`mainThreadPoster`的相关申明
``` java
mainThreadPoster = mainThreadSupport != null ? mainThreadSupport.createPoster(this) : null;
```

``` java
public interface MainThreadSupport {

    boolean isMainThread();

    Poster createPoster(EventBus eventBus);

    class AndroidHandlerMainThreadSupport implements MainThreadSupport {

        private final Looper looper;

        public AndroidHandlerMainThreadSupport(Looper looper) {
            this.looper = looper;
        }

        @Override
        public boolean isMainThread() {
            return looper == Looper.myLooper();
        }

        @Override
        public Poster createPoster(EventBus eventBus) {
            return new HandlerPoster(eventBus, looper, 10);
        }
    }

}
```
可以看到他是个`HandlerPoster`对象, 然后再回来看`HandlerPoster.enqueue`对应的代码
``` java
public void enqueue(Subscription subscription, Object event) {
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        synchronized (this) {
            queue.enqueue(pendingPost);
            if (!handlerActive) {
                handlerActive = true;
                if (!sendMessage(obtainMessage())) {
                    throw new EventBusException("Could not send handler message");
                }
            }
        }
    }
```
这里首先维护了内部的`PendingPost`, 并且将对应的`pendingPost`加入执行队列中.`HandlerPoster`继承于`Handler`, 根据他前面传入的Looper可以判定保证信息的执行是在主线程中做处理的, 现在我们看下`handleMessage`的处理
``` java
@Override
    public void handleMessage(Message msg) {
        boolean rescheduled = false;
        try {
            long started = SystemClock.uptimeMillis();
            while (true) {
                PendingPost pendingPost = queue.poll();
                if (pendingPost == null) {
                    synchronized (this) {
                        // Check again, this time in synchronized
                        pendingPost = queue.poll();
                        if (pendingPost == null) {
                            handlerActive = false;
                            return;
                        }
                    }
                }
                // 调用订阅者
                eventBus.invokeSubscriber(pendingPost);
                long timeInMethod = SystemClock.uptimeMillis() - started;
                if (timeInMethod >= maxMillisInsideHandleMessage) {
                    if (!sendMessage(obtainMessage())) {
                        throw new EventBusException("Could not send handler message");
                    }
                    rescheduled = true;
                    return;
                }
            }
        } finally {
            handlerActive = rescheduled;
        }
    }
```
这里做的处理, 主要就是调用了`EventBus`对象的`invokeSubscriber`方法, 最终走到了订阅者的方法的执行.
至于其他的几个线程模式, 查看对应的`POST`也可以大致知道他的原理, 这里就暂且不表了.
### 解绑
相对前面的, 其实解绑的逻辑就非常简单了, 我们先看代码
``` java
/**
     * 解绑
     */
    /** Unregisters the given subscriber from all event classes. */
    public synchronized void unregister(Object subscriber) {
        // 根据订阅者获取对应的eventType
        List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
        // 如果不为空
        if (subscribedTypes != null) {
            // 遍历解绑
            for (Class<?> eventType : subscribedTypes) {
                unsubscribeByEventType(subscriber, eventType);
            }
            // 移除相关的eventType
            typesBySubscriber.remove(subscriber);
        } else {
            logger.log(Level.WARNING, "Subscriber to unregister was not registered before: " + subscriber.getClass());
        }
    }
```
我们在绑定相关的解析中, 已经知道其实内部管理订阅事件和订阅者是通过`typesBySubscriber`和`subscriptionsByEventType`来实现的, 而这里就是移除掉与对应订阅者相关的对象即可.
