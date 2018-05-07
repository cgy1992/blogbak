title: RxJava2源码解析(一)
date: 2018-02-07 00:00:00
categories:
- 源码解析
tags:
- 源码解析
- RxJava2
---
## 前言
最近组内大佬打算分享RxJava2的源码, 赶紧先预习一波, 防止技术分享会上有听没懂.大概个人准备了几天的时间, 打算先整理以下自己的源码阅读记录.RxJava2的源码解析系列打算分别从以下三面来阐述:
1. 数据源的订阅和响应原理
2. 线程切换的原理
3. 背压的实现(Flowable)

本篇主要尝试阐明**数据源的订阅和响应原理**
<!-- more -->
## 基础使用的Demo
抛开线程切换和背压, 我们来写一个单纯的发送数据, 订阅响应的Demo,为了便于理解, 我们抛开链式调用来写
``` java
// 被观察者
Observable<Integer> observable = Observable.create(new ObservableOnSubscribe<Integer>() {
    @Override
    public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
        Log.e(TAG, "subscribe");
        emitter.onNext(123);
        emitter.onComplete();
    }
});
// 观察者
Observer<Integer> observer = new Observer<Integer>() {
    @Override
    public void onSubscribe(Disposable d) {
        Log.e(TAG, "onSubscribe");
    }

    @Override
    public void onNext(Integer integer) {
        Log.e(TAG, "onNext" + integer);
    }

    @Override
    public void onError(Throwable e) {
        Log.e(TAG, "onError");
    }

    @Override
    public void onComplete() {
        Log.e(TAG, "onComplete");
    }
};
// 订阅
observable.subscribe(observer);
```
## ObservableSource
我们首先来看当我们创建一个`Observable`(被观察者)的时候, 实际上他做了什么
``` java
public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
        // npe校验
        ObjectHelper.requireNonNull(source, "source is null");
        return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
    }

public static <T> Observable<T> onAssembly(@NonNull Observable<T> source) {
    Function<? super Observable, ? extends Observable> f = onObservableAssembly;
    if (f != null) {
        return apply(f, source);
    }
    return source;
}
```
`RxJavaPlugins.onAssembly()`这个方法主要是为了hook使用, 本篇暂且不表. 所以这里`Observable.create()`返回的是一个`ObervableCreate`对象.它继承于`Observable`, 是`ObservableSource`的实现类
## observable.subscribe(observer)
我们主要看订阅的时候做了什么, 先上源码
``` java
public final void subscribe(Observer<? super T> observer) {
        // npe校验
        ObjectHelper.requireNonNull(observer, "observer is null");
        try {
            // hook, 主要返回的就是我们的observer
            observer = RxJavaPlugins.onSubscribe(this, observer);
            // npe校验
            ObjectHelper.requireNonNull(observer, "Plugin returned null Observer");

            subscribeActual(observer);
        } catch (NullPointerException e) { // NOPMD
            throw e;
        } catch (Throwable e) {
            Exceptions.throwIfFatal(e);
            // can't call onError because no way to know if a Disposable has been set or not
            // can't call onSubscribe because the call might have set a Subscription already
            RxJavaPlugins.onError(e);

            NullPointerException npe = new NullPointerException("Actually not, but can't throw other exceptions due to RS");
            npe.initCause(e);
            throw npe;
        }
    }
```
可以看到这里实际执行的是`subscribeActual(observer)`这个方法, 这里调用是个抽象接口, 我们在`ObervableCreate`找具体的实现
``` java
@Override
protected void subscribeActual(Observer<? super T> observer) {
        // 包装数据发射器
        CreateEmitter<T> parent = new CreateEmitter<T>(observer);
        // 订阅监听
        observer.onSubscribe(parent);

        try {
            // 上游的执行
            source.subscribe(parent);
        } catch (Throwable ex) {
            Exceptions.throwIfFatal(ex);
            parent.onError(ex);
        }
    }
```
``` java
emitter.onNext(123);
emitter.onComplete();
```
从`source.subscribe(parent);`我们就会走到以下我们自己写的数据发送事件.这里的`emitter`通过源码我们可以看到是将`observer`包装后的`CreateEmitter`类对象, 我们在往里面看.
``` java
static final class CreateEmitter<T> extends AtomicReference<Disposable> implements ObservableEmitter<T>, Disposable {


        private static final long serialVersionUID = -3434801548987643227L;

        final Observer<? super T> observer;

        CreateEmitter(Observer<? super T> observer) {
            this.observer = observer;
        }

        @Override
        public void onNext(T t) {
            if (t == null) {
                onError(new NullPointerException("onNext called with null. Null values are generally not allowed in 2.x operators and sources."));
                return;
            }
            if (!isDisposed()) {
                observer.onNext(t);
            }
        }

        @Override
        public void onError(Throwable t) {
            if (!tryOnError(t)) {
                RxJavaPlugins.onError(t);
            }
        }

        @Override
        public boolean tryOnError(Throwable t) {
            if (t == null) {
                t = new NullPointerException("onError called with null. Null values are generally not allowed in 2.x operators and sources.");
            }
            if (!isDisposed()) {
                try {
                    observer.onError(t);
                } finally {
                    dispose();
                }
                return true;
            }
            return false;
        }

        @Override
        public void onComplete() {
            if (!isDisposed()) {
                try {
                    observer.onComplete();
                } finally {
                    dispose();
                }
            }
        }

        @Override
        public void setDisposable(Disposable d) {
            DisposableHelper.set(this, d);
        }

        @Override
        public void setCancellable(Cancellable c) {
            setDisposable(new CancellableDisposable(c));
        }

        @Override
        public ObservableEmitter<T> serialize() {
            return new SerializedEmitter<T>(this);
        }

        @Override
        public void dispose() {
            DisposableHelper.dispose(this);
        }

        @Override
        public boolean isDisposed() {
            return DisposableHelper.isDisposed(get());
        }
    }
```
通过之前将`observer`传入`CreateEmitter`, 调用`emitter.onNext`最终调用走到了`observer.onNext`.
整体的流程非常的清晰. 下面我们看下, 如果中间有多重数据转换, 是什么样的流程
## 数据转换实现流程
以第一个基础demo为例, 我们改造下`Observable`(被观察者), 将他进行一次数据转换, 并且做一次筛除.这个demo的意思就是发送123, 中间做+1处理, 然后筛选出大于122的数据发送给观察者.这个很容易理解.
``` java
Observable<Integer> observable =
                Observable
                        .create(new ObservableOnSubscribe<Integer>() {
                            @Override
                            public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
                                Log.e(TAG, "subscribe");
                                emitter.onNext(123);
                                emitter.onComplete();
                            }
                        })
                        .map(new Function<Integer, Integer>() {
                            @Override
                            public Integer apply(Integer integer) throws Exception {
                                Log.e(TAG, "map");
                                return integer + 1;
                            }
                        })
                        .filter(new Predicate<Integer>() {
                            @Override
                            public boolean test(Integer integer) throws Exception {
                                Log.e(TAG, "filter");
                                return integer > 122;
                            }
                        });
```
我们依旧来看下`map`操作符的源码
``` java
public final <R> Observable<R> map(Function<? super T, ? extends R> mapper) {
        ObjectHelper.requireNonNull(mapper, "mapper is null");
        return RxJavaPlugins.onAssembly(new ObservableMap<T, R>(this, mapper));
    }
```
是不是很眼熟? 忽略掉hook, 这里返回的是`ObservableMap`对象.同样, `filter`操作符返回的是一个`ObservableFilter`
``` java
public final Observable<T> filter(Predicate<? super T> predicate) {
        ObjectHelper.requireNonNull(predicate, "predicate is null");
        return RxJavaPlugins.onAssembly(new ObservableFilter<T>(this, predicate));
    }
```
不论是`ObservableMap`还是`ObservableFilter`他们都继承于`AbstractObservableWithUpstream`抽象类, 它继承于`Observable`, 带有上游的`Observable`
``` java
abstract class AbstractObservableWithUpstream<T, U> extends Observable<U> implements HasUpstreamObservableSource<T> {

    /** The source consumable Observable. */
    // 上游Obervable
    protected final ObservableSource<T> source;

    /**
     * Constructs the ObservableSource with the given consumable.
     * @param source the consumable Observable
     */
    AbstractObservableWithUpstream(ObservableSource<T> source) {
        this.source = source;
    }

    @Override
    public final ObservableSource<T> source() {
        return source;
    }

}
```
这时候, 我们重新看下订阅的处理, 当我们执行`observable.subscribe(observer)`的时候, `observable`最终返回的是`ObservableFilter`对象, 所以我们需要看这个类对象的`subscribeActual(observer)`方法.他的代码很简洁, 实际就是将我们的`observer`和`filter`操作符的具体操作方法包装成一个`FilterObserver`对象, 然后由上游`ObservableMap`对象来subscribe(订阅)它.
``` java
@Override
    public void subscribeActual(Observer<? super T> s) {
        source.subscribe(new FilterObserver<T>(s, predicate));
    }
```
我们已经知道`Observable.subscribe(observer)`方法实际调用的是对应实现类的`subscribeActual(observer)`方法, 所以我们直接去看`ObservableMap.subscribeActual(observer)`方法就可以了, 他的方法与`FilterObserver`内的类似, 这时候是将前面传进来的`FilterObserver`对象和我们`map`操作符做的操作包装成一个`MapObserver`对象, 交给上游.
``` java
@Override
    public void subscribeActual(Observer<? super U> t) {
        source.subscribe(new MapObserver<T, U>(t, function));
    }
```
这时候我们的上游是`ObservableCreate`对象,它的`subscribeActual(observer)`方法上文有提到, 他将`MapObserver`对象包装进`CreateEmitter`对象, 这个时候, 才开始执行订阅动作, 然后我们走到`CreateEmitter`的`onNext()`方法,实际会执行到下游观察者的`onNext`方法, 在这层, 我们的观察者是`MapObserver`.它继承于`BasicFuseableObserver`, 表示一个流程执行中间的观察者对象. 现在我们看`MapObserver`的`onNext`的执行, 这里我们主要关注主流程的执行逻辑, 忽略掉其他代码, 可以看到它最终调用的是`actual.onNext(v)`, 首先将我们`map`操作符的逻辑处理返回的数据赋值给`v`, 这里的`actual`指的是我们下游的`observer`(观察者), 那么这个时候是我们的`FilterObserver`对象, 将`v`对象通过`onNext`传递下去.
``` java
static final class MapObserver<T, U> extends BasicFuseableObserver<T, U> {
        final Function<? super T, ? extends U> mapper;

        MapObserver(Observer<? super U> actual, Function<? super T, ? extends U> mapper) {
            super(actual);
            this.mapper = mapper;
        }

        @Override
        public void onNext(T t) {
            if (done) {
                return;
            }

            if (sourceMode != NONE) {
                actual.onNext(null);
                return;
            }

            U v;

            try {
                v = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper function returned a null value.");
            } catch (Throwable ex) {
                fail(ex);
                return;
            }
            actual.onNext(v);
        }

        @Override
        public int requestFusion(int mode) {
            return transitiveBoundaryFusion(mode);
        }

        @Nullable
        @Override
        public U poll() throws Exception {
            T t = qs.poll();
            return t != null ? ObjectHelper.<U>requireNonNull(mapper.apply(t), "The mapper function returned a null value.") : null;
        }
    }
```
然后我们看`FilterObserver`的源码, 他的`onNext`逻辑就是会执行我们传进去的`Predicate`对象的`test`方法, 如果符合筛选逻辑, 就会通过调用下游的`onNext`将数据传下去, 这个时候的下游是我们new的`Observer`, 这时候的执行,我们应该就清楚了.
``` java
static final class FilterObserver<T> extends BasicFuseableObserver<T, T> {
        final Predicate<? super T> filter;

        FilterObserver(Observer<? super T> actual, Predicate<? super T> filter) {
            super(actual);
            this.filter = filter;
        }

        @Override
        public void onNext(T t) {
            if (sourceMode == NONE) {
                boolean b;
                try {
                    b = filter.test(t);
                } catch (Throwable e) {
                    fail(e);
                    return;
                }
                if (b) {
                    actual.onNext(t);
                }
            } else {
                actual.onNext(null);
            }
        }

        @Override
        public int requestFusion(int mode) {
            return transitiveBoundaryFusion(mode);
        }

        @Nullable
        @Override
        public T poll() throws Exception {
            for (;;) {
                T v = qs.poll();
                if (v == null || filter.test(v)) {
                    return v;
                }
            }
        }
    }
```
## 总结
订阅和数据的传输的原理就是如此. 我们用流程图来总结下上面的整个流程.
![流程图](RxJava2源码解析/rxjava实现原理.jpg)
总的来说, 订阅的动作是层层递归上传到最开始的`Observable`, 然后从最开始的`Observable`将数据一层层往下传.
当然, 从`装饰模式`来讲, 他这里的实际动作就是将`Observable`做了层层装饰来传递订阅, 对设计模式有兴趣的同学可以看看相关的书籍, 对于理解这段代码有点睛之用
