---
layout: post
title:  Android中Activity启动(上)
category: Android
tags: Activity的开启
---
* content
{:toc}

#### 前言
从一个App的角度来看，应用启动很简单，只用调用一下Activity中的startActivity方法就行了.但是背后  
发生了什么？我想不管是一个App开发者还是FW开发者都应该掌握的.

我把这部分大致分为两个部分.  
**1. 开启Activity**
**2. 创建新的进程**

#### 1.1 涉及到的类

ActivityThread : 主要的App入口，一个应用的主线程  
ApplicationThread ： 代理类，继承了aidl接口, 在ActivityThread成员变量中创建  
Instrumentation ： 负责发起Activity的启动、并具体负责Activity的创建以及Activity生命周期的回调  
一个应用里面只有一个Instrumentation对象,并且这个对象是在ActivityThread中创建的  
ActivityManagerService ：管理所有Activity, 并且Acivity通过binder跨进程发送信息给  
AMS,然后AMS通过Socket来创建进程.
ActivityStack ： Activity在AMS的栈管理，用来记录已经启动的Activity的先后关系，状态信息等通  
过ActivityStack决定是否需要启动新的进程。
ActivityRecord：ActivityStack的管理对象，每个Activity在AMS对应一个ActivityRecord，  
来记录Activity的状态以及其他的管理信息。其实就是服务器端的Activity对象的映像


#### 1.2 startActivity
在Activity中调用startActivity最终会调用到startActivityForResult
```java
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
        //没有设置父的Activity
        if (mParent == null) {
            options = transferSpringboardActivityOptions(options);
            Instrumentation.ActivityResult ar =
                //开启execStartActivity，mMainThread创建进程的时候的
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            if (ar != null) {
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
            if (requestCode >= 0) {
                // If this start is requesting a result, we can avoid making
                // the activity visible until the result is received.  Setting
                // this code during onCreate(Bundle savedInstanceState) or onResume() will keep the
                // activity hidden during this time, to avoid flickering.
                // This can only be done when a result is requested because
                // that guarantees we will get information back when the
                // activity is finished, no matter what happens to it.
                mStartedActivity = true;
            }

            cancelInputsAndStartExitTransition(options);
            // TODO Consider clearing/flushing other event sources and events for child windows.
        } else {
            if (options != null) {
                mParent.startActivityFromChild(this, intent, requestCode, options);
            } else {
                // Note we want to go through this method for compatibility with
                // existing applications that may have overridden it.
                mParent.startActivityFromChild(this, intent, requestCode);
            }
        }
    }

```

接着调用 execStartActivity

```java
public ActivityResult execStartActivity(
1636            Context who, IBinder contextThread, IBinder token, Activity target,
1637            Intent intent, int requestCode, Bundle options) {
1638        .......................
1665        try {
1666            intent.migrateExtraStreamToClipData();
1667            intent.prepareToLeaveProcess(who);
                //通过binder获取AMS，实际上是调用的AMS中的startActivity
1668            int result = ActivityManager.getService()
1669                .startActivity(whoThread, who.getBasePackageName(), intent,
1670                        intent.resolveTypeIfNeeded(who.getContentResolver()),
1671                        token, target != null ? target.mEmbeddedID : null,
1672                        requestCode, 0, null, options);
1673            checkStartActivityResult(result, intent);
1674        } catch (RemoteException e) {
1675            throw new RuntimeException("Failure from system", e);
1676        }
1677        return null;
            ........................
1678    }
1679
```

可以看下ActivityManager.getService()实现
```java
/**
4123     * @hide
4124     */
4125    public static IActivityManager getService() {
4126        return IActivityManagerSingleton.get();
4127    }
4128
4129    private static final Singleton<IActivityManager> IActivityManagerSingleton =
4130            new Singleton<IActivityManager>() {
4131                @Override
4132                protected IActivityManager create() {
4133                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                        //就是AMS的binder接口
4134                    final IActivityManager am = IActivityManager.Stub.asInterface(b);
4135                    return am;
4136                }
4137            };
```

在AMS里面调用startActivity方法:
```java
@Override
5079    public final int startActivity(IApplicationThread caller, String callingPackage,
5080            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
5081            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
5082        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
5083                resultWho, requestCode, startFlags, profilerInfo, bOptions,
5084                UserHandle.getCallingUserId());
5085    }
```

这个地方其实和8.0的代码差别还是挺大的.mActivityStartController这个地方做了一个封装，运用建造者模式.  
看来设计模式还要好好学学呀。源码中处处有设计模式的思想
```java
public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
5097            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
5098            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId,
5099            boolean validateIncomingUser) {
5100        enforceNotIsolatedCaller("startActivity");
5101
5102        userId = mActivityStartController.checkTargetUser(userId, validateIncomingUser,
5103                Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");
5104        
5105        // TODO: Switch to user app stacks here.
5106        return mActivityStartController.obtainStarter(intent, "startActivityAsUser")
5107                .setCaller(caller)
5108                .setCallingPackage(callingPackage)
5109                .setResolvedType(resolvedType)
5110                .setResultTo(resultTo)
5111                .setResultWho(resultWho)
5112                .setRequestCode(requestCode)
5113                .setStartFlags(startFlags)
5114                .setProfilerInfo(profilerInfo)
5115                .setActivityOptions(bOptions)
5116                .setMayWait(userId)
5117                .execute();
5118
5119    }
5120
```
接着查看mActivityStartController类里面的obtainStarter方法  

我就把这些全部都拿过来了.从代码里面看，构造函数里面传进了DefaultFactory.那接着查看DefaultFactory  
在哪里创建。

```java
ActivityStartController(ActivityManagerService service) {
115        this(service, service.mStackSupervisor,
116                new DefaultFactory(service, service.mStackSupervisor,
117                    new ActivityStartInterceptor(service, service.mStackSupervisor)));
118    }
119
120    @VisibleForTesting
121    ActivityStartController(ActivityManagerService service, ActivityStackSupervisor supervisor,
122            Factory factory) {
123        mService = service;
124        mSupervisor = supervisor;
125        mHandler = new StartHandler(mService.mHandlerThread.getLooper());
126        mFactory = factory;
127        mFactory.setController(this);
128        mPendingRemoteAnimationRegistry = new PendingRemoteAnimationRegistry(service,
129                service.mHandler);
130    }
131
132    /**
133     * @return A starter to configure and execute starting an activity. It is valid until after
134     *         {@link ActivityStarter#execute} is invoked. At that point, the starter should be
135     *         considered invalid and no longer modified or used.
136     */
137    ActivityStarter obtainStarter(Intent intent, String reason) {
138        return mFactory.obtain().setIntent(intent).setReason(reason);
139    }
140
```
在ActivityStarter里面定义了DefaultFactory，其实就是调用ActivityStarter的execute()方法
```java
static class DefaultFactory implements Factory {
227        /**
228         * The maximum count of starters that should be active at one time:
229         * 1. last ran starter (for logging and post activity processing)
230         * 2. current running starter
231         * 3. starter from re-entry in (2)
232         */
233        private final int MAX_STARTER_COUNT = 3;
234
235        private ActivityStartController mController;
236        private ActivityManagerService mService;
237        private ActivityStackSupervisor mSupervisor;
238        private ActivityStartInterceptor mInterceptor;
239
240        private SynchronizedPool<ActivityStarter> mStarterPool =
241                new SynchronizedPool<>(MAX_STARTER_COUNT);
242
243        DefaultFactory(ActivityManagerService service,
244                ActivityStackSupervisor supervisor, ActivityStartInterceptor interceptor) {
245            mService = service;
246            mSupervisor = supervisor;
247            mInterceptor = interceptor;
248        }
249
250        @Override
251        public void setController(ActivityStartController controller) {
252            mController = controller;
253        }
254
255        @Override
256        public ActivityStarter obtain() {
257            ActivityStarter starter = mStarterPool.acquire();
258
259            if (starter == null) {
260                starter = new ActivityStarter(mController, mService, mSupervisor, mInterceptor);
261            }
262
263            return starter;
264        }
265
266        @Override
267        public void recycle(ActivityStarter starter) {
268            starter.reset(true /* clearRequest*/);
269            mStarterPool.release(starter);
270        }
271    }
```
最终调用到的是ActivityStarter的execute方法. 这里我就省略一下了  
```java
private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
1194                IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
1195                int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
1196                ActivityRecord[] outActivity) {
1197            
1199            mService.mWindowManager.deferSurfaceLayout();
1200            result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
1201                    startFlags, doResume, options, inTask, outActivity);
1202        
1213
1214        postStartActivityProcessing(r, result, mTargetStack);
1215
1216        return result;
1217    }
```

接着调用ActivityStarter的startActivityUnchecked方法
```java
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
1221            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
1222            int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
1223            ActivityRecord[] outActivity) {
1224
1225        setInitialState(r, options, inTask, doResume, startFlags, sourceRecord, voiceSession,
1226                voiceInteractor);
1227
1228        computeLaunchingTaskFlags();
1229
1230        computeSourceStack();

            mTargetStack.startActivityLocked(mStartActivity, topFocused, newTask, mKeepCurTransition,

            mSupervisor.resumeFocusedStackTopActivityLocked();
1231}
```

```java
boolean resumeFocusedStackTopActivityLocked(
2215            ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {
2216
2217        if (!readyToResume()) {
2218            return false;
2219        }
2220
2221        if (targetStack != null && isFocusedStack(targetStack)) {
2222            return targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
2223        }
2224
2225        final ActivityRecord r = mFocusedStack.topRunningActivityLocked();
2226        if (r == null || !r.isState(RESUMED)) {
                //调用ActivityStack.java的resumeTopActivityUncheckedLocked
2227            mFocusedStack.resumeTopActivityUncheckedLocked(null, null);
2228        } else if (r.isState(RESUMED)) {
2229            // Kick off any lingering app transitions form the MoveTaskToFront operation.
2230            mFocusedStack.executeAppTransition(targetOptions);
2231        }
2232
2233        return false;
2234    }
```

```java
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
2283        if (mStackSupervisor.inResumeTopActivity) {
2284            // Don't even start recursing.
2285            return false;
2286        }
2287
2288        boolean result = false;
2289        try {
2290            // Protect against recursion.
2291            mStackSupervisor.inResumeTopActivity = true;
                //自身的resumeTopActivityInnerLocked方法啊
2292            result = resumeTopActivityInnerLocked(prev, options);
2293
2294            // When resuming the top activity, it may be necessary to pause the top activity (for
2295            // example, returning to the lock screen. We suppress the normal pause logic in
2296            // {@link #resumeTopActivityUncheckedLocked}, since the top activity is resumed at the
2297            // end. We call the {@link ActivityStackSupervisor#checkReadyForSleepLocked} again here
2298            // to ensure any necessary pause logic occurs. In the case where the Activity will be
2299            // shown regardless of the lock screen, the call to
2300            // {@link ActivityStackSupervisor#checkReadyForSleepLocked} is skipped.
2301            final ActivityRecord next = topRunningActivityLocked(true /* focusableOnly */);
2302            if (next == null || !next.canTurnScreenOn()) {
2303                checkReadyForSleep();
2304            }
2305        } finally {
2306            mStackSupervisor.inResumeTopActivity = false;
2307        }
2308
2309        return result;
2310    }
```
```java
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
  if (mResumedActivity != null) {
2443            if (DEBUG_STATES) Slog.d(TAG_STATES,
2444                    "resumeTopActivityLocked: Pausing " + mResumedActivity);
                //当前的Activiyt肯定执行了resume,所以先调用了自己的onPause方法，通过IPC通信
2445            pausing |= startPausingLocked(userLeaving, false, next, false);
2446        }
      mStackSupervisor.startSpecificActivityLocked(next, true, true);
}
```

这个地方要注意, 如果这个进程没有启动过，
```java
# com.android.server.am.ActivityStackSupervisor
void startSpecificActivityLocked(ActivityRecord r,
        boolean andResume, boolean checkConfig) {
    ...
    //这个地方取出来的为null,所以肯定会先创建进程
    ProcessRecord app = mService.getProcessRecordLocked(r.processName,
    1682                r.info.applicationInfo.uid, true);
    1683


    if (app != null && app.thread != null) {
        try {
    ...
            realStartActivityLocked(r, app, andResume, checkConfig);
            return;
        } catch (RemoteException e) {
           ...
        }

    }

    //去创建进程，这个在下面一章讲解
    mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
            "activity", r.intent.getComponent(), false, false, true);
}
```


上面的startProcessLocked会通过ActivityThread这个类创建一个进程，这个会在下面一个章节中讲解  
这个地方先记着，会调用到ActivityThread的main函数.创建一个新进程

```java
public static void main(String[] args) {
6624        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
6625
6626        // CloseGuard defaults to true and can be quite spammy.  We
6627        // disable it here, but selectively enable it later (via
6628        // StrictMode) on debug builds, but using DropBox, not logs.
6629        CloseGuard.setEnabled(false);
6630
6631        Environment.initForCurrentUser();
6632
6633        // Set the reporter for event logging in libcore
6634        EventLogger.setReporter(new EventLoggingReporter());
6635
6636        // Make sure TrustedCertificateStore looks in the right place for CA certificates
6637        final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
6638        TrustedCertificateStore.setDefaultUserDirectory(configDir);
6639
6640        Process.setArgV0("<pre-initialized>");
6641
6642        Looper.prepareMainLooper();
6643
6644        // Find the value for {@link #PROC_START_SEQ_IDENT} if provided on the command line.
6645        // It will be in the format "seq=114"
6646        long startSeq = 0;
6647        if (args != null) {
6648            for (int i = args.length - 1; i >= 0; --i) {
6649                if (args[i] != null && args[i].startsWith(PROC_START_SEQ_IDENT)) {
6650                    startSeq = Long.parseLong(
6651                            args[i].substring(PROC_START_SEQ_IDENT.length()));
6652                }
6653            }
6654        }
            //创建ActivityThread对象，
6655        ActivityThread thread = new ActivityThread();
6656        thread.attach(false, startSeq);
6657
6658        if (sMainThreadHandler == null) {
6659            sMainThreadHandler = thread.getHandler();
6660        }
6661
6662        if (false) {
6663            Looper.myLooper().setMessageLogging(new
6664                    LogPrinter(Log.DEBUG, "ActivityThread"));
6665        }
6666
6667        // End of event ActivityThreadMain.
6668        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            //开启looper循环，确保进程不退出
6669        Looper.loop();
6670
6671        throw new RuntimeException("Main thread loop unexpectedly exited");
6672    }
6673
```
函数attach最终调用了ActivityManagerService的远程接口ActivityManagerProxy的attachApplication函数,  
传入的参数是mAppThread，这是一个ApplicationThread类型的Binder对象，它的作用是用来进行进程间通信的。  

```java

private void attach(boolean system, long startSeq) {
6479        sCurrentActivityThread = this;
6480        mSystemThread = system;
6481        if (!system) {
6482            ViewRootImpl.addFirstDrawHandler(new Runnable() {
6483                @Override
6484                public void run() {
6485                    ensureJitEnabled();
6486                }
6487            });
6488            android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",
6489                                                    UserHandle.myUserId());
6490            RuntimeInit.setApplicationObject(mAppThread.asBinder());
                //获取到AMS对象
6491            final IActivityManager mgr = ActivityManager.getService();
6492            try {
                    //这个地方回调到AMS里面的attachApplication方法
6493                mgr.attachApplication(mAppThread, startSeq);
6494            } catch (RemoteException ex) {
6495                throw ex.rethrowFromSystemServer();
6496            }
6497            // Watch for getting close to heap limit.
6498            BinderInternal.addGcWatcher(new Runnable() {
6499                @Override public void run() {
6500                    if (!mSomeActivitiesChanged) {
6501                        return;
6502                    }
6503                    Runtime runtime = Runtime.getRuntime();
6504                    long dalvikMax = runtime.maxMemory();
6505                    long dalvikUsed = runtime.totalMemory() - runtime.freeMemory();
6506                    if (dalvikUsed > ((3*dalvikMax)/4)) {
6507                        if (DEBUG_MEMORY_TRIM) Slog.d(TAG, "Dalvik max=" + (dalvikMax/1024)
6508                                + " total=" + (runtime.totalMemory()/1024)
6509                                + " used=" + (dalvikUsed/1024));
6510                        mSomeActivitiesChanged = false;
6511                        try {
6512                            mgr.releaseSomeActivities(mAppThread);
6513                        } catch (RemoteException e) {
6514                            throw e.rethrowFromSystemServer();
6515                        }
6516                    }
6517                }
6518            });

```
上面获取到AMS并且调用到AMS里面的attachApplication方法, 接着调用AMS里面的attachApplicationLocked方法,
```java

private final boolean attachApplicationLocked(IApplicationThread thread,
7573            int pid, int callingUid, long startSeq) {
  // See if the top visible activity is waiting to run in this process...
  7867        if (normalMode) {
  7868            try {
                      //调用ActivityStackSupervisor.java里面的attachApplicationLocked方法
  7869                if (mStackSupervisor.attachApplicationLocked(app)) {
  7870                    didSomething = true;
  7871                }
  7872            } catch (Exception e) {
  7873                Slog.wtf(TAG, "Exception thrown launching activities in " + app, e);
  7874                badApp = true;
  7875            }
  7876        }

}
```

ActivityStackSupervisor类里面的attachApplicationLocked方法


```java
boolean attachApplicationLocked(ProcessRecord app) throws RemoteException {
972        final String processName = app.processName;
973        boolean didSomething = false;
974        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
975            final ActivityDisplay display = mActivityDisplays.valueAt(displayNdx);
976            for (int stackNdx = display.getChildCount() - 1; stackNdx >= 0; --stackNdx) {
977                final ActivityStack stack = display.getChildAt(stackNdx);
978                if (!isFocusedStack(stack)) {
979                    continue;
980                }
981                stack.getAllRunningVisibleActivitiesLocked(mTmpActivityList);
982                final ActivityRecord top = stack.topRunningActivityLocked();
983                final int size = mTmpActivityList.size();
984                for (int i = 0; i < size; i++) {
985                    final ActivityRecord activity = mTmpActivityList.get(i);
986                    if (activity.app == null && app.uid == activity.info.applicationInfo.uid
987                            && processName.equals(activity.processName)) {
988                        try {
                              // startActivity
989                            if (realStartActivityLocked(activity, app,
990                                    top == activity /* andResume */, true /* checkConfig */)) {
991                                didSomething = true;
992                            }
993                        } catch (RemoteException e) {
994                            Slog.w(TAG, "Exception in new application when starting activity "
995                                    + top.intent.getComponent().flattenToShortString(), e);
996                            throw e;
997                        }
998                    }
999                }
1000            }
1001        }
1002        if (!didSomething) {
1003            ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
1004        }
1005        return didSomething;
1006    }

```

通过realStartActivityLocked方法来调用scheduleTransaction方法

```java
final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
1378            boolean andResume, boolean checkConfig) throws RemoteException {

  // Create activity launch transaction.
1523                final ClientTransaction clientTransaction = ClientTransaction.obtain(app.thread,
1524                        r.appToken);
1525                clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
1526                        System.identityHashCode(r), r.info,
1527                        // TODO: Have this take the merged configuration instead of separate global
1528                        // and override configs.
1529                        mergedConfiguration.getGlobalConfiguration(),
1530                        mergedConfiguration.getOverrideConfiguration(), r.compat,
1531                        r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
1532                        r.persistentState, results, newIntents, mService.isNextTransitionForward(),
1533                        profilerInfo));
1534
1535                // Set desired final state.
1536                final ActivityLifecycleItem lifecycleItem;
1537                if (andResume) {
1538                    lifecycleItem = ResumeActivityItem.obtain(mService.isNextTransitionForward());
1539                } else {
1540                    lifecycleItem = PauseActivityItem.obtain();
1541                }
1542                clientTransaction.setLifecycleStateRequest(lifecycleItem);
1543
1544                // Schedule transaction.//在9.0上面这个地方重构了全部转化为Transaction一类的方法来
                    // 调用launch
1545                mService.getLifecycleManager().scheduleTransaction(clientTransaction);

}
```

调用ClientLifecycleManager 里面的scheduleTransaction方法

```java
void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
46        final IApplicationThread client = transaction.getClient();
47        transaction.schedule();
48        if (!(client instanceof Binder)) {
49            // If client is not an instance of Binder - it's a remote call and at this point it is
50            // safe to recycle the object. All objects used for local calls will be recycled after
51            // the transaction is executed on client in ActivityThread.
52            transaction.recycle();
53        }
54    }
```

其实是调用了ClientTransaction中的schedule方法

```java
private IApplicationThread mClient;
public void schedule() throws RemoteException {
129        mClient.scheduleTransaction(this);
130    }

```

IApplicationThread其实是一个aidl接口在ActivityThread中定义,  
继续查看IApplicationThread里面的scheduleTransaction方法,


```java
private class ApplicationThread extends IApplicationThread.Stub {
  ..........................
@Override
1539        public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
1540            ActivityThread.this.scheduleTransaction(transaction);
1541        }
}
  ..........................
```

接着调用ActivityThread里面的scheduleTransaction方法，如果你在ActivityThread类里面搜的话  
是找不到的这个方法的.那就去查看是ActivityThread继承了谁ClientTransactionHandler，进而查找  
这个里面的方法  

```java
/** Prepare and schedule transaction for execution. */
43    void scheduleTransaction(ClientTransaction transaction) {
44        transaction.preExecute(this);
          //调用ActivityThread类里面的方法sendMessage方法
45        sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
46    }
47
```

查看ActivityThread里面的sendMessage方法，不管怎样都会调用到mH的sendMessage方法

```java
void sendMessage(int what, Object obj) {
2757        sendMessage(what, obj, 0, 0, false);
2758    }
2759
2760    private void sendMessage(int what, Object obj, int arg1) {
2761        sendMessage(what, obj, arg1, 0, false);
2762    }
2763
2764    private void sendMessage(int what, Object obj, int arg1, int arg2) {
2765        sendMessage(what, obj, arg1, arg2, false);
2766    }
2767
2768    private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
2769        if (DEBUG_MESSAGES) Slog.v(
2770            TAG, "SCHEDULE " + what + " " + mH.codeToString(what)
2771            + ": " + arg1 + " / " + obj);
2772        Message msg = Message.obtain();
2773        msg.what = what;
2774        msg.obj = obj;
2775        msg.arg1 = arg1;
2776        msg.arg2 = arg2;
2777        if (async) {
2778            msg.setAsynchronous(true);
2779        }
2780        mH.sendMessage(msg);
2781    }
2782
2783    private void sendMessage(int what, Object obj, int arg1, int arg2, int seq) {
2784        if (DEBUG_MESSAGES) Slog.v(
2785                TAG, "SCHEDULE " + mH.codeToString(what) + " arg1=" + arg1 + " arg2=" + arg2 +
2786                        "seq= " + seq);
2787        Message msg = Message.obtain();
2788        msg.what = what;
2789        SomeArgs args = SomeArgs.obtain();
2790        args.arg1 = obj;
2791        args.argi1 = arg1;
2792        args.argi2 = arg2;
2793        args.argi3 = seq;
2794        msg.obj = args;
2795        mH.sendMessage(msg);
2796    }
```

接着查看mH这个是什么，ActivityThread的成员变量

```java
final H mH = new H();

class H extends Handler {
   public void handleMessage(Message msg) {
      .................
      case EXECUTE_TRANSACTION:
1807                    final ClientTransaction transaction = (ClientTransaction) msg.obj;
1808                    mTransactionExecutor.execute(transaction);
1809                    if (isSystem()) {
1810                        // Client transactions inside system process are recycled on the client side
1811                        // instead of ClientLifecycleManager to avoid being cleared before this
1812                        // message is handled.
1813                        transaction.recycle();
1814                    }
1815                    // TODO(lifecycler): Recycle locally scheduled transactions.
      .................
   }
}
```

接着查看mTransactionExecutor，这个成员变量是在ActivityThread里面被赋值的.TransactionExecutor.java  

```java
public void execute(ClientTransaction transaction) {
65        final IBinder token = transaction.getActivityToken();
66        log("Start resolving transaction for client: " + mTransactionHandler + ", token: " + token);
67        
68        executeCallbacks(transaction);
69
70        executeLifecycleState(transaction);
71        mPendingActions.clear();
72        log("End resolving transaction");
73    }
```

对于executeCallbacks和executeLifecycleState来说都是调用到了cycleToPath方法，查看这个方法  

```java
private void cycleToPath(ActivityClientRecord r, int finish,
161            boolean excludeLastState) {
162        final int start = r.getLifecycleState();
163        log("Cycle from: " + start + " to: " + finish + " excludeLastState:" + excludeLastState);
164        final IntArray path = mHelper.getLifecyclePath(start, finish, excludeLastState);
165        performLifecycleSequence(r, path);
166    }
167
```

继续调用到ActivityThread类里面的方法, mTransactionHandler是ActivityThread的父类

```java
/** Transition the client through previously initialized state sequence. */
169    private void performLifecycleSequence(ActivityClientRecord r, IntArray path) {
170        final int size = path.size();
171        for (int i = 0, state; i < size; i++) {
172            state = path.get(i);
173            log("Transitioning to state: " + state);
174            switch (state) {
175                case ON_CREATE:
176                    mTransactionHandler.handleLaunchActivity(r, mPendingActions,
177                            null /* customIntent */);
178                    break;
179                case ON_START:
180                    mTransactionHandler.handleStartActivity(r, mPendingActions);
181                    break;
182                case ON_RESUME:
183                    mTransactionHandler.handleResumeActivity(r.token, false /* finalStateRequest */,
184                            r.isForward, "LIFECYCLER_RESUME_ACTIVITY");
185                    break;
186                case ON_PAUSE:
187                    mTransactionHandler.handlePauseActivity(r.token, false /* finished */,
188                            false /* userLeaving */, 0 /* configChanges */, mPendingActions,
189                            "LIFECYCLER_PAUSE_ACTIVITY");
190                    break;
191                case ON_STOP:
192                    mTransactionHandler.handleStopActivity(r.token, false /* show */,
193                            0 /* configChanges */, mPendingActions, false /* finalStateRequest */,
194                            "LIFECYCLER_STOP_ACTIVITY");
195                    break;
196                case ON_DESTROY:
197                    mTransactionHandler.handleDestroyActivity(r.token, false /* finishing */,
198                            0 /* configChanges */, false /* getNonConfigInstance */,
199                            "performLifecycleSequence. cycling to:" + path.get(size - 1));
200                    break;
201                case ON_RESTART:
202                    mTransactionHandler.performRestartActivity(r.token, false /* start */);
203                    break;
204                default:
205                    throw new IllegalArgumentException("Unexpected lifecycle state: " + state);
206            }
207        }
208    }
```

在ActivityThread里面调用handleLaunchActivity, 接着调用performLaunchActivity这个方法


```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
   .......
   mInstrumentation.callActivityOnCreate(activity, r.state);
   .......
}
```

查看Instrumentation.java里面的callActivityOnCreate方法

```java
终于调用到了performCreate的方法，调用Activity里面的oncreate方法
public void callActivityOnCreate(Activity activity, Bundle icicle) {
1270        prePerformCreate(activity);
1271        activity.performCreate(icicle);
1272        postPerformCreate(activity);
1273    }
1274
```

接着调用handleStartActivity方法，handleResumeActivity等方法，添加view的操作在onresume里面操作
