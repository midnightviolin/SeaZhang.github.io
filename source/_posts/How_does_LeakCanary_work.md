---

title: LeakCanary 内存监测库工作原理简析
date: 2018-09-13 18:05:50
tags: [LeakCanary,内存泄漏,生命周期,监听,弱引用,heap dump,GC]

---



#### 关联要点：

Activity LifeCycle , WeakReference , ReferenceQueue , GC , heap dump



#### 原理：

##### 监听：

在应用安装后，会注册应用内所有 `Activity` 生命周期的监听。通过 `Application` 的`registerActivityLifecycleCallbacks` 方法,在 `onDestory` 方法回调设置 activity 的弱引用，并设置监听。

##### 检测：

通过 `WeakReference` 和 `ReferenceQueue` 来判断对象是否被系统 GC 回收。

1. 后台线程检测引用是否被清除，如果没有，调用 GC.
2. 如果 GC 后，引用还未被清除，做的将 heap 内存到 APP 对应文件系统中的一个 .hprof 文件中.
3. 在另一个进程中 `HeapAnalyzerSevice` 有一个 `HeapAnalyzer` 使用 haha 解析这个文件.
4. `HeapAnalyzer` 计算到 GC roots 的最短强引用路径，并确定是否是泄漏。如果是的话，建立导致泄漏的引用链.
5. 应用链传递到 App 进程中的 `DisplayLeakService`，并以通知形式展示出来。

<!-- more -->

#### 应用接入配置：

在 Application onCreate 方法中注册初始化操作：

```java
if (DebugConfig.DEBUG){
    LeakCanary.install(this);
}
```



#### 关联类及配置：

```java
内部主要相关类：
LeakCanary
RefWatcher
ActivityRefWatcher	其中有个 installOnIcsPlus(application,refWatcher) 方法
AndroidHeapDumper

```

```java
Mainfest.xml 配置服务 和 展示 Activity：

HeapAnalyzerService
DisplayLeakService
DisplayLeakActivity
```



#### 内部实现：

以下是库内部大致实现：

1、 应用启动时在 Application `onCreate` 方法中进行 初始化` LeakCanary.install(this)`.

2、 `LeanCanary` 类里面的 `install` 方法中，设置 `HeapDump` 监听器，实例化 `RefWatcher`.

3、 调用 `ActivityRefWatcher` 的静态方法 `installOnIcsPlus`，传入 `refWatcher` 实例 和 `application`.

4、 `installOnlcsPlus()` 方法内部会调用 `watchActivities()` 方法.

5、 `watchActivities()` 会先停止先前监听(如果有)，然后调用 application 的 `registerActivityLifecycleCallbacks` 注册应用生命周期监听。

6、 在应用生命周期监听的 `onActivityDestroyed(Activity activity) `中，调用 `refWatcher.watch(activity)` 方法.

7、 watch 方法内 会将 activity 是立即 包装在自定义的弱引用 `KeyedWeakReference` 中，同事指定一个 `ReferenceQueque` 队列. 当弱引用持有对象被回收时会进入该队列。

所以具体做法是在 activity `onDestory` 方法被调用时，将 activity 对象绑定在弱引用中，然后手动执行一次  GC，然后观察 `ReferenceQueue` 中是不是包含对应的 activity 对象，如果不包含，说明 activity 被强引用，也就发生内存泄漏.

8、 在另外一个进程中 `HeapAnalyzerSevice` 有一个 `HeapAnalyzer` 使用 [haha](https://github.com/square/haha) 解析这个文件.

9、 `HeapAnalyzer` 计算到` GC roots` 的最短强引用路径，并确定是否是泄漏。如果是的话，建立导致泄漏的引用链.

10、应用链传递到 App 进程中的 `DisplayLeakService`，并以通知形式展示出来。



#### 扩展

 下面大致列举下库部分扩展配置：

0、应用中针对其他有生命周期的组件，如 Fragment，Service，Dagger 组件等，可以使用 RefWatcher 监听对应的实例。

```java
RefWatcher refWatcher = LeakCanary.installedRefWatcher();

// We expect schrodingerCat to be gone soon (or not), let's watch it.
refWatcher.watch(schrodingerCat);

```

1、使用 no-op 空包.

2、自定义通知页面图标 icon 和 label 样式。

3、定义 heap dump 的数量(默认为 7)

```java
public class DebugExampleApplication extends ExampleApplication {
  @Override protected void installLeakCanary() {
    RefWatcher refWatcher = LeakCanary.refWatcher(this)
      .maxStoredHeapDumps(42)
      .buildAndInstall();
  }
}
```

4、增加文件服务上传

可以继承 DisplayLeakService，复写 afterDefaultHandling 方法，添加相关文件服务上传功能。

在 application 中添加相关服务配置

```java
public class DebugExampleApplication extends ExampleApplication {
  @Override protected void installLeakCanary() {
    RefWatcher refWatcher = LeakCanary.refWatcher(this)
      .listenerServiceClass(LeakUploadService.class);
      .buildAndInstall();
  }
}
```

另外需要在 Manifest.xml 中进行配置。

5、添加忽略引用检测(可以是 activity)

实例 ExcludedRefs ，进行忽略配置。

```java
public class DebugExampleApplication extends ExampleApplication {
  @Override protected void installLeakCanary() {
    ExcludedRefs excludedRefs = AndroidExcludedRefs.createAppDefaults()
        .instanceField("com.example.ExampleClass", "exampleField")
        .build();
    RefWatcher refWatcher = LeakCanary.refWatcher(this)
      .excludedRefs(excludedRefs)
      .buildAndInstall();
      
      //可以忽略指定的 activity
      registerActivityLifecycleCallbacks(new ActivityLifecycleCallbacks() {
      public void onActivityDestroyed(Activity activity) {
        if (activity instanceof ThirdPartyActivity) {
            return;
        }
        refWatcher.watch(activity);
      }
  }
}
```

6、可以设置运行时开关,通过继承复写 HeapDumper 

7、可以介入单元测试,通过 leakcanary-android-instrumentation.

如果最底层的Activity一直未走`onDestroy`生命周期(它在 Activity 栈的最底层)，无法检测出它的调用栈的内存泄漏。




#### 关联类内部的部分方法：

##### LeanCanary 类

```java
public final class LeakCanary {
    public static RefWatcher install(Application application) {
        return install(application, DisplayLeakService.class, AndroidExcludedRefs.createAppDefaults().build());
    }

    public static RefWatcher install(Application application, Class<? extends AbstractAnalysisResultService> listenerServiceClass, ExcludedRefs excludedRefs) {
        if(isInAnalyzerProcess(application)) {
            return RefWatcher.DISABLED;
        } else {
            enableDisplayLeakActivity(application);
            ServiceHeapDumpListener heapDumpListener = new ServiceHeapDumpListener(application, listenerServiceClass);
            RefWatcher refWatcher = androidWatcher(application, heapDumpListener, excludedRefs);
            ActivityRefWatcher.installOnIcsPlus(application, refWatcher);
            return refWatcher;
        }
    }

    public static RefWatcher androidWatcher(Context context, Listener heapDumpListener, ExcludedRefs excludedRefs) {
        AndroidDebuggerControl debuggerControl = new AndroidDebuggerControl();
        AndroidHeapDumper heapDumper = new AndroidHeapDumper(context);
        heapDumper.cleanup();
        return new RefWatcher(new AndroidWatchExecutor(), debuggerControl, GcTrigger.DEFAULT, heapDumper, heapDumpListener, excludedRefs);
    }

    public static void enableDisplayLeakActivity(Context context) {
        LeakCanaryInternals.setEnabled(context, DisplayLeakActivity.class, true);
    }
    
    ... 其他部分代码
}
```



##### ActivityRefWatcher 类：

```java
@TargetApi(14)
public final class ActivityRefWatcher {
    private final ActivityLifecycleCallbacks lifecycleCallbacks = new ActivityLifecycleCallbacks() {
        public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
        }

        public void onActivityStarted(Activity activity) {
        }

        public void onActivityResumed(Activity activity) {
        }

        public void onActivityPaused(Activity activity) {
        }

        public void onActivityStopped(Activity activity) {
        }

        public void onActivitySaveInstanceState(Activity activity, Bundle outState) {
        }

        public void onActivityDestroyed(Activity activity) {
            ActivityRefWatcher.this.onActivityDestroyed(activity);
        }
    };
    private final Application application;
    private final RefWatcher refWatcher;

    public static void installOnIcsPlus(Application application, RefWatcher refWatcher) {
        if(VERSION.SDK_INT >= 14) {
            ActivityRefWatcher activityRefWatcher = new ActivityRefWatcher(application, refWatcher);
            activityRefWatcher.watchActivities();
        }
    }

    public ActivityRefWatcher(Application application, RefWatcher refWatcher) {
        this.application = (Application)Preconditions.checkNotNull(application, "application");
        this.refWatcher = (RefWatcher)Preconditions.checkNotNull(refWatcher, "refWatcher");
    }

    void onActivityDestroyed(Activity activity) {
        this.refWatcher.watch(activity);
    }

    public void watchActivities() {
        this.stopWatchingActivities();
        this.application.registerActivityLifecycleCallbacks(this.lifecycleCallbacks);
    }

    public void stopWatchingActivities() {
        this.application.unregisterActivityLifecycleCallbacks(this.lifecycleCallbacks);
    }
}
```



##### RefWatcher 类：

```java
 public RefWatcher(Executor watchExecutor, DebuggerControl debuggerControl, GcTrigger gcTrigger, HeapDumper heapDumper, Listener heapdumpListener, ExcludedRefs excludedRefs) {
        this.watchExecutor = (Executor)Preconditions.checkNotNull(watchExecutor, "watchExecutor");
        this.debuggerControl = (DebuggerControl)Preconditions.checkNotNull(debuggerControl, "debuggerControl");
        this.gcTrigger = (GcTrigger)Preconditions.checkNotNull(gcTrigger, "gcTrigger");
        this.heapDumper = (HeapDumper)Preconditions.checkNotNull(heapDumper, "heapDumper");
        this.heapdumpListener = (Listener)Preconditions.checkNotNull(heapdumpListener, "heapdumpListener");
        this.excludedRefs = (ExcludedRefs)Preconditions.checkNotNull(excludedRefs, "excludedRefs");
        this.retainedKeys = new CopyOnWriteArraySet();
        this.queue = new ReferenceQueue();
    }

    public void watch(Object watchedReference) {
        this.watch(watchedReference, "");
    }

    public void watch(Object watchedReference, String referenceName) {
        Preconditions.checkNotNull(watchedReference, "watchedReference");
        Preconditions.checkNotNull(referenceName, "referenceName");
        if(!this.debuggerControl.isDebuggerAttached()) {
            final long watchStartNanoTime = System.nanoTime();
            String key = UUID.randomUUID().toString();
            this.retainedKeys.add(key);
            final KeyedWeakReference reference = new KeyedWeakReference(watchedReference, key, referenceName, this.queue);
            this.watchExecutor.execute(new Runnable() {
                public void run() {
                    RefWatcher.this.ensureGone(reference, watchStartNanoTime);
                }
            });
        }
    }

    void ensureGone(KeyedWeakReference reference, long watchStartNanoTime) {
        long gcStartNanoTime = System.nanoTime();
        long watchDurationMs = TimeUnit.NANOSECONDS.toMillis(gcStartNanoTime - watchStartNanoTime);
        this.removeWeaklyReachableReferences();
        if(!this.gone(reference) && !this.debuggerControl.isDebuggerAttached()) {
            this.gcTrigger.runGc();
            this.removeWeaklyReachableReferences();
            if(!this.gone(reference)) {
                long startDumpHeap = System.nanoTime();
                long gcDurationMs = TimeUnit.NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);
                File heapDumpFile = this.heapDumper.dumpHeap();
                if(heapDumpFile == HeapDumper.NO_DUMP) {
                    return;
                }

                long heapDumpDurationMs = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);
                this.heapdumpListener.analyze(new HeapDump(heapDumpFile, reference.key, reference.name, this.excludedRefs, watchDurationMs, gcDurationMs, heapDumpDurationMs));
            }

        }
    }
```







#### 参考文章链接：

[LeakCanary FAQ](https://github.com/square/leakcanary/wiki/FAQ#)

[Customizing LeakCanary](https://github.com/square/leakcanary/wiki/Customizing-LeakCanary)

[LeakCanary原理解析](https://blog.csdn.net/xiaohanluo/article/details/78196755)

[LeakCanary工作原理](https://ivanljt.github.io/blog/2017/12/15/%E6%8B%86%E8%BD%AE%E5%AD%90%E7%B3%BB%E5%88%97%E2%80%94%E2%80%94LeakCanary%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86/)

[深入理解 Android 之 LeakCanary 源码解析](https://allenwu.itscoder.com/leakcanary-source)

