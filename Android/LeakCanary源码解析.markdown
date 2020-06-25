# LeakCanary源码解析



## 概述

`LeakCanary`是Android开发中非常常用的一个内存泄漏监测和分析工具。了解其工作原理，有助于对Android的内存泄漏有更深层次的认识。

## 源码分析

```java
  protected void setupLeakCanary() {
    enabledStrictMode();
    if (LeakCanary.isInAnalyzerProcess(this)) {
      // This process is dedicated to LeakCanary for heap analysis.
      // You should not init your app in this process.
      return;
    }
    LeakCanary.install(this);
  }
```

### isInAnalyzerProcess

先看分析进程判断的方法

```java
  /**
   * Whether the current process is the process running the {@link HeapAnalyzerService}, which is
   * a different process than the normal app process.
   */
  public static boolean isInAnalyzerProcess(@NonNull Context context) {
    Boolean isInAnalyzerProcess = LeakCanaryInternals.isInAnalyzerProcess;
    // This only needs to be computed once per process.
    if (isInAnalyzerProcess == null) {
      isInAnalyzerProcess = isInServiceProcess(context, HeapAnalyzerService.class);
      LeakCanaryInternals.isInAnalyzerProcess = isInAnalyzerProcess;
    }
    return isInAnalyzerProcess;
  }
```

关键是判断`HeapAnalyzerService`是否在进程中。

```java
  public static boolean isInServiceProcess(Context context, Class<? extends Service> serviceClass) {
    PackageManager packageManager = context.getPackageManager();
    PackageInfo packageInfo;
    try {
      packageInfo = packageManager.getPackageInfo(context.getPackageName(), GET_SERVICES);
    } catch (Exception e) {
      CanaryLog.d(e, "Could not get package info for %s", context.getPackageName());
      return false;
    }
    String mainProcess = packageInfo.applicationInfo.processName;

    ComponentName component = new ComponentName(context, serviceClass);
    ServiceInfo serviceInfo;
    try {
      serviceInfo = packageManager.getServiceInfo(component, PackageManager.GET_DISABLED_COMPONENTS);
    } catch (PackageManager.NameNotFoundException ignored) {
      // Service is disabled.
      return false;
    }

    if (serviceInfo.processName.equals(mainProcess)) {
      CanaryLog.d("Did not expect service %s to run in main process %s", serviceClass, mainProcess);
      // Technically we are in the service process, but we're not in the service dedicated process.
      return false;
    }

    int myPid = android.os.Process.myPid();
    ActivityManager activityManager =
        (ActivityManager) context.getSystemService(Context.ACTIVITY_SERVICE);
    ActivityManager.RunningAppProcessInfo myProcess = null;
    List<ActivityManager.RunningAppProcessInfo> runningProcesses;
    try {
      runningProcesses = activityManager.getRunningAppProcesses();
    } catch (SecurityException exception) {
      // https://github.com/square/leakcanary/issues/948
      CanaryLog.d("Could not get running app processes %d", exception);
      return false;
    }
    if (runningProcesses != null) {
      for (ActivityManager.RunningAppProcessInfo process : runningProcesses) {
        if (process.pid == myPid) {
          myProcess = process;
          break;
        }
      }
    }
    if (myProcess == null) {
      CanaryLog.d("Could not find running process for %d", myPid);
      return false;
    }

    return myProcess.processName.equals(serviceInfo.processName);
  }
```

这个方法比较长，但其实只做了两个判断：
1. 判断`HeapAnalyzerService`与App主进程名是否相等
2. 判断`HeapAnalyzerService`与当前进程名是否相等

### install

```java
  /**
   * Creates a {@link RefWatcher} that works out of the box, and starts watching activity
   * references (on ICS+).
   */
  public static @NonNull RefWatcher install(@NonNull Application application) {
    return refWatcher(application).listenerServiceClass(DisplayLeakService.class)
        .excludedRefs(AndroidExcludedRefs.createAppDefaults().build())
        .buildAndInstall();
  }
```

从方法结构上看，这个方法是要返回一个`RefWatcher`。它是由`RefWatcherBuilder`调用`buildAndInstall`方法生成的。

那先看一下`AndroidRefWatcherBuilder`的源码：

```java
package com.squareup.leakcanary;

import android.content.Context;
import android.support.annotation.NonNull;
import com.squareup.leakcanary.internal.DisplayLeakActivity;
import com.squareup.leakcanary.internal.FragmentRefWatcher;
import com.squareup.leakcanary.internal.LeakCanaryInternals;
import java.util.List;
import java.util.concurrent.TimeUnit;

import static com.squareup.leakcanary.RefWatcher.DISABLED;
import static java.util.concurrent.TimeUnit.SECONDS;

/** A {@link RefWatcherBuilder} with appropriate Android defaults. */
public final class AndroidRefWatcherBuilder extends RefWatcherBuilder<AndroidRefWatcherBuilder> {

  private static final long DEFAULT_WATCH_DELAY_MILLIS = SECONDS.toMillis(5);

  private final Context context;
  private boolean watchActivities = true;
  private boolean watchFragments = true;
  private boolean enableDisplayLeakActivity = false;

  AndroidRefWatcherBuilder(@NonNull Context context) {
    this.context = context.getApplicationContext();
  }

  /**
   * Sets a custom {@link AbstractAnalysisResultService} to listen to analysis results. This
   * overrides any call to {@link #heapDumpListener(HeapDump.Listener)}.
   */
  public @NonNull AndroidRefWatcherBuilder listenerServiceClass(
      @NonNull Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
    enableDisplayLeakActivity = DisplayLeakService.class.isAssignableFrom(listenerServiceClass);
    return heapDumpListener(new ServiceHeapDumpListener(context, listenerServiceClass));
  }

  /**
   * Sets a custom delay for how long the {@link RefWatcher} should wait until it checks if a
   * tracked object has been garbage collected. This overrides any call to {@link
   * #watchExecutor(WatchExecutor)}.
   */
  public @NonNull AndroidRefWatcherBuilder watchDelay(long delay, @NonNull TimeUnit unit) {
    return watchExecutor(new AndroidWatchExecutor(unit.toMillis(delay)));
  }

  /**
   * Whether we should automatically watch activities when calling {@link #buildAndInstall()}.
   * Default is true.
   */
  public @NonNull AndroidRefWatcherBuilder watchActivities(boolean watchActivities) {
    this.watchActivities = watchActivities;
    return this;
  }

  /**
   * Whether we should automatically watch fragments when calling {@link #buildAndInstall()}.
   * Default is true. When true, LeakCanary watches native fragments on Android O+ and support
   * fragments if the leakcanary-support-fragment dependency is in the classpath.
   */
  public @NonNull AndroidRefWatcherBuilder watchFragments(boolean watchFragments) {
    this.watchFragments = watchFragments;
    return this;
  }

  /**
   * Sets the maximum number of heap dumps stored. This overrides any call to
   * {@link LeakCanary#setLeakDirectoryProvider(LeakDirectoryProvider)}
   *
   * @throws IllegalArgumentException if maxStoredHeapDumps < 1.
   */
  public @NonNull AndroidRefWatcherBuilder maxStoredHeapDumps(int maxStoredHeapDumps) {
    LeakDirectoryProvider leakDirectoryProvider =
        new DefaultLeakDirectoryProvider(context, maxStoredHeapDumps);
    LeakCanary.setLeakDirectoryProvider(leakDirectoryProvider);
    return self();
  }

  /**
   * Creates a {@link RefWatcher} instance and makes it available through {@link
   * LeakCanary#installedRefWatcher()}.
   *
   * Also starts watching activity references if {@link #watchActivities(boolean)} was set to true.
   *
   * @throws UnsupportedOperationException if called more than once per Android process.
   */
  public @NonNull RefWatcher buildAndInstall() {
    if (LeakCanaryInternals.installedRefWatcher != null) {
      throw new UnsupportedOperationException("buildAndInstall() should only be called once.");
    }
    RefWatcher refWatcher = build();
    if (refWatcher != DISABLED) {
      if (enableDisplayLeakActivity) {
        LeakCanaryInternals.setEnabledAsync(context, DisplayLeakActivity.class, true);
      }
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

  @Override protected boolean isDisabled() {
    return LeakCanary.isInAnalyzerProcess(context);
  }

  @Override protected @NonNull HeapDumper defaultHeapDumper() {
    LeakDirectoryProvider leakDirectoryProvider =
        LeakCanaryInternals.getLeakDirectoryProvider(context);
    return new AndroidHeapDumper(context, leakDirectoryProvider);
  }

  @Override protected @NonNull DebuggerControl defaultDebuggerControl() {
    return new AndroidDebuggerControl();
  }

  @Override protected @NonNull HeapDump.Listener defaultHeapDumpListener() {
    return new ServiceHeapDumpListener(context, DisplayLeakService.class);
  }

  @Override protected @NonNull ExcludedRefs defaultExcludedRefs() {
    return AndroidExcludedRefs.createAppDefaults().build();
  }

  @Override protected @NonNull WatchExecutor defaultWatchExecutor() {
    return new AndroidWatchExecutor(DEFAULT_WATCH_DELAY_MILLIS);
  }

  @Override protected @NonNull
  List<Class<? extends Reachability.Inspector>> defaultReachabilityInspectorClasses() {
    return AndroidReachabilityInspectors.defaultAndroidInspectors();
  }
}

```

大致扫一眼源码可知，这是一个`RefWatcher`的Builder类。最后通过`buildAndInstall`方法生成最后的`RefWatcher`：
```java
  /**
   * Creates a {@link RefWatcher} instance and makes it available through {@link
   * LeakCanary#installedRefWatcher()}.
   *
   * Also starts watching activity references if {@link #watchActivities(boolean)} was set to true.
   *
   * @throws UnsupportedOperationException if called more than once per Android process.
   */
  public @NonNull RefWatcher buildAndInstall() {
    if (LeakCanaryInternals.installedRefWatcher != null) {
      throw new UnsupportedOperationException("buildAndInstall() should only be called once.");
    }
    RefWatcher refWatcher = build();
    if (refWatcher != DISABLED) {
      if (enableDisplayLeakActivity) {
        LeakCanaryInternals.setEnabledAsync(context, DisplayLeakActivity.class, true);
      }
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
这个方法中，调用`build`方法生成`RefWatcher`，初始化`DisplayLeakActivity`类，然后将其传入`ActivityRefWatcher`和`FragmentRefWatcher`中。

我们就分别看一下，这几个类是如何工作的。

---

### RefWatcher

* watchExecutor 确保分析任务操作是在主线程进行的，同时默认延迟5秒执行分析任务，留时间给系统GC
* debuggerControl debug控制中心
* gcTrigger 内部调用Runtime.getRuntime().gc()，手动触发系统GC
* heapDumper 用于创建.hprof文件，用于储存heap堆的快照，可以获知程序的哪些部分正在使用大部分的内存
* heapDumpListener 分析结果完成后，会告诉这个监听者
* heapDumpBuilder
* retainedKeys
* queue 存放被标记gc的对象的引用，当LeakCanary认为对象已经被回收，但在queue中未发现对象的引用时，认为泄漏。

#### RefWatcher.watch

watch是`RefWatcher`的核心方法。当我们觉得对象应该被销毁时，调用它。用来确定它是否真的被销毁。在监听Activity的场景中，我们就是在Activity的onDestroy生命周期中，调用`watch`方法，并传入Activity对象。

```java
  /**
   * Identical to {@link #watch(Object, String)} with an empty string reference name.
   *
   * @see #watch(Object, String)
   */
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

`watch`方法中，会生成随机的key，存入`retainedKeys`中，然后生成`KeyedWeakReference`对象，然后调用`ensureGoneAsync`方法。
其中，`KeyedWeakReference`就是加了一个key和name的弱引用。
```java
/** @see {@link HeapDump#referenceKey}. */
final class KeyedWeakReference extends WeakReference<Object> {
  public final String key;
  public final String name;

  KeyedWeakReference(Object referent, String key, String name,
      ReferenceQueue<Object> referenceQueue) {
    super(checkNotNull(referent, "referent"), checkNotNull(referenceQueue, "referenceQueue"));
    this.key = checkNotNull(key, "key");
    this.name = checkNotNull(name, "name");
  }
}
```
这里值得注意的是，我们传入了`referenceQueue`。这表明，当对象被标记GC时，会将对象加入到我们`RefWatcher`的`queue`中。

#### RefWatcher.ensureGone
```java
  private void ensureGoneAsync(final long watchStartNanoTime, final KeyedWeakReference reference) {
    watchExecutor.execute(new Retryable() {
      @Override public Retryable.Result run() {
        return ensureGone(reference, watchStartNanoTime);
      }
    });
  }

  @SuppressWarnings("ReferenceEquality") // Explicitly checking for named null.
  Retryable.Result ensureGone(final KeyedWeakReference reference, final long watchStartNanoTime) {
    long gcStartNanoTime = System.nanoTime();
    long watchDurationMs = NANOSECONDS.toMillis(gcStartNanoTime - watchStartNanoTime);

    removeWeaklyReachableReferences();

    if (debuggerControl.isDebuggerAttached()) {
      // The debugger can create false leaks.
      return RETRY;
    }
    if (gone(reference)) {
      return DONE;
    }
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

这一段逻辑比较关键，我们分段来看。
首先我们要清楚，我们调用这个方法时，是在Activity的onDestroy中，此时，retainedKeys中，有Activity的key，queue中是否有Activity的对象是未知的。因为`ensureGone`是在新线程调用的。

`removeWeaklyReachableReferences`方法是清理掉，存在于`queue`中的对象，在`retainedKeys`中的key。我们在onDestroy刚开始时，将对象的key放入了`retainedKey`中，如果这次清理，将它清理掉了，则说明对象也存在于`queue`中了，也就是说明对象被标记回收了。这一段的逻辑在代码中就是：
```java
  Retryable.Result ensureGone(final KeyedWeakReference reference, final long watchStartNanoTime) {
    long gcStartNanoTime = System.nanoTime();
    long watchDurationMs = NANOSECONDS.toMillis(gcStartNanoTime - watchStartNanoTime);

    removeWeaklyReachableReferences();

    if (debuggerControl.isDebuggerAttached()) {
      // The debugger can create false leaks.
      return RETRY;
    }
    if (gone(reference)) {
      return DONE;
    }
    ...
}
```
其中，`gone`方法就是一个简单的判断：
```java
  private boolean gone(KeyedWeakReference reference) {
    return !retainedKeys.contains(reference.key);
  }
```
如果这时候发现，Activity的key还存在于`retainedKeys`中，则说明Activity并没有在`queue`中，则说明Activity还没有被标记回收。那么，这时候，是不是说明Activity已经泄漏了呢？未必！

有可能此时Activity虽然与GCRoot没有联系了，但GC尚未启动。所以它的弱引用，并未被放至`queue`中。

我们看接下来的代码：
```java
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
```

```java
/**
 * Called when a watched reference is expected to be weakly reachable, but hasn't been enqueued
 * in the reference queue yet. This gives the application a hook to run the GC before the {@link
 * RefWatcher} checks the reference queue again, to avoid taking a heap dump if possible.
 */
public interface GcTrigger {
  GcTrigger DEFAULT = new GcTrigger() {
    @Override public void runGc() {
      // Code taken from AOSP FinalizationTest:
      // https://android.googlesource.com/platform/libcore/+/master/support/src/test/java/libcore/
      // java/lang/ref/FinalizationTester.java
      // System.gc() does not garbage collect every time. Runtime.gc() is
      // more likely to perform a gc.
      Runtime.getRuntime().gc();
      enqueueReferences();
      System.runFinalization();
    }

    private void enqueueReferences() {
      // Hack. We don't have a programmatic way to wait for the reference queue daemon to move
      // references to the appropriate queues.
      try {
        Thread.sleep(100);
      } catch (InterruptedException e) {
        throw new AssertionError();
      }
    }
  };

  void runGc();
}
```

接下来，LeakCanary强制调用了一次GC，再做了一次检查。此时，如果对象引用不在`queue`中，则表示对象已成功释放。如果对象引用依然未处于`queue`中，则需要对`dumpHeap`进行分析了。通过对`dumpHeap`进行分析，得出最终是否泄漏的结论。`dumpHeap`的分析比较复杂，放在最后说。我们可以先看一下，LeakCanary是如何监测Activity和Fragment的泄漏的。

### ActivityRefWatcher

Activity的监测比较简单。就是注册Activity生命周期的回调，然后在onActivityDestroyed方法中，开始watch需要被回收的Activity对象。

```java
public final class ActivityRefWatcher {

  public static void installOnIcsPlus(@NonNull Application application,
      @NonNull RefWatcher refWatcher) {
    install(application, refWatcher);
  }

  public static void install(@NonNull Context context, @NonNull RefWatcher refWatcher) {
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
    ...
}
```

### FragmentRefWatcher
FragmentRefWatcher比ActivityRefWatcher稍微复杂一点，主要涉及版本兼容问题。这里不贴代码了，大致思路就是，在Activity的`onActivityCreated`中，获取它的FragmentManager，然后监听Fragment的`onFragmentViewDestroyed`和`onFragmentDestroyed`

---

### dump分析

刚刚说到，即使我们gc之后，还发现对象并未出来在`RefWatcher`的`queue`中，也不能说明对象已经泄露了，此时我们依然需要对对象进行判断。下面我们看看`dump分析`的代码：
```java
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
```

我们先看一下dump分析的大致流程。
1. 获取 headDumpFile 文件
2. 生成 heapDump 
3. 分析 heapDump

#### 获取headDumpFile

**AndroidHeapDumper.java**
```java
  public File dumpHeap() {
    File heapDumpFile = leakDirectoryProvider.newHeapDumpFile();

    if (heapDumpFile == RETRY_LATER) {
      return RETRY_LATER;
    }

    FutureResult<Toast> waitingForToast = new FutureResult<>();
    showToast(waitingForToast);

    if (!waitingForToast.wait(5, SECONDS)) {
      CanaryLog.d("Did not dump heap, too much time waiting for Toast.");
      return RETRY_LATER;
    }

    Notification.Builder builder = new Notification.Builder(context)
        .setContentTitle(context.getString(R.string.leak_canary_notification_dumping));
    Notification notification = LeakCanaryInternals.buildNotification(context, builder);
    NotificationManager notificationManager =
        (NotificationManager) context.getSystemService(Context.NOTIFICATION_SERVICE);
    int notificationId = (int) SystemClock.uptimeMillis();
    notificationManager.notify(notificationId, notification);

    Toast toast = waitingForToast.get();
    try {
      Debug.dumpHprofData(heapDumpFile.getAbsolutePath());
      cancelToast(toast);
      notificationManager.cancel(notificationId);
      return heapDumpFile;
    } catch (Exception e) {
      CanaryLog.d(e, "Could not dump heap");
      // Abort heap dump
      return RETRY_LATER;
    }
  }
```
这个方法有很多逻辑是为了通知，真正的核心分析逻辑只有两行：
```java
    File heapDumpFile = leakDirectoryProvider.newHeapDumpFile();
    ...
    Debug.dumpHprofData(heapDumpFile.getAbsolutePath());
```
第一行是创建一个新的`heapDumpFile`，后面第二行`Debug.dumpHprofData`的作用是抓取内存信息，写入`heapDumpFile`中。


#### 生成 heapDump 
HeapDump中本身没有什么逻辑，主要是封装了HeapDump的各种参数。
**HeapDump.java**
```java
/** Data structure holding information about a heap dump. */
public final class HeapDump implements Serializable {

  public static Builder builder() {
    return new Builder();
  }

  /** Receives a heap dump to analyze. */
  public interface Listener {
    Listener NONE = new Listener() {
      @Override public void analyze(HeapDump heapDump) {
      }
    };

    void analyze(HeapDump heapDump);
  }

  /** The heap dump file, which you might want to upload somewhere. */
  public final File heapDumpFile;

  /**
   * Key associated to the {@link KeyedWeakReference} used to detect the memory leak.
   * When analyzing a heap dump, search for all {@link KeyedWeakReference} instances, then open
   * the one that has its "key" field set to this value. Its "referent" field contains the
   * leaking object. Computing the shortest path to GC roots on that leaking object should enable
   * you to figure out the cause of the leak.
   */
  public final String referenceKey;

  /**
   * User defined name to help identify the leaking instance.
   */
  public final String referenceName;

  /** References that should be ignored when analyzing this heap dump. */
  public final ExcludedRefs excludedRefs;

  /** Time from the request to watch the reference until the GC was triggered. */
  public final long watchDurationMs;
  public final long gcDurationMs;
  public final long heapDumpDurationMs;
  public final boolean computeRetainedHeapSize;
  public final List<Class<? extends Reachability.Inspector>> reachabilityInspectorClasses;
  ...
}
```
值得注意的是HeapDump是一个可序列化的类。

#### 分析 heapDump

heapDump的分析，主要在`ServiceHeapDumpListener`中完成
**ServiceHeapDumpListener.java**
```java
public final class ServiceHeapDumpListener implements HeapDump.Listener {

  private final Context context;
  private final Class<? extends AbstractAnalysisResultService> listenerServiceClass;

  public ServiceHeapDumpListener(@NonNull final Context context,
      @NonNull final Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
    this.listenerServiceClass = checkNotNull(listenerServiceClass, "listenerServiceClass");
    this.context = checkNotNull(context, "context").getApplicationContext();
  }

  @Override public void analyze(@NonNull HeapDump heapDump) {
    checkNotNull(heapDump, "heapDump");
    HeapAnalyzerService.runAnalysis(context, heapDump, listenerServiceClass);
  }
}

```
**HeapAnalyzerService.java**

```java
public final class HeapAnalyzerService extends ForegroundService
    implements AnalyzerProgressListener {

  private static final String LISTENER_CLASS_EXTRA = "listener_class_extra";
  private static final String HEAPDUMP_EXTRA = "heapdump_extra";

  public static void runAnalysis(Context context, HeapDump heapDump,
      Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
    setEnabledBlocking(context, HeapAnalyzerService.class, true);
    setEnabledBlocking(context, listenerServiceClass, true);
    Intent intent = new Intent(context, HeapAnalyzerService.class);
    intent.putExtra(LISTENER_CLASS_EXTRA, listenerServiceClass.getName());
    intent.putExtra(HEAPDUMP_EXTRA, heapDump);
    ContextCompat.startForegroundService(context, intent);
  }
  ...
}
```
我们可以看到，对于`heapDump`的分析，是在一个HeapAnalyzerService中实现的。我们会将`heapDump`作为一个参数传入到这个HeapAnalyzerService中。

```java
public final class HeapAnalyzerService extends ForegroundService
    implements AnalyzerProgressListener {
  ...

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
  ...
}
```
我们可以看到，对于`heapDump`的分析，委托给了`HeapAnalyzer`去完成。我们继续看`HeapAnalyzer`的代码：
```java
  /**
   * Searches the heap dump for a {@link KeyedWeakReference} instance with the corresponding key,
   * and then computes the shortest strong reference path from that instance to the GC roots.
   */
  public @NonNull AnalysisResult checkForLeak(@NonNull File heapDumpFile,
      @NonNull String referenceKey,
      boolean computeRetainedSize) {
    long analysisStartNanoTime = System.nanoTime();

    if (!heapDumpFile.exists()) {
      Exception exception = new IllegalArgumentException("File does not exist: " + heapDumpFile);
      return failure(exception, since(analysisStartNanoTime));
    }

    try {
      listener.onProgressUpdate(READING_HEAP_DUMP_FILE);
      HprofBuffer buffer = new MemoryMappedFileBuffer(heapDumpFile);
      HprofParser parser = new HprofParser(buffer);
      listener.onProgressUpdate(PARSING_HEAP_DUMP);
      Snapshot snapshot = parser.parse();
      listener.onProgressUpdate(DEDUPLICATING_GC_ROOTS);
      deduplicateGcRoots(snapshot);
      listener.onProgressUpdate(FINDING_LEAKING_REF);
      Instance leakingRef = findLeakingReference(referenceKey, snapshot);

      // False alarm, weak reference was cleared in between key check and heap dump.
      if (leakingRef == null) {
        return noLeak("UnknownNoKeyedWeakReference", since(analysisStartNanoTime));
      }
      return findLeakTrace(analysisStartNanoTime, snapshot, leakingRef, computeRetainedSize);
    } catch (Throwable e) {
      return failure(e, since(analysisStartNanoTime));
    }
  }
```

其中核心步骤是：
1. 通过 heapDumpFile生成HprofParser
2. HprofParser解析获得Snapshot
3. deduplicateGcRoots去掉重复的gc节点
4. findLeakingReference，从Snapshot中获得泄露的引用
5. findLeakTrace找到内存泄露的最短强引用路径

以上，就是LeakCanary基本检测流程。

## 总结

LeakCanary对对象泄漏的监听，当对象需要销毁时，大致可以分为三步：
1. 通过弱引用队列，看对象是否被回收
2. 强制gc后，再次通过弱引用队列，看对象是否被回收
3. 通过haha库对内存进行快照分析，判断是否被回收

但要真正理解`LeakCanary`的工作原理，还需要对`haha`库进行分析。才能够真正地了解`LeakCanary`是如何工作的。
