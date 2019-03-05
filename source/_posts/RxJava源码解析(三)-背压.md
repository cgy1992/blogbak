title: RxJava源码解析(三)-背压
date: 2019-03-05 00:00:00
categories:  
- 源码解析
tags:
- 源码解析
---
## 什么是背压(Backpressure)?
我们来看下RxJava自身对其的解释
> When the dataflow runs through asynchronous steps, each step may perform different things with different speed. To avoid overwhelming such steps, which usually would manifest itself as increased memory usage due to temporary buffering or the need for skipping/dropping data, a so-called backpressure is applied, which is a form of flow control where the steps can express how many items are they ready to process. This allows constraining the memory usage of the dataflows in situations where there is generally no way for a step to know how many items the upstream will send to it.

来祭出google翻译,
> 当数据流通过异步步骤时，每个步骤可以以不同的速度执行不同的操作。
为了避免压倒这些步骤，这些步骤通常表现为由于临时缓冲或需要跳过/丢弃数据而增加的内存使用量，所以应用所谓的背压，这是流量控制的一种形式，其中步骤可以表达多少
物品准备好了。
这允许在通常无法知道上游将向其发送多少项的步骤的情况下约束数据流的存储器使用。

简而言之, 当`异步`情况下, 上流(被观察者)发送数据过快, 而下流(消费者)无法及时处理数据, 导致缓存内存增大, 最终导致OOM, 这个时候, 背压就是为了处理这种情况.
<!-- more -->
## `Observable / Observer`
我们都知道`RxJava2`主要是增加了使用背压机制下调用的api, 而原来的`Observable / Observer`组合是不支持背压的, 那么我们先回顾下原来的`Observable / Observer`组合使用的订阅流程
``` java
Observable
                // 新建 ObservableCreate
                .create(ObservableOnSubscribe<Int> {
                    var i = 0
                    while (true){
                        it.onNext(i ++)
                    }
                })
                // 新建 ObservableSubscribeOn
                .subscribeOn(Schedulers.io())
                // 新建 ObservableObserveOn
                .observeOn(Schedulers.newThread())
                // 返回 LambdaObserver
                .subscribe{
                    Thread.sleep(5000)
                    Log.d("rxjava", it.toString())
                }
```
[`Observable / Observer`组合的订阅消费流程](https://yutiantina.github.io/2018/02/07/RxJava2%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%901/)和[线程切换原理](https://yutiantina.github.io/2018/02/08/RxJava2%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%902/)在去年都曾经写过, 这里就不再赘述, 我们主要看下, 上游的缓冲池(buffer)的做法, 相关代码在`ObserveOnObserver#onNext`
``` java
static final class ObserveOnObserver<T> extends BasicIntQueueDisposable<T>
    implements Observer<T>, Runnable {
      ...
      @Override
       public void onSubscribe(Disposable d) {
           if (DisposableHelper.validate(this.upstream, d)) {
               this.upstream = d;
               ...

               queue = new SpscLinkedArrayQueue<T>(bufferSize);

               downstream.onSubscribe(this);
           }
       }

       @Override
       public void onNext(T t) {
           if (done) {
               return;
           }

           if (sourceMode != QueueDisposable.ASYNC) {
               queue.offer(t);
           }
           schedule();
       }
       ...
    }

public final class SpscLinkedArrayQueue<T> implements SimplePlainQueue<T> {
  public SpscLinkedArrayQueue(final int bufferSize) {
        int p2capacity = Pow2.roundToPowerOfTwo(Math.max(8, bufferSize));
        int mask = p2capacity - 1;
        AtomicReferenceArray<Object> buffer = new AtomicReferenceArray<Object>(p2capacity + 1);
        producerBuffer = buffer;
        producerMask = mask;
        adjustLookAheadStep(p2capacity);
        consumerBuffer = buffer;
        consumerMask = mask;
        producerLookAhead = mask - 1; // we know it's all empty to start with
        soProducerIndex(0L);
    }
    @Override
    public boolean offer(final T e) {
        if (null == e) {
            throw new NullPointerException("Null is not a valid element");
        }
        // local load of field to avoid repeated loads after volatile reads
        final AtomicReferenceArray<Object> buffer = producerBuffer;
        final long index = lpProducerIndex();
        final int mask = producerMask;
        final int offset = calcWrappedOffset(index, mask);
        if (index < producerLookAhead) {
            return writeToQueue(buffer, e, index, offset);
        } else {
            final int lookAheadStep = producerLookAheadStep;
            // go around the buffer or resize if full (unless we hit max capacity)
            int lookAheadElementOffset = calcWrappedOffset(index + lookAheadStep, mask);
            if (null == lvElement(buffer, lookAheadElementOffset)) { // LoadLoad
                producerLookAhead = index + lookAheadStep - 1; // joy, there's plenty of room
                return writeToQueue(buffer, e, index, offset);
            } else if (null == lvElement(buffer, calcWrappedOffset(index + 1, mask))) { // buffer is not full
                return writeToQueue(buffer, e, index, offset);
            } else {
                resize(buffer, index, offset, e, mask); // add a buffer and link old to new
                return true;
            }
        }
    }
}
```
省略掉大部分不相关代码, 这里大致做的是, 当被观察者被订阅的时候, 会初始化一个`SpscLinkedArrayQueue`类对象, 他的内部实际维护了一个`AtomicReferenceArray`对象, 这个就是我们的缓冲队列, 他的初始默认容量是128, 当我们调用到`onNext`方法的时候, 会首先判断buffer是否已满, 若未满, 则进行插入; 若已满, 则会进行扩容处理, 在异步情况下, 我们的下游无法消费掉buffer内的缓存, 导致内存占用越来越大, 最终导致OOM

## Flowable / Subscriber
好, 快速过完无背压机制下的订阅流程后, 我们来看下RxJava2是怎么通过`Flowable / Subscriber`来处理背压的
### 订阅响应流程
首先, 我们来跟着基础Demo来看下Flowable的订阅流程
``` java
Flowable
// 创建 FlowableCreate 对象
.create(FlowableOnSubscribe<Int> {
for(i in 0..150){
it.onNext(i)
Log.d("send", i.toString())
}
}, BackpressureStrategy.LATEST)
// 创建 FlowableSubscribeOn 对象
.subscribeOn(Schedulers.io())
// 创建 FlowableObserveOn 对象
.observeOn(AndroidSchedulers.mainThread())
.subscribe (object : FlowableSubscriber<Int> {
lateinit var s: Subscription
var i = 1
override fun onComplete() {
Log.d("rxjava", "complete")
}

override fun onSubscribe(s: Subscription) {
this.s = s
s.request(130)
}

override fun onNext(t: Int?) {
Log.e("rxjava receive", "${i++} $t")
}

override fun onError(t: Throwable?) {
t?.printStackTrace()
}

})
Thread.sleep(5000)
```
1. `create`操作符用来初始化一个`FlowableCreate`对象, 并返回
``` java
public static <T> Flowable<T> create(FlowableOnSubscribe<T> source, BackpressureStrategy mode) {
        ObjectHelper.requireNonNull(source, "source is null");
        ObjectHelper.requireNonNull(mode, "mode is null");
        return RxJavaPlugins.onAssembly(new FlowableCreate<T>(source, mode));
    }
```
2. `subscribeOn`操作符 将`FlowableCreate`进行包装, 以`FlowableSubscribeOn`对象返回
``` java
public final Flowable<T> subscribeOn(@NonNull Scheduler scheduler) {
        ObjectHelper.requireNonNull(scheduler, "scheduler is null");
        return subscribeOn(scheduler, !(this instanceof FlowableCreate));
    }
public final Flowable<T> subscribeOn(@NonNull Scheduler scheduler, boolean requestOn) {
        ObjectHelper.requireNonNull(scheduler, "scheduler is null");
        return RxJavaPlugins.onAssembly(new FlowableSubscribeOn<T>(this, scheduler, requestOn));
    }
```
3. `observeOn`操作符 将`FlowableSubscribeOn`进行包装, 并以`FlowableObserveOn`对象返回
``` java
public final Flowable<T> observeOn(Scheduler scheduler) {
        return observeOn(scheduler, false, bufferSize());
    }
public final Flowable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
        ObjectHelper.requireNonNull(scheduler, "scheduler is null");
        ObjectHelper.verifyPositive(bufferSize, "bufferSize");
        return RxJavaPlugins.onAssembly(new FlowableObserveOn<T>(this, scheduler, delayError, bufferSize));
    }
```
注意这里传入的`butterSize()`方法, 他的详细代码如下
``` java
static final int BUFFER_SIZE;
    static {
        BUFFER_SIZE = Math.max(1, Integer.getInteger("rx2.buffer-size", 128));
    }
public static int bufferSize() {
        return BUFFER_SIZE;
    }
```
在`FlowableObserveOn`类里, `prefetch`表示的是初始的缓存池大小, 可以看出, 如果不指定buffer大小, 那么我们默认大小就是`128`
``` java
public final class FlowableObserveOn<T> extends AbstractFlowableWithUpstream<T, T> {
  ...
  public FlowableObserveOn(
             Flowable<T> source,
             Scheduler scheduler,
             boolean delayError,
             int prefetch) {
         super(source);
         this.scheduler = scheduler;
         this.delayError = delayError;
         // 预约的缓存池初始化大小
         this.prefetch = prefetch;
     }
   ...
}
```
4. `subscribe`操作符, 以当前Demo为例, 我们这里生成了一个`LambdaSubscriber`, 而后调用到的是我们上个操作符流程返回的`Flowable#subscribe`
``` java
public final Disposable subscribe(Consumer<? super T> onNext, Consumer<? super Throwable> onError) {
        return subscribe(onNext, onError, Functions.EMPTY_ACTION, FlowableInternalHelper.RequestMax.INSTANCE);
    }
```
``` java
public final Disposable subscribe(Consumer<? super T> onNext, Consumer<? super Throwable> onError,
            Action onComplete, Consumer<? super Subscription> onSubscribe) {
        ...

        LambdaSubscriber<T> ls = new LambdaSubscriber<T>(onNext, onError, onComplete, onSubscribe);

        subscribe(ls);

        return ls;
    }
```
``` java
public final void subscribe(FlowableSubscriber<? super T> s) {
        ObjectHelper.requireNonNull(s, "s is null");
        try {
            Subscriber<? super T> z = RxJavaPlugins.onSubscribe(this, s);

            ...

            subscribeActual(z);
        } catch (NullPointerException e) { // NOPMD
            throw e;
        } catch (Throwable e) {
            Exceptions.throwIfFatal(e);
            // can't call onError because no way to know if a Subscription has been set or not
            // can't call onSubscribe because the call might have set a Subscription already
            RxJavaPlugins.onError(e);

            NullPointerException npe = new NullPointerException("Actually not, but can't throw other exceptions due to RS");
            npe.initCause(e);
            throw npe;
        }
    }
```
最终走到`Flowable#subscribeActual(Subscriber<? super T> s)`方法, 它是个抽象方法, 由调用方`FlowableObserveOn`对象实现
``` java
public final class FlowableObserveOn<T> extends AbstractFlowableWithUpstream<T, T> {
  ...
    @Override
    public void subscribeActual(Subscriber<? super T> s) {
        Worker worker = scheduler.createWorker();

        if (s instanceof ConditionalSubscriber) {
            source.subscribe(new ObserveOnConditionalSubscriber<T>(
                    (ConditionalSubscriber<? super T>) s, worker, delayError, prefetch));
        } else {
            source.subscribe(new ObserveOnSubscriber<T>(s, worker, delayError, prefetch));
        }
    }
    ...
}
```
5. 这里`source`对象是我们前面被包装的`FlowableSubscribeOn`对象, 是不是很熟悉?
`FlowableSubscribeOn`类继承于`Flowable`, 而`Flowable#subscribe(FlowableSubscriber<? super T> s)`方法最终走到 `Flowable#subscribeActual(Subscriber<? super T> s)`,上文已知这是个抽象方法, 最终执行到`FlowableSubscribeOn#subscribeActual`, 并将`LambdaSubscriber`包装成`ObserveOnSubscriber`作为参数传递进去
``` java
public final class FlowableSubscribeOn<T> extends AbstractFlowableWithUpstream<T , T> {
  ...
    @Override
    public void subscribeActual(final Subscriber<? super T> s) {
        Scheduler.Worker w = scheduler.createWorker();
        final SubscribeOnSubscriber<T> sos = new SubscribeOnSubscriber<T>(s, w, source, nonScheduledRequests);
        // s 为 ObserveOnSubscriber 对象
        s.onSubscribe(sos);

        w.schedule(sos);
   }
   ...
}
```
  -  我们先来看下`  s.onSubscribe(sos);`执行代码
``` java
static final class ObserveOnSubscriber<T> extends BaseObserveOnSubscriber<T>
    implements FlowableSubscriber<T> {
      ...
      @Override
        public void onSubscribe(Subscription s) {
            if (SubscriptionHelper.validate(this.upstream, s)) {
                this.upstream = s;

                if (s instanceof QueueSubscription) {
                    @SuppressWarnings("unchecked")
                    QueueSubscription<T> f = (QueueSubscription<T>) s;

                    int m = f.requestFusion(ANY | BOUNDARY);

                    if (m == SYNC) {
                        sourceMode = SYNC;
                        queue = f;
                        done = true;

                        downstream.onSubscribe(this);
                        return;
                    } else
                    if (m == ASYNC) {
                        sourceMode = ASYNC;
                        queue = f;

                        downstream.onSubscribe(this);

                        s.request(prefetch);

                        return;
                    }
                }
                // 初始化缓存队列
                // prefetch 默认大小为128
                queue = new SpscArrayQueue<T>(prefetch);
                // 下流订阅执行
                downstream.onSubscribe(this);
                // 上流的subscription request 执行
                s.request(prefetch);
            }
        }
      ...
    }
```
可以看到, 在这个时候, 就会执行到消费者的`onSubscribe`方法, 然后会执行到`SubscribeOnSubscriber#request`
``` java
static final class SubscribeOnSubscriber<T> extends AtomicReference<Thread>
    implements FlowableSubscriber<T>, Subscription, Runnable {
      @Override
        public void request(final long n) {
            if (SubscriptionHelper.validate(n)) {
                Subscription s = this.upstream.get();
                if (s != null) {
                    requestUpstream(n, s);
                } else {
                    BackpressureHelper.add(requested, n);
                    s = this.upstream.get();
                    if (s != null) {
                        long r = requested.getAndSet(0L);
                        if (r != 0L) {
                            requestUpstream(r, s);
                        }
                    }
                }
            }
        }
    }
```
`SubscribeOnSubscriber`实现了`Subscription`的接口, `request`方法主要用来处理内部缓存最大值
``` java
public interface Subscription {
    /**
     * No events will be sent by a {@link Publisher} until demand is signaled via this method.
     * <p>
     * It can be called however often and whenever needed—but the outstanding cumulative demand must never exceed Long.MAX_VALUE.
     * An outstanding cumulative demand of Long.MAX_VALUE may be treated by the {@link Publisher} as "effectively unbounded".
     * <p>
     * Whatever has been requested can be sent by the {@link Publisher} so only signal demand for what can be safely handled.
     * <p>
     * A {@link Publisher} can send less than is requested if the stream ends but
     * then must emit either {@link Subscriber#onError(Throwable)} or {@link Subscriber#onComplete()}.
     *
     * @param n the strictly positive number of elements to requests to the upstream {@link Publisher}
     */
    public void request(long n);

    public void cancel();
}
```
  - 这里, 和原来的线程切换原理相同, 根据`scheduler.createWorker`根据工厂模式的帮助, 创建新的`Worker`对象, 它的内部的工作原理都是通过线程池来执行对应的`Runnable`, 而我们包装后所得的`SubscribeOnSubscriber`对象正是继承于`Runnable`, 我们可以看他内部实现的`run`
方法
``` java
static final class SubscribeOnSubscriber<T> extends AtomicReference<Thread>
    implements FlowableSubscriber<T>, Subscription, Runnable {
        ...
        SubscribeOnSubscriber(Subscriber<? super T> actual, Scheduler.Worker worker, Publisher<T> source, boolean requestOn) {
            this.downstream = actual;
            this.worker = worker;
            this.source = source;
            this.upstream = new AtomicReference<Subscription>();
            this.requested = new AtomicLong();
            this.nonScheduledRequests = !requestOn;
        }

        @Override
        public void run() {
            lazySet(Thread.currentThread());
            Publisher<T> src = source;
            source = null;
            src.subscribe(this);
        }
        ...
  }
```
这里的`source`是在`FlowableSubscribeOn`初始化的时候, 通过构造函数传入的, 而我们知道, 他的内部是`FlowableCreate`
``` java
public FlowableSubscribeOn(Flowable<T> source, Scheduler scheduler, boolean nonScheduledRequests) {
        super(source);
        this.scheduler = scheduler;
        this.nonScheduledRequests = nonScheduledRequests;
    }
```
- 根据前面的套路, 我们直接去看`FlowableCreate`下的`subscribeActual`实现方法
``` java
@Override
    public void subscribeActual(Subscriber<? super T> t) {
        BaseEmitter<T> emitter;

        switch (backpressure) {
        case MISSING: {
            emitter = new MissingEmitter<T>(t);
            break;
        }
        case ERROR: {
            emitter = new ErrorAsyncEmitter<T>(t);
            break;
        }
        case DROP: {
            emitter = new DropAsyncEmitter<T>(t);
            break;
        }
        case LATEST: {
            emitter = new LatestAsyncEmitter<T>(t);
            break;
        }
        default: {
            emitter = new BufferAsyncEmitter<T>(t, bufferSize());
            break;
        }
        }
        // SubscribeOnSubscriber.onSubscribe
        t.onSubscribe(emitter);
        try {
          // 执行被观察者发送事件
            source.subscribe(emitter);
        } catch (Throwable ex) {
            Exceptions.throwIfFatal(ex);
            emitter.onError(ex);
        }
    }
```
可以看到, 根据我们选择的背压策略, 新建不同类型继承于`BaseEmitter`类对象.
  - 而这时候, 首先执行的是`SubscribeOnSubscriber#onSubscribe`
``` java
        @Override
        public void onSubscribe(Subscription s) {
            if (SubscriptionHelper.setOnce(this.upstream, s)) {
              // 初始128
                long r = requested.getAndSet(0L);
                if (r != 0L) {
                    requestUpstream(r, s);
                }
            }
        }
        // 切换线程到正确的目标线程上
        // 最终执行 Subscription#request(n)
        void requestUpstream(final long n, final Subscription s) {
            if (nonScheduledRequests || Thread.currentThread() == get()) {
                s.request(n);
            } else {
                worker.schedule(new Request(s, n));
            }
        }
```
这里的`Subscription`是前面`BaseEmitter`对象, 将其内部的缓存容量大小设为默认的128大小.
``` java
abstract static class BaseEmitter<T>
    extends AtomicLong
    implements FlowableEmitter<T>, Subscription {
      @Override
        public final void request(long n) {
            if (SubscriptionHelper.validate(n)) {
                BackpressureHelper.add(this, n);
                onRequested();
            }
        }

        void onRequested() {
            // default is no-op
        }
    }
```
  - 到这里我们的订阅整个流程已经完成, 然后我们通过实现的被观察者的发送事件, 通过`emitter`来循环发送自增整型`i`
- 我们来看下排除掉背压控制后的响应流程, 关于几个背压策略我们后文细讲,  这里我们需要知道`MissingEmitter`是不会处理背压, 所以我们从他的`onNext`看起
  ``` java
  static final class MissingEmitter<T> extends BaseEmitter<T> {
          MissingEmitter(Subscriber<? super T> downstream) {
              super(downstream);
          }

        @Override
        public void onNext(T t) {
            if (isCancelled()) {
                return;
            }

            if (t != null) {
                downstream.onNext(t);
            } else {
                onError(new NullPointerException("onNext called with null. Null values are generally not allowed in 2.x operators and sources."));
                return;
            }
            ...
        }

    }
  ```
  直接执行下流的`onNext`方法, 而它的下流是`SubscribeOnSubscriber`
  ``` java
  static final class SubscribeOnSubscriber<T> extends AtomicReference<Thread>
    implements FlowableSubscriber<T>, Subscription, Runnable {
      @Override
        public void onNext(T t) {
            downstream.onNext(t);
        }
    }
  ```
- `SubscribeOnSubscriber`的下流是`ObserveOnSubscriber`, 他的`onNext`方法在父类`BaseObserveOnSubscriber`中
  ``` java
  abstract static class BaseObserveOnSubscriber<T>
    extends BasicIntQueueSubscription<T>
    implements FlowableSubscriber<T>, Runnable {
      @Override
        public final void onNext(T t) {
            if (done) {
                return;
            }
            if (sourceMode == ASYNC) {
                trySchedule();
                return;
            }
            if (!queue.offer(t)) {
                upstream.cancel();

                error = new MissingBackpressureException("Queue is full?!");
                done = true;
            }
            trySchedule();
        }
        final void trySchedule() {
            if (getAndIncrement() != 0) {
                return;
            }
            worker.schedule(this);
        }

        @Override
        public final void run() {
            if (outputFused) {
                runBackfused();
            } else if (sourceMode == SYNC) {
                runSync();
            } else {
                runAsync();
            }
        }
    }
  ```
`ObserveOnSubscriber`内部维护的`queue`是`SpscArrayQueue`, 当维护的队列满了以后, 直接返回false, 而不会再做扩容操作, 当检测到内部缓存队列已满的时候, 会设置`done`为true, 异常为`MissingBackpressureException`
``` java
@Override
    public boolean offer(E e) {
        if (null == e) {
            throw new NullPointerException("Null is not a valid element");
        }
        // local load of field to avoid repeated loads after volatile reads
        final int mask = this.mask;
        final long index = producerIndex.get();
        final int offset = calcElementOffset(index, mask);
        if (index >= producerLookAhead) {
            int step = lookAheadStep;
            if (null == lvElement(calcElementOffset(index + step, mask))) { // LoadLoad
                producerLookAhead = index + step;
            } else if (null != lvElement(offset)) {
                return false;
            }
        }
        soElement(offset, e); // StoreStore
        soProducerIndex(index + 1); // ordered store -> atomic and ordered for size()
        return true;
    }
```
来看下他的异步处理
``` java
@Override
void runAsync() {
    int missed = 1;

    final Subscriber<? super T> a = downstream;
    final SimpleQueue<T> q = queue;

    long e = produced;

    for (;;) {

        long r = requested.get();

        while (e != r) {
            boolean d = done;
            T v;

            try {
                v = q.poll();
            } catch (Throwable ex) {
                Exceptions.throwIfFatal(ex);

                cancelled = true;
                upstream.cancel();
                q.clear();

                a.onError(ex);
                worker.dispose();
                return;
            }

            boolean empty = v == null;

            if (checkTerminated(d, empty, a)) {
                return;
            }

            if (empty) {
                break;
            }

            a.onNext(v);

            e++;
            if (e == limit) {
                if (r != Long.MAX_VALUE) {
                    r = requested.addAndGet(-e);
                }
                upstream.request(e);
                e = 0L;
            }
        }

        if (e == r && checkTerminated(done, q.isEmpty(), a)) {
            return;
        }

        int w = get();
        if (missed == w) {
            produced = e;
            missed = addAndGet(-missed);
            if (missed == 0) {
                break;
            }
        } else {
            missed = w;
        }
    }
}
```
``` java
final boolean checkTerminated(boolean d, boolean empty, Subscriber<?> a) {
    if (cancelled) {
        clear();
        return true;
    }
    if (d) {
        if (delayError) {
            if (empty) {
                cancelled = true;
                Throwable e = error;
                if (e != null) {
                    a.onError(e);
                } else {
                    a.onComplete();
                }
                worker.dispose();
                return true;
            }
        } else {
            Throwable e = error;
            if (e != null) {
                cancelled = true;
                clear();
                a.onError(e);
                worker.dispose();
                return true;
            } else
            if (empty) {
                cancelled = true;
                a.onComplete();
                worker.dispose();
                return true;
            }
        }
    }

    return false;
}
```
执行流程基本与`ObservableObserveOn`相同, 这里不再赘述.
可以注意到, 当`done`为true并且`error`不为空的时候, 会调用到下流的`onError`, 最终是抛出`MissingBackpressureException`异常
大概的流程图可参考下图![流程图](./process.jpg)
## 背压策略
Emitter的类图如下![emitterUML](./emitterUML.png)
### BaseEmitter
我们首先看下`BaseEmitter`, 它是一个抽象类, 由于继承`AtomicLong`, 所以本身可维护下流缓存阈值, 而他内部维护的`downsteam`表示的是下流的`Subscriber`, 在本文的demo中, 它就是`SubscribeOnSubscriber`对象, 他的内部还实现了`request(long n)`方法, 这个方法主要是设置内部的缓存允许阈值, 详细代码我们再上文有带到.
### NoOverflowBaseAsyncEmitter
`NoOverflowBaseAsyncEmitter`继承于`BaseEmitter`, 并覆写了onNext, 这里的代码也比较容易理解, 在缓存大小允许范围内发送数据, 则直接发送, 并计数-1, 否则调用`onOverflow`抽象方法, 可以看出`onOverflow`就是出现背压情况时候的实际处理策略
``` java
abstract static class NoOverflowBaseAsyncEmitter<T> extends BaseEmitter<T> {
        @Override
        public final void onNext(T t) {
            if (isCancelled()) {
                return;
            }

            ...

            if (get() != 0) {
                downstream.onNext(t);
                // 阈值-1
                BackpressureHelper.produced(this, 1);
            } else {
                onOverflow();
            }
        }
        // 超出阈值情况处理
        abstract void onOverflow();
    }
```
### ErrorAsyncEmitter
`ErrorAsyncEmitter`继承于`NoOverflowBaseAsyncEmitter`, 当出现背压情况的时候, 则直接调用`onError`, 抛出`MissingBackpressureException`异常
``` java
static final class ErrorAsyncEmitter<T> extends NoOverflowBaseAsyncEmitter<T> {
        ...
        @Override
        void onOverflow() {
            onError(new MissingBackpressureException("create: could not emit value due to lack of requests"));
        }

    }
```
### DropAsyncEmitter
`DropAsyncEmitter`继承于`NoOverflowBaseAsyncEmitter`, 背压策略不做任务处理, 则当超出缓存允许大小的时候, 对应的发送数据就不会再调用到下流的响应事件, 相当于drop
``` java
static final class DropAsyncEmitter<T> extends NoOverflowBaseAsyncEmitter<T> {

        ...
        @Override
        void onOverflow() {
            // nothing to do
        }

    }
```
### LatestAsyncEmitter
它的内部维护了一个`AtomicReference`的`queue`, 可以理解为其自身的缓存池, 而这个池只有一个内存大小, 它在调用到`onNext`方法的时候, 将发送的数据存入`queue`, 并调用`drain`, 我们主要看下`drain`方法
``` java
void drain() {
            // 省略保证线程安全相关代码
            ...
            final Subscriber<? super T> a = downstream;
            final AtomicReference<T> q = queue;

            for (;;) {
                long r = get();
                long e = 0L;

                while (e != r) {
                    if (isCancelled()) {
                        q.lazySet(null);
                        return;
                    }

                    boolean d = done;

                    T o = q.getAndSet(null);

                    boolean empty = o == null;

                    if (d && empty) {
                        Throwable ex = error;
                        if (ex != null) {
                            error(ex);
                        } else {
                            complete();
                        }
                        return;
                    }

                    if (empty) {
                        break;
                    }

                    a.onNext(o);

                    e++;
                }

                if (e == r) {
                    if (isCancelled()) {
                        q.lazySet(null);
                        return;
                    }

                    boolean d = done;

                    boolean empty = q.get() == null;

                    if (d && empty) {
                        Throwable ex = error;
                        if (ex != null) {
                            error(ex);
                        } else {
                            complete();
                        }
                        return;
                    }
                }

                if (e != 0) {
                    BackpressureHelper.produced(this, e);
                }

                ...
            }
        }
```
我们可以看到, 它在内部缓存阈值允许范围内, 会遍历读取`queue`的数据, 并将其发送给下流,当然这里`queue`的缓存大小也只有一个, 所以才会出现消费的最后一个数据会是上流发送的最后一个数据
### BufferAsyncEmitter
`BufferAsyncEmitter`内部维护了一个128大小的`SpscLinkedArrayQueue`缓存队列, `SpscLinkedArrayQueue`我们在`ObserveOnObserver`中见过, 他在存入数据的时候, 当达到队列最大长度的时候, 会进行扩容处理, 而通过`BufferAsyncEmitter`每次发送数据的时候, 首先会存入到`queue`中, 然后调用`drain`, 这里的代码与`LatestAsyncEmitter`也很相似, 区别只在于缓冲区的区别, 而当缓存区一再扩容的时候, 有可能出现OOM的现象, 他的逻辑其实与`ObserveOnObserver`内的消费逻辑是一样的
``` java
void drain() {
            ...
            final Subscriber<? super T> a = downstream;
            final SpscLinkedArrayQueue<T> q = queue;

            for (;;) {
                long r = get();
                long e = 0L;

                while (e != r) {
                    if (isCancelled()) {
                        q.clear();
                        return;
                    }

                    boolean d = done;

                    T o = q.poll();

                    boolean empty = o == null;

                    if (d && empty) {
                        Throwable ex = error;
                        if (ex != null) {
                            error(ex);
                        } else {
                            complete();
                        }
                        return;
                    }

                    if (empty) {
                        break;
                    }

                    a.onNext(o);

                    e++;
                }

                if (e == r) {
                    if (isCancelled()) {
                        q.clear();
                        return;
                    }

                    boolean d = done;

                    boolean empty = q.isEmpty();

                    if (d && empty) {
                        Throwable ex = error;
                        if (ex != null) {
                            error(ex);
                        } else {
                            complete();
                        }
                        return;
                    }
                }

                if (e != 0) {
                    BackpressureHelper.produced(this, e);
                }

                ...
            }
        }
```
## 背压总结
我们再回头看下, 几种背压策略的效果

| 策略 | 效果 |  
| ------ | ------ |
| MISSING | 无任何背压策略执行, 除非调用`onBackpressureXXX` |  
| ERROR | 抛出异常 |  
|BUFFER|内部维护可扩容的缓存池, 效果与`Observer`一样, 可能导致OOM|
|DROP|如果下流无法跟上上流发射速度, 则会丢弃这块数据|
|LATEST|当下流无法跟上上流的发射速度的时候, 则只保存最近生产的数据|
