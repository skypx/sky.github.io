---
layout: post
title:  Android 开机启动流程SystemServer下篇(三)
category: Android
tags: SystemServer
---
* content
{:toc}

源码  
```java
</frameworks/base/services/java/com/android/server/SystemServer.java>
```
上面的文章中我们知道了怎么通过zygote来启动SystemServer进程，接下来我们来具体分析SystemServer里面的细节

###### 3.1 main
在zygote里面会通过反射来调用到main方法
```java
public static void main(String[] args) {
294        new SystemServer().run();
295    }
```

###### 3.2 SystemServer run
```java
private void run() {
308        try {
309            traceBeginAndSlog("InitBeforeStartServices");
310            // If a device's clock is before 1970 (before 0), a lot of
311            // APIs crash dealing with negative numbers, notably
312            // java.io.File#setLastModified, so instead we fake it and
313            // hope that time from cell towers or NTP fixes it shortly.
               如果时间比1970年早, 设置为1970年
314            if (System.currentTimeMillis() < EARLIEST_SUPPORTED_TIME) {
315                Slog.w(TAG, "System clock is before 1970; setting to 1970.");
316                SystemClock.setCurrentTimeMillis(EARLIEST_SUPPORTED_TIME);
317            }
318
319            //
320            // Default the timezone property to GMT if not set.
321            //
322            String timezoneProperty =  SystemProperties.get("persist.sys.timezone");
323            if (timezoneProperty == null || timezoneProperty.isEmpty()) {
324                Slog.w(TAG, "Timezone not set; setting to GMT.");
325                SystemProperties.set("persist.sys.timezone", "GMT");
326            }
327
328            // If the system has "persist.sys.language" and friends set, replace them with
329            // "persist.sys.locale". Note that the default locale at this point is calculated
330            // using the "-Duser.locale" command line flag. That flag is usually populated by
331            // AndroidRuntime using the same set of system properties, but only the system_server
332            // and system apps are allowed to set them.
333            //
334            // NOTE: Most changes made here will need an equivalent change to
335            // core/jni/AndroidRuntime.cpp
336            if (!SystemProperties.get("persist.sys.language").isEmpty()) {
337                final String languageTag = Locale.getDefault().toLanguageTag();
338
339                SystemProperties.set("persist.sys.locale", languageTag);
340                SystemProperties.set("persist.sys.language", "");
341                SystemProperties.set("persist.sys.country", "");
342                SystemProperties.set("persist.sys.localevar", "");
343            }
344
345            // The system server should never make non-oneway calls
346            Binder.setWarnOnBlocking(true);
347            // The system server should always load safe labels
348            PackageItemInfo.setForceSafeLabels(true);
349            // Deactivate SQLiteCompatibilityWalFlags until settings provider is initialized
350            SQLiteCompatibilityWalFlags.init(null);
351
352            // Here we go!
353            Slog.i(TAG, "Entered the Android system server!");
354            int uptimeMillis = (int) SystemClock.elapsedRealtime();
355            EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_SYSTEM_RUN, uptimeMillis);
356            if (!mRuntimeRestart) {
357                MetricsLogger.histogram(null, "boot_system_server_init", uptimeMillis);
358            }
359
360            // In case the runtime switched since last boot (such as when
361            // the old runtime was removed in an OTA), set the system
362            // property so that it is in sync. We can | xq oqi't do this in
363            // libnativehelper's JniInvocation::Init code where we already
364            // had to fallback to a different runtime because it is
365            // running as root and we need to be the system user to set
366            // the property. http://b/11463182
367            SystemProperties.set("persist.sys.dalvik.vm.lib.2", VMRuntime.getRuntime().vmLibrary());
368
369            // Mmmmmm... more memory!
370            VMRuntime.getRuntime().clearGrowthLimit();
371
372            // The system server has to run all of the time, so it needs to be
373            // as efficient as possible with its memory usage.
374            VMRuntime.getRuntime().setTargetHeapUtilization(0.8f);
375
376            // Some devices rely on runtime fingerprint generation, so make sure
377            // we've defined it before booting further.
378            Build.ensureFingerprintProperty();
379
380            // Within the system server, it is an error to access Environment paths without
381            // explicitly specifying a user.
382            Environment.setUserRequired(true);
383
384            // Within the system server, any incoming Bundles should be defused
385            // to avoid throwing BadParcelableException.
386            BaseBundle.setShouldDefuse(true);
387
388            // Within the system server, when parceling exceptions, include the stack trace
389            Parcel.setStackTraceParceling(true);
390
391            // Ensure binder calls into the system always run at foreground priority.
392            BinderInternal.disableBackgroundScheduling(true);
393
394            // Increase the number of binder threads in system_server
395            BinderInternal.setMaxThreads(sMaxBinderThreads);
396
397            // Prepare the main looper thread (this thread).
398            android.os.Process.setThreadPriority(
399                android.os.Process.THREAD_PRIORITY_FOREGROUND);
400            android.os.Process.setCanSelfBackground(false);
401            Looper.prepareMainLooper();
402            Looper.getMainLooper().setSlowLogThresholdMs(
403                    SLOW_DISPATCH_THRESHOLD_MS, SLOW_DELIVERY_THRESHOLD_MS);
404
405            // Initialize native services.
406            System.loadLibrary("android_servers");
407
408            // Check whether we failed to shut down last time we tried.
409            // This call may not return.
410            performPendingShutdown();
411
412            // Initialize the system context.
               //创建系统的上下文
413            createSystemContext();
414
415            // Create the system service manager.
               //系统服务管理者
416            mSystemServiceManager = new SystemServiceManager(mSystemContext);
417            mSystemServiceManager.setStartInfo(mRuntimeRestart,
418                    mRuntimeStartElapsedTime, mRuntimeStartUptime);
419            LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
420            // Prepare the thread pool for init tasks that can be parallelized
421            SystemServerInitThreadPool.get();
422        } finally {
423            traceEnd();  // InitBeforeStartServices
424        }
425
426        // Start services.
427        try {
428            traceBeginAndSlog("StartServices");
429            startBootstrapServices(); // 启动引导服务
430            startCoreServices(); //核心服务
431            startOtherServices(); //一些其它的服务
432            SystemServerInitThreadPool.shutdown();
433        } catch (Throwable ex) {
434            Slog.e("System", "******************************************");
435            Slog.e("System", "************ Failure starting system services", ex);
436            throw ex;
437        } finally {
438            traceEnd();
439        }
440
441        StrictMode.initVmDefaults(null);
442
443        if (!mRuntimeRestart && !isFirstBootOrUpgrade()) {
444            int uptimeMillis = (int) SystemClock.elapsedRealtime();
445            MetricsLogger.histogram(null, "boot_system_server_ready", uptimeMillis);
446            final int MAX_UPTIME_MILLIS = 60 * 1000;
447            if (uptimeMillis > MAX_UPTIME_MILLIS) {
448                Slog.wtf(SYSTEM_SERVER_TIMING_TAG,
449                        "SystemServer init took too long. uptimeMillis=" + uptimeMillis);
450            }
451        }
452        //保证SystemServer进程一直存活
453        // Loop forever.
454        Looper.loop();
455        throw new RuntimeException("Main thread loop unexpectedly exited");
456    }
```
###### 3.3 createSystemContext

```java
private void createSystemContext() {
           //创建ActivityThread, 在systemMain函数里面调用ActivityThread的构造函数
           //ActivityThread thread = new ActivityThread();
522        ActivityThread activityThread = ActivityThread.systemMain();
523        mSystemContext = activityThread.getSystemContext();
524        mSystemContext.setTheme(DEFAULT_SYSTEM_THEME);
525
526        final Context systemUiContext = activityThread.getSystemUiContext();
527        systemUiContext.setTheme(DEFAULT_SYSTEM_THEME);
528    }
529
```
当然在SystemServer里面启动了好多的java层的service，他们内部通过binder 来进行通信，这个地方就暂时不说了  
以后会专门开一篇来说SystemServer里面启动的服务，以及怎么通过binder来进行通信的。到这个地方我们的SystemServer  
算是启动完成了，那下面是怎么启动到launcher呢，继续看下一篇文章。
