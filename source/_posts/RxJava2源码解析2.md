title: RxJava2源码解析(二)
date: 2018-02-08 00:00:00
categories:
- 源码解析
tags:
- 源码解析
- rxJava2
---
## 前言
本篇主要解析RxJava的线程切换的原理实现
<!-- more -->
## subscribeOn
首先, 我们先看下`subscribeOn()`方法, 老样子, 先上Demo
``` java
Observable<Integer> observable =
                Observable
                .create(new ObservableOnSubscribe<Integer>() {
                    @Override
                    public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
                        emitter.onNext(123);
                        emitter.onComplete();
                    }
                });

observable
        .subscribeOn(Schedulers.io())
        .subscribe(getObserver());
```
`subscribeOn`操作符源码里其实是返回了一个`ObservableSubscribeOn`对象, 而从[上篇](https://yutiantina.github.io/2018/02/07/RxJava2%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%901/)我们已经知道, 订阅的动作其实在每个`Observable`的`subscribeActual(observer)`中执行, 所以我们直接去看`ObservableSubscribeOn`中的对应重载方法就行了.
``` java
@Override
    public void subscribeActual(final Observer<? super T> s) {
        final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(s);

        s.onSubscribe(parent);

        parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
    }
```
``` java
final class SubscribeTask implements Runnable {
        private final SubscribeOnObserver<T> parent;

        SubscribeTask(SubscribeOnObserver<T> parent) {
            this.parent = parent;
        }

        @Override
        public void run() {
            source.subscribe(parent);
        }
    }
```
`SubscribeTask`是一个Runnable的实现类, 执行内容就是修饰后的`Observer`订阅上游的动作, 我们先看`scheduler.scheduleDirect(runable)`方法
``` java
@NonNull
public Disposable scheduleDirect(@NonNull Runnable run) {
    return scheduleDirect(run, 0L, TimeUnit.NANOSECONDS);
}

@NonNull
public Disposable scheduleDirect(@NonNull Runnable run, long delay, @NonNull TimeUnit unit) {
    final Worker w = createWorker();

    final Runnable decoratedRun = RxJavaPlugins.onSchedule(run);

    DisposeTask task = new DisposeTask(decoratedRun, w);

    w.schedule(task, delay, unit);

    return task;
}
```
这里`createWorker`是个抽象方法, 我们需要找到对应的修饰类, 我们返回去看`Schedulers.io()`, `IO`是`IoScheduler`的实例, 它的重载方法代码如下
``` java
final AtomicReference<CachedWorkerPool> pool;
public Worker createWorker() {
        return new EventLoopWorker(pool.get());
    }
```
可以看到IO线程实际使用的是一个有线程缓存的线程调度器.它内部通过`ScheduledExecutorService`实例来尝试重用之前`worker`开始使用的实例, 由于本篇着重在流程实现原理, 所以略过细节处.
在`EventLoopWorker`中, 我们看下对应的重载方法
``` java
public Disposable schedule(@NonNull Runnable action, long delayTime, @NonNull TimeUnit unit) {
            if (tasks.isDisposed()) {
                // don't schedule, we are unsubscribed
                return EmptyDisposable.INSTANCE;
            }

            return threadWorker.scheduleActual(action, delayTime, unit, tasks);
        }
```
继续往下, 其实这个时候已经是在线程池目标线程执行相关的工作了. 再深入就是线程池的操作了, 所以这里我们不再赘述
``` java
public ScheduledRunnable scheduleActual(final Runnable run, long delayTime, @NonNull TimeUnit unit, @Nullable DisposableContainer parent) {
        Runnable decoratedRun = RxJavaPlugins.onSchedule(run);

        ScheduledRunnable sr = new ScheduledRunnable(decoratedRun, parent);

        if (parent != null) {
            if (!parent.add(sr)) {
                return sr;
            }
        }

        Future<?> f;
        try {
            if (delayTime <= 0) {
                f = executor.submit((Callable<Object>)sr);
            } else {
                f = executor.schedule((Callable<Object>)sr, delayTime, unit);
            }
            sr.setFuture(f);
        } catch (RejectedExecutionException ex) {
            if (parent != null) {
                parent.remove(sr);
            }
            RxJavaPlugins.onError(ex);
        }

        return sr;
    }
```
由此我们可以看出来, 每次每个`subscribeOn`操作符执行的时候, 其实在`source.subscribe(parent);`订阅动作就做了线程切换, 所以在多次调`subscribeOn`的时候, 就会一直切换线程, 直到离`ObservableSource`最近的`subscribeOn`线程切换生效.
## observeOn
废话不说, 我们直接看`ObservableObserveOn.subscribeActual(observer)`
``` java
protected void subscribeActual(Observer<? super T> observer) {
        if (scheduler instanceof TrampolineScheduler) {
            source.subscribe(observer);
        } else {
            Scheduler.Worker w = scheduler.createWorker();

            source.subscribe(new ObserveOnObserver<T>(observer, w, delayError, bufferSize));
        }
    }
```
熟悉的配方, 当相同想成的时候, 直接订阅, 而当不同线程的时候, 可以看到我们获取目标切换线程对应的worker实例以及装饰对应的`Observer`成`ObserveOnOberver`,后面的流程我们心知肚明, 就是`Observer`层层订阅上去, 然后我们看当碰到最上流的`ObservableSource`往下执行的时候, 做什么操作.具体我们看`ObserveOnOberver`代码, 我们这里着重看下`onSubscribe`和`onNext`方法
``` java
@Override
public void onSubscribe(Disposable s) {
    if (DisposableHelper.validate(this.s, s)) {
        this.s = s;
        // 发送的数据是集合队列形式的时候
        if (s instanceof QueueDisposable) {
            @SuppressWarnings("unchecked")
            QueueDisposable<T> qd = (QueueDisposable<T>) s;

            int m = qd.requestFusion(QueueDisposable.ANY | QueueDisposable.BOUNDARY);
            //是同步模式的时候
            if (m == QueueDisposable.SYNC) {
                sourceMode = m;
                queue = qd;
                done = true;
                actual.onSubscribe(this);
                // 线程调度
                schedule();
                return;
            }
            // 异步模式
            if (m == QueueDisposable.ASYNC) {
                sourceMode = m;
                queue = qd;
                actual.onSubscribe(this);
                return;
            }
        }

        queue = new SpscLinkedArrayQueue<T>(bufferSize);

        actual.onSubscribe(this);
    }
}

@Override
public void onNext(T t) {
    // 是否已经调用到onComplete 或者 onError, 如果是, 则不再执行后面的onNext
    if (done) {
        return;
    }
    // 如果是非异步操作, 将数据添加到队列中
    if (sourceMode != QueueDisposable.ASYNC) {
        queue.offer(t);
    }
    // 线程调度
    schedule();
}
```
大概的注释都加在代码上了, 我们再补充看下看`onSubscribe`方法, 首先判断发送的数据是否属于`QueueDisposable`, 如果不是, 直接执行下游的`onSubscribe`,这里我卡了一下, 看不到他的线程切换是在哪里做, 后来往回看, 发现在我们执行`ObservableSubscribeOn.subscribeActual(observer)`的时候, `onSubscribe()`方法本身的确不是在切换后的线程内执行的. 但是, 当我们发送的是集合数据, 那么我们需要判断是哪种线程模式进行线程调度.

我们来看具体的`schedule()`方法代码
``` java
void schedule() {
            // 判断当前自增值是否为0, 原子性保证worker.schedule(this);不会在调用结束前被重复调用
            if (getAndIncrement() == 0) {
                worker.schedule(this);
            }
        }
```
这个时候就是在指定线程内run了, `Disposable schedule(@NonNull Runnable run)`传入的是个`Runnable`的实现类, 我们来找重载的`run`方法
``` java
@Override
public void run() {
    if (outputFused) {
        drainFused();
    } else {
        drainNormal();
    }
}

void drainNormal() {
    int missed = 1;

    final SimpleQueue<T> q = queue;
    final Observer<? super T> a = actual;
    // 无限循环
    for (;;) {
        // 判断是否被取消, 或者调用onError 或者调用onComplete则退出循环
        if (checkTerminated(done, q.isEmpty(), a)) {
            return;
        }
        // 无限循环
        for (;;) {
            boolean d = done;
            T v;

            try {
                // 队列数据分发
                v = q.poll();
            } catch (Throwable ex) {
                Exceptions.throwIfFatal(ex);
                s.dispose();
                q.clear();
                a.onError(ex);
                worker.dispose();
                return;
            }
            boolean empty = v == null;
            // 判断是否应该被终止
            if (checkTerminated(d, empty, a)) {
                return;
            }

            if (empty) {
                break;
            }
            a.onNext(v);
        }
        // 原子性保证worker.schedule(this)的调用
        missed = addAndGet(-missed);
        if (missed == 0) {
            break;
        }
    }
}

// 判断循环是否终止
boolean checkTerminated(boolean d, boolean empty, Observer<? super T> a) {
    // 如果订阅已经被取消, 则清除队列, 终止
    if (cancelled) {
        queue.clear();
        return true;
    }
    // 如果调用过onError 或者 onComplete
    if (d) {
        Throwable e = error;
        // 默认false
        if (delayError) {
            // 等到队列为空的时候再调用onError或者onComplete
            if (empty) {
                if (e != null) {
                    a.onError(e);
                } else {
                    a.onComplete();
                }
                worker.dispose();
                return true;
            }
        } else {
            // 如果有抛出异常, 走下游的onError
            // 线程任务停止
            if (e != null) {
                queue.clear();
                a.onError(e);
                worker.dispose();
                return true;
            }
            // 没有, 走下游的onComplete
            // 线程任务停止
            else if (empty) {
                a.onComplete();
                worker.dispose();
                return true;
            }
        }
    }
    // 否则不结束
    return false;
}
```
由此我们可以得出结论, `observeOn`的操作符可以保证我们下流操作线程切换生效
## 总结
到这里, 我们线程切换的原理大体流程就基本分析完毕了, 可以看出`subscribeOn`操作符只对上游生效, 而且因为他是在订阅的时候进行线程切换, 而我们每个操作符中间都有订阅动作, 所以越接近我们的`ObservableSource`的订阅的`subscribeOn`越是最后生效的. 而`observeOn`生效在我们的`onNext`,`onComplete`, `onError`方法内, 所以每次的`observeOn`针对它的下游都可以生效.
