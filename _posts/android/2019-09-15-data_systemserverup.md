---
layout: post
title:  Android 开机启动流程SystemServer上篇(三)
category: Android
tags: Systemserver
---
* content
{:toc}

###### 前言
前面的篇章已经打通了kernel到init只zygote进程，下面就说下怎么从zygote中把SystemServer启动
起来的.

```java
/frameworks/base/core/java/com/android/internal/os/
  - ZygoteInit.java
  - RuntimeInit.java
  - Zygote.java

/frameworks/base/core/services/java/com/android/server/
  - SystemServer.java

/frameworks/base/core/jni/
  - com_android_internal_os_Zygote.cpp
  - AndroidRuntime.cpp

```
###### 3.1 forkSystemServer
从上面的代码我们可以知道在zygote中调用startSystemServer方法，最终会调用到Zygote.forkSystemServer
的方法

```java
public static int forkSystemServer(int uid, int gid, int[] gids, int debugFlags, int[][] rlimits, long permittedCapabilities, long effectiveCapabilities) {
    VM_HOOKS.preFork();
    // 调用native方法fork system_server进程【见小节3】
    int pid = nativeForkSystemServer(
            uid, gid, gids, debugFlags, rlimits, permittedCapabilities, effectiveCapabilities);
    if (pid == 0) {
        Trace.setTracingEnabled(true);
    }
    VM_HOOKS.postForkCommon();
    return pid;
}

nativeForkSystemServer()方法在AndroidRuntime.cpp中注册的,这个我们在上一篇说过会有一个native的对应关系
com_android_internal_os_Zygote.cpp中的  
register_com_android_internal_os_Zygote()所以接下来进入如下方法。
```

###### 3.2 nativeForkSystemServer

在这个cpp文件里面
com_android_internal_os_Zygote.cpp

```java
static jint com_android_internal_os_Zygote_nativeForkSystemServer(
871        JNIEnv* env, jclass, uid_t uid, gid_t gid, jintArray gids,
872        jint runtime_flags, jobjectArray rlimits, jlong permittedCapabilities,
873        jlong effectiveCapabilities) {
     //fork子进程
874  pid_t pid = ForkAndSpecializeCommon(env, uid, gid, gids,
875                                      runtime_flags, rlimits,
876                                      permittedCapabilities, effectiveCapabilities,
877                                      MOUNT_EXTERNAL_DEFAULT, NULL, NULL, true, NULL,
878                                      NULL, false, NULL, NULL);
879  if (pid > 0) {
880      // The zygote process checks whether the child process has died or not.
         zygote进程，检测system_server进程是否创建
881      ALOGI("System server process %d has been created", pid);
882      gSystemServerPid = pid;
883      // There is a slight window that the system server process has crashed
884      // but it went unnoticed because we haven't published its pid yet. So
885      // we recheck here just to make sure that all is well.
886      int status;
887      if (waitpid(pid, &status, WNOHANG) == pid) {
888          ALOGE("System server process %d has died. Restarting Zygote!", pid);
             当system_server进程死亡后，重启zygote进程
889          RuntimeAbort(env, __LINE__, "System server process has died. Restarting Zygote!");
890      }
891
892      // Assign system_server to the correct memory cgroup.
893      // Not all devices mount /dev/memcg so check for the file first
894      // to avoid unnecessarily printing errors and denials in the logs.
895      if (!access("/dev/memcg/system/tasks", F_OK) &&
896                !WriteStringToFile(StringPrintf("%d", pid), "/dev/memcg/system/tasks")) {
897        ALOGE("couldn't write %d to /dev/memcg/system/tasks", pid);
898      }
899  }
900  return pid;
901}
```

###### 3.3 ForkAndSpecializeCommon

这个方法太长了，我只截取其中的一小部分的关键字
```java
static pid_t ForkAndSpecializeCommon(JNIEnv* env, uid_t uid, gid_t gid, jintArray javaGids,
540                                     jint runtime_flags, jobjectArray javaRlimits,
541                                     jlong permittedCapabilities, jlong effectiveCapabilities,
542                                     jint mount_external,
543                                     jstring java_se_info, jstring java_se_name,
544                                     bool is_system_server, jintArray fdsToClose,
545                                     jintArray fdsToIgnore, bool is_child_zygote,
546                                     jstring instructionSet, jstring dataDir) {
547
570
571  // Temporarily block SIGCHLD during forks. The SIGCHLD handler might
572  // log, which would result in the logging FDs we close being reopened.
573  // This would cause failures because the FDs are not whitelisted.
574  //
575  // Note that the zygote process is single threaded at this point.
576  if (sigprocmask(SIG_BLOCK, &sigchld, nullptr) == -1) {
577    fail_fn(CREATE_ERROR("sigprocmask(SIG_SETMASK, { SIGCHLD }) failed: %s", strerror(errno)));
578  }
579
580  // Close any logging related FDs before we start evaluating the list of
581  // file descriptors.
582  __android_log_close();
583
584  std::string error_msg;
585
586  // If this is the first fork for this zygote, create the open FD table.
587  // If it isn't, we just need to check whether the list of open files has
588  // changed (and it shouldn't in the normal case).
589  std::vector<int> fds_to_ignore;
590  if (!FillFileDescriptorVector(env, fdsToIgnore, &fds_to_ignore, &error_msg)) {
591    fail_fn(error_msg);
592  }
593  if (gOpenFdTable == NULL) {
594    gOpenFdTable = FileDescriptorTable::Create(fds_to_ignore, &error_msg);
595    if (gOpenFdTable == NULL) {
596      fail_fn(error_msg);
597    }
598  } else if (!gOpenFdTable->Restat(fds_to_ignore, &error_msg)) {
599    fail_fn(error_msg);
600  }
601  //在这里调用linux的标准创建进程的方法
602  pid_t pid = fork();
603
604  if (pid == 0) {
605    PreApplicationInit();
606
607    // Clean up any descriptors which must be closed immediately
608    if (!DetachDescriptors(env, fdsToClose, &error_msg)) {
609      fail_fn(error_msg);
610    }
611
612    // Re-open all remaining open file descriptors so that they aren't shared
613    // with the zygote across a fork.
614    if (!gOpenFdTable->ReopenOrDetach(&error_msg)) {
615      fail_fn(error_msg);
616    }
       .............
       最后return 这个pid，这里创建已经完成了

       fork()创建新进程，采用copy on write方式，这是linux创建进程的标准方法，会有两次return,
       对于pid==0为子进程的返回，对于pid>0为父进程的返回.到此system_server进程已完成了创建的所有工  
       作，接下来开始了system_server进程的真正工作.在zygoteinit.java中，创建完成systemserver进程之后
       执行handleSystemServerProcess
```

###### 3.4 handleSystemServerProcess
```java
private static Runnable handleSystemServerProcess(ZygoteConnection.Arguments parsedArgs) {
454        // set umask to 0077 so new files and directories will default to owner-only permissions.
455        Os.umask(S_IRWXG | S_IRWXO);
456
457        if (parsedArgs.niceName != null) {
458            Process.setArgV0(parsedArgs.niceName);
459        }
460        获取SYSTEMSERVERCLASSPATH环境变量中的值
           adb shell, export,查看所有环境变量的值, $SYSTEMSERVERCLASSPATH 取值
461        final String systemServerClasspath = Os.getenv("SYSTEMSERVERCLASSPATH");
462        if (systemServerClasspath != null) {
463            performSystemServerDexOpt(systemServerClasspath);
464            // Capturing profiles is only supported for debug or eng builds since selinux normally
465            // prevents it.
466            boolean profileSystemServer = SystemProperties.getBoolean(
467                    "dalvik.vm.profilesystemserver", false);
468            if (profileSystemServer && (Build.IS_USERDEBUG || Build.IS_ENG)) {
469                try {
470                    prepareSystemServerProfile(systemServerClasspath);
471                } catch (Exception e) {
472                    Log.wtf(TAG, "Failed to set up system server profile", e);
473                }
474            }
475        }
476
477        if (parsedArgs.invokeWith != null) {
478            String[] args = parsedArgs.remainingArgs;
479            // If we have a non-null system server class path, we'll have to duplicate the
480            // existing arguments and append the classpath to it. ART will handle the classpath
481            // correctly when we exec a new process.
482            if (systemServerClasspath != null) {
483                String[] amendedArgs = new String[args.length + 2];
484                amendedArgs[0] = "-cp";
485                amendedArgs[1] = systemServerClasspath;
486                System.arraycopy(args, 0, amendedArgs, 2, args.length);
487                args = amendedArgs;
488            }
489
490            WrapperInit.execApplication(parsedArgs.invokeWith,
491                    parsedArgs.niceName, parsedArgs.targetSdkVersion,
492                    VMRuntime.getCurrentInstructionSet(), null, args);
493
494            throw new IllegalStateException("Unexpected return from WrapperInit.execApplication");
495        } else {
496            ClassLoader cl = null;
497            if (systemServerClasspath != null) {
                   //加载SYSTEMSERVERCLASSPATH环境变量中的类
498                cl = createPathClassLoader(systemServerClasspath, parsedArgs.targetSdkVersion);
499
500                Thread.currentThread().setContextClassLoader(cl);
501            }
502
503            /*
504             * Pass the remaining arguments to SystemServer.
505             */
506            return ZygoteInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs, cl);
507        }
508
509        /* should never reach here */
510    }
```
###### 3.5 zygoteInit

```java
public static final Runnable zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader) {
903        if (RuntimeInit.DEBUG) {
904            Slog.d(RuntimeInit.TAG, "RuntimeInit: Starting application from zygote");
905        }
906
907        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ZygoteInit");
908        RuntimeInit.redirectLogStreams();
909
910        RuntimeInit.commonInit();
911        ZygoteInit.nativeZygoteInit();
912        return RuntimeInit.applicationInit(targetSdkVersion, argv, classLoader);
913    }
```
在AndroidRuntime.cpp中可以得知
nativeZygoteInit 所对应的c++方法为 com_android_internal_os_ZygoteInit_nativeZygoteInit
```java
int register_com_android_internal_os_ZygoteInit_nativeZygoteInit(JNIEnv* env)
258{
259    const JNINativeMethod methods[] = {
260        { "nativeZygoteInit", "()V",
261            (void*) com_android_internal_os_ZygoteInit_nativeZygoteInit },
262    };
263    return jniRegisterNativeMethods(env, "com/android/internal/os/ZygoteInit",
264        methods, NELEM(methods));
265}
```

gCurRuntime为zygote中的runtime
```java
static void com_android_internal_os_ZygoteInit_nativeZygoteInit(JNIEnv* env, jobject clazz)
231{
       //开启一个binder线程
232    gCurRuntime->onZygoteInit();
233}
234
```
###### 3.6 applicationInit
在zygoteinit中继续开启
```java
return RuntimeInit.applicationInit(targetSdkVersion, argv, classLoader);
```

```java
protected static Runnable applicationInit(int targetSdkVersion, String[] argv,
346            ClassLoader classLoader) {
347        // If the application calls System.exit(), terminate the process
348        // immediately without running any shutdown hooks.  It is not possible to
349        // shutdown an Android application gracefully.  Among other things, the
350        // Android runtime shutdown hooks close the Binder driver, which can cause
351        // leftover running threads to crash before the process actually exits.
352        nativeSetExitWithoutCleanup(true);
353
354        // We want to be fairly aggressive about heap utilization, to avoid
355        // holding on to a lot of memory that isn't needed.
356        VMRuntime.getRuntime().setTargetHeapUtilization(0.75f);
357        VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);
358
359        final Arguments args = new Arguments(argv);
360
361        // The end of of the RuntimeInit event (see #zygoteInit).
362        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
363
364        // Remaining arguments are passed to the start class's static main
365        return findStaticMain(args.startClass, args.startArgs, classLoader);
366    }

protected static Runnable findStaticMain(String className, String[] argv,
288            ClassLoader classLoader) {
289        Class<?> cl;
290
291        try {
292            cl = Class.forName(className, true, classLoader);
293        } catch (ClassNotFoundException ex) {
294            throw new RuntimeException(
295                    "Missing class when invoking static main " + className,
296                    ex);
297        }
298
299        Method m;
300        try {
              获取com.android.server.SystemServer的main方法
301            m = cl.getMethod("main", new Class[] { String[].class });
302        } catch (NoSuchMethodException ex) {
303            throw new RuntimeException(
304                    "Missing static main on " + className, ex);
305        } catch (SecurityException ex) {
306            throw new RuntimeException(
307                    "Problem getting static main on " + className, ex);
308        }
309
310        int modifiers = m.getModifiers();
311        if (! (Modifier.isStatic(modifiers) && Modifier.isPublic(modifiers))) {
312            throw new RuntimeException(
313                    "Main method is not public and static on " + className);
314        }
315
316        /*
317         * This throw gets caught in ZygoteInit.main(), which responds
318         * by invoking the exception's run() method. This arrangement
319         * clears up all the stack frames that were required in setting
320         * up the process.
321         */
322        return new MethodAndArgsCaller(m, argv);
323    }

紧接着
static class MethodAndArgsCaller implements Runnable {
480        /** method to call */
481        private final Method mMethod;
482
483        /** argument array */
484        private final String[] mArgs;
485
486        public MethodAndArgsCaller(Method method, String[] args) {
487            mMethod = method;
488            mArgs = args;
489        }
490
491        public void run() {
492            try {
493                mMethod.invoke(null, new Object[] { mArgs });
494            } catch (IllegalAccessException ex) {
495                throw new RuntimeException(ex);
496            } catch (InvocationTargetException ex) {
497                Throwable cause = ex.getCause();
498                if (cause instanceof RuntimeException) {
499                    throw (RuntimeException) cause;
500                } else if (cause instanceof Error) {
501                    throw (Error) cause;
502                }
503                throw new RuntimeException(ex);
504            }
505        }
506    }

最终会返回一个runnable. 会在zygote main函数里面调用. 最终com.android.server.SystemServer中的
main方法执行，并且作为一个fork出来的进程执行
```

到此终于进入到systemserver中的main方法里面，并且系统中最为重要的一个线程起来了.
