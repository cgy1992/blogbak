title: LeakCanary简易解析
date: 2018-12-03 00:00:00
categories:
- 源码解析
tags:
- android
- 源码解析
---
## 前言
本篇基于`1.6.1`版本源码阅读, 本篇内容就是搞懂`LeakCanary`如何做到内存泄漏定位的主要流程, 不抠具体细节.
<!--more-->
## 正文
老样子, 我们直接从从`LeakCanary.install(this)`作为入口开始看
``` java
public static RefWatcher install(Application application) {
    return refWatcher(application).listenerServiceClass(DisplayLeakService.class)
        .excludedRefs(AndroidExcludedRefs.createAppDefaults().build())
        .buildAndInstall();
  }

  public RefWatcher buildAndInstall() {
      if (LeakCanaryInternals.installedRefWatcher != null) {
        throw new UnsupportedOperationException("buildAndInstall() should only be called once.");
      }
      RefWatcher refWatcher = build();
      if (refWatcher != DISABLED) {
        if (watchActivities) {
          ActivityRefWatcher.install(context, refWatcher);
        }
        if (watchFragments) {
          FragmentRefWatcher.Helper.install(context, refWatcher);
        }
      }
      LeakCanaryInternals.installedRefWatcher = refWatcher;
      return refWatcher;
    }

```
通过`AndroidRefWatcherBuilder`对象进行一系列相关对象的初始化, 包括`ServiceHeapDumpListener`, `ExcludedRefs`,以及最重要的`RefWatcher`.
1. `ServiceHeapDumpListener`:堆解析监听, 主要负责启动解析Heap服务以及负责处理引用路径的解析服务的连接
2. `ExcludedRefs`: 内存泄漏分析的白名单
3. `RefWatcher`: 观测引用是否弱可达, 通过它来观测是否该回收的内存未被回收导致内存泄漏, 如果存在这种情况, 会触发HeapDumper的记录

我们通过`ActivityRefWatcher.install(context, refWatcher)`查看Actvitiy的内存泄漏的分析流程
``` java
public static void install(Context context, RefWatcher refWatcher) {
    Application application = (Application) context.getApplicationContext();
    ActivityRefWatcher activityRefWatcher = new ActivityRefWatcher(application, refWatcher);

    application.registerActivityLifecycleCallbacks(activityRefWatcher.lifecycleCallbacks);
  }

  private final Application.ActivityLifecycleCallbacks lifecycleCallbacks =
      new ActivityLifecycleCallbacksAdapter() {
        @Override public void onActivityDestroyed(Activity activity) {
          refWatcher.watch(activity);
        }
      };
```
可以看到注册了一个`ActivityLifecycleCallbacks`, 在页面生命周期走到`onDestroy`的时候, 会触发`refWatcher.watch(activity)`
``` java
public void watch(Object watchedReference) {
    watch(watchedReference, "");
  }

  /**
   * Watches the provided references and checks if it can be GCed. This method is non blocking,
   * the check is done on the {@link WatchExecutor} this {@link RefWatcher} has been constructed
   * with.
   *
   * @param referenceName An logical identifier for the watched object.
   */
  public void watch(Object watchedReference, String referenceName) {
    if (this == DISABLED) {
      return;
    }
    checkNotNull(watchedReference, "watchedReference");
    checkNotNull(referenceName, "referenceName");
    final long watchStartNanoTime = System.nanoTime();
    String key = UUID.randomUUID().toString();
    retainedKeys.add(key);
    final KeyedWeakReference reference =
        new KeyedWeakReference(watchedReference, key, referenceName, queue);

    ensureGoneAsync(watchStartNanoTime, reference);
  }
```
申明弱引用, 放入`activity`对象, 注册关联`queue`引用队列
``` java
private void ensureGoneAsync(final long watchStartNanoTime, final KeyedWeakReference reference) {
    watchExecutor.execute(new Retryable() {
      @Override public Retryable.Result run() {
        return ensureGone(reference, watchStartNanoTime);
      }
    });
  }
```
在`AndroidRefWatcherBuilder`可以通过`watchDelay(long delay, TimeUnit unit)`设置是否延迟观测, 如果有设置, 则会在保证在主线程内延迟进行分析内存泄漏; 否则直接执行分析处理
``` java
Retryable.Result ensureGone(final KeyedWeakReference reference, final long watchStartNanoTime) {
    long gcStartNanoTime = System.nanoTime();
    long watchDurationMs = NANOSECONDS.toMillis(gcStartNanoTime - watchStartNanoTime);

    removeWeaklyReachableReferences();

    if (debuggerControl.isDebuggerAttached()) {
      // The debugger can create false leaks.
      return RETRY;
    }
    // 不存在内存泄漏的对象
    if (gone(reference)) {
      return DONE;
    }
    // 重新执行gc
    gcTrigger.runGc();
    removeWeaklyReachableReferences();
    if (!gone(reference)) {
      long startDumpHeap = System.nanoTime();
      long gcDurationMs = NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);

      File heapDumpFile = heapDumper.dumpHeap();
      if (heapDumpFile == RETRY_LATER) {
        // Could not dump the heap.
        return RETRY;
      }
      long heapDumpDurationMs = NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);

      HeapDump heapDump = heapDumpBuilder.heapDumpFile(heapDumpFile).referenceKey(reference.key)
          .referenceName(reference.name)
          .watchDurationMs(watchDurationMs)
          .gcDurationMs(gcDurationMs)
          .heapDumpDurationMs(heapDumpDurationMs)
          .build();

      heapdumpListener.analyze(heapDump);
    }
    return DONE;
  }
```
``` java
private void removeWeaklyReachableReferences() {
    // WeakReferences are enqueued as soon as the object to which they point to becomes weakly
    // reachable. This is before finalization or garbage collection has actually happened.
    KeyedWeakReference ref;
    while ((ref = (KeyedWeakReference) queue.poll()) != null) {
      retainedKeys.remove(ref.key);
    }
  }
```
每个`activity`申明弱引用的时候都会有个ID, ID保存在`retainedKeys`集合中, 首先遍历移除被gc回收的对象, 如果这个时候`retainedKeys`集合为空, 则表示不存在内存泄漏的情况. 否则手动执行GC, 再次判断移除, 这个时候如果`retainedKeys`内仍存在ID, 则说明有内存泄漏的情况存在.

在存在内存泄漏的情况下, 通过`heapDumper.dumpHeap()`获取堆内存快照, 通过`heapdumpListener.analyze`去进行解析.
``` java
public void analyze(HeapDump heapDump) {
    checkNotNull(heapDump, "heapDump");
    HeapAnalyzerService.runAnalysis(context, heapDump, listenerServiceClass);
  }
```
``` java
public static void runAnalysis(Context context, HeapDump heapDump,
      Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
    setEnabledBlocking(context, HeapAnalyzerService.class, true);
    setEnabledBlocking(context, listenerServiceClass, true);
    Intent intent = new Intent(context, HeapAnalyzerService.class);
    intent.putExtra(LISTENER_CLASS_EXTRA, listenerServiceClass.getName());
    intent.putExtra(HEAPDUMP_EXTRA, heapDump);
    ContextCompat.startForegroundService(context, intent);
  }
```
这里启动了`HeapAnalyzerService`IntentService, 这个服务主要做的就是去解析我们的.hprof文件, 主要的工作内容在`onHandleIntentInForeground`方法内
``` java
@Override protected void onHandleIntentInForeground(@Nullable Intent intent) {
    if (intent == null) {
      CanaryLog.d("HeapAnalyzerService received a null intent, ignoring.");
      return;
    }
    String listenerClassName = intent.getStringExtra(LISTENER_CLASS_EXTRA);
    HeapDump heapDump = (HeapDump) intent.getSerializableExtra(HEAPDUMP_EXTRA);

    HeapAnalyzer heapAnalyzer =
        new HeapAnalyzer(heapDump.excludedRefs, this, heapDump.reachabilityInspectorClasses);

    AnalysisResult result = heapAnalyzer.checkForLeak(heapDump.heapDumpFile, heapDump.referenceKey,
        heapDump.computeRetainedHeapSize);
    AbstractAnalysisResultService.sendResultToListener(this, listenerClassName, heapDump, result);
  }
```
然后通过`haha`库, 将.hprof文件解析结果`AnalysisResult`对象, 通过`AbstractAnalysisResultService.sendResultToListener`传递启动`DisplayLeakService`服务.
``` java
@Override protected final void onHeapAnalyzed(HeapDump heapDump, AnalysisResult result) {
    String leakInfo = leakInfo(this, heapDump, result, true);
    CanaryLog.d("%s", leakInfo);

    boolean resultSaved = false;
    boolean shouldSaveResult = result.leakFound || result.failure != null;
    if (shouldSaveResult) {
      heapDump = renameHeapdump(heapDump);
      resultSaved = saveResult(heapDump, result);
    }

    PendingIntent pendingIntent;
    String contentTitle;
    String contentText;

    if (!shouldSaveResult) {
      contentTitle = getString(R.string.leak_canary_no_leak_title);
      contentText = getString(R.string.leak_canary_no_leak_text);
      pendingIntent = null;
    } else if (resultSaved) {
      pendingIntent = DisplayLeakActivity.createPendingIntent(this, heapDump.referenceKey);

      if (result.failure == null) {
        if (result.retainedHeapSize == AnalysisResult.RETAINED_HEAP_SKIPPED) {
          String className = classSimpleName(result.className);
          if (result.excludedLeak) {
            contentTitle = getString(R.string.leak_canary_leak_excluded, className);
          } else {
            contentTitle = getString(R.string.leak_canary_class_has_leaked, className);
          }
        } else {
          String size = formatShortFileSize(this, result.retainedHeapSize);
          String className = classSimpleName(result.className);
          if (result.excludedLeak) {
            contentTitle = getString(R.string.leak_canary_leak_excluded_retaining, className, size);
          } else {
            contentTitle =
                getString(R.string.leak_canary_class_has_leaked_retaining, className, size);
          }
        }
      } else {
        contentTitle = getString(R.string.leak_canary_analysis_failed);
      }
      contentText = getString(R.string.leak_canary_notification_message);
    } else {
      contentTitle = getString(R.string.leak_canary_could_not_save_title);
      contentText = getString(R.string.leak_canary_could_not_save_text);
      pendingIntent = null;
    }
    // New notification id every second.
    int notificationId = (int) (SystemClock.uptimeMillis() / 1000);
    showNotification(this, contentTitle, contentText, pendingIntent, notificationId);
    afterDefaultHandling(heapDump, result, leakInfo);
  }
```
将对应的.hprof文件重命名, 对应泄漏的内存对象关联key, 传递到`DisplayLeakActivity`做显示, 另外进行通知显示.

## 总结
通过上文的简易解析, 我们可以得出`LeakCanary`的一个大概的原理流程.通过application注册`ActivityLifecycleCallbacks`的回调, 在每个`activity`销毁的时候, 将`activity`的弱引用包装绑定在`ReferenceQueue`上, 当GC的时候, 可以通过`queue`移除已被回收的activity对象key, 获得始终未被回收的对象, 判断为是内存泄漏, 根据[haha库](https://github.com/square/haha)解析heap dumps,获取引用路径最终在`DisplayLeakActivity`上显示我们熟悉的内存泄漏的列表内容.
