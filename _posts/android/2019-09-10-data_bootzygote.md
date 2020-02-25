---
layout: post
title:  Android 开机启动流程至Launcher zygote篇(二)
category: Android
tags: 开机启动
---
* content
{:toc}

###### 2 zygote概述
Zygote是由init进程通过解析init.zygote.rc文件而创建的，zygote所对应的可执行程序app_process，
所对应的源文件是App_main.cpp，进程名为zygote。

###### 2.1 zygote 启动
通过查看<http://androidxref.com/9.0.0_r3/xref/system/core/rootdir/init.rc>文件  
在 **/system/core/rootdir/init.rc** 文件里面会导入  
**import /init.${ro.zygote}.rcimport /init.${ro.zygote}.rc**
这个目录下面<http://androidxref.com/9.0.0_r3/xref/system/core/rootdir/> zygote文件里面  

```java
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
    class main
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart media
    onrestart restart netd
```

大致流程

![avatar](https://github.com/skypx/BlogResource/raw/master/androidsystem/zygote_process.jpg)

这里不在画了，用了gityuan先生的图，很简单的，跟着代码就可以画了


而init.rc文件会在init进程里面解析::
```java
static void LoadBootScripts(ActionManager& action_manager, ServiceList& service_list) {
111    Parser parser = CreateParser(action_manager, service_list);
112
113    std::string bootscript = GetProperty("ro.boot.init_rc", "");
114    if (bootscript.empty()) {
115        parser.ParseConfig("/init.rc");
116        if (!parser.ParseConfig("/system/etc/init")) {
117            late_import_paths.emplace_back("/system/etc/init");
118        }
119        if (!parser.ParseConfig("/product/etc/init")) {
120            late_import_paths.emplace_back("/product/etc/init");
121        }
122        if (!parser.ParseConfig("/odm/etc/init")) {
123            late_import_paths.emplace_back("/odm/etc/init");
124        }
125        if (!parser.ParseConfig("/vendor/etc/init")) {
126            late_import_paths.emplace_back("/vendor/etc/init");
127        }
128    } else {
129        parser.ParseConfig(bootscript);
130    }
131}
132
```
所以zygote进程就启动了  
源码所在目录<http://androidxref.com/9.0.0_r3/xref/frameworks/base/cmds/app_process/>

在zygote文件里面可以看到
```java
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
```
后面的参数是 **--zygote** 和 **--start-system-server**
```java
while (i < argc) {
279        const char* arg = argv[i++];
280        if (strcmp(arg, "--zygote") == 0) {
	             下面要用到
281            zygote = true;
282            niceName = ZYGOTE_NICE_NAME;
283        } else if (strcmp(arg, "--start-system-server") == 0) {
	             需要开启systemserver
284            startSystemServer = true;
285        } else if (strcmp(arg, "--application") == 0) {
286            application = true;
287        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
288            niceName.setTo(arg + 12);
289        } else if (strncmp(arg, "--", 2) != 0) {
290            className.setTo(arg);
291            break;
292        } else {
293            --i;
294            break;
295        }
296    }
297
```

```java
348    zygote为true,通过反射调用zygoteinit,开启虚拟机，在开启虚拟机的时候加载android基础的类
       bootclasspath路径下的类
349    if (zygote) {
350        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
351    //如果是App的话
        } else if (className) {
352        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
353    } else {
354        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
355        app_usage();
356        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
357    }
358}
359
```
###### 2.1 AndroidRuntime启动
可以看到在app_main.cpp里面定义了runtime

```java
class AppRuntime : public AndroidRuntime
34{
35public:
36    AppRuntime(char* argBlockStart, const size_t argBlockLength)
37        : AndroidRuntime(argBlockStart, argBlockLength)
38        , mClass(NULL)
39    {
40    }
41
42    void setClassNameAndArgs(const String8& className, int argc, char * const *argv) {
43        mClassName = className;
44        for (int i = 0; i < argc; ++i) {
45             mArgs.add(String8(argv[i]));
46        }
47    }
48
49    virtual void onVmCreated(JNIEnv* env)
50    {
51        if (mClassName.isEmpty()) {
52            return; // Zygote. Nothing to do here.
53        }
54
55        /*
56         * This is a little awkward because the JNI FindClass call uses the
57         * class loader associated with the native method we're executing in.
58         * If called in onStarted (from RuntimeInit.finishInit because we're
59         * launching "am", for example), FindClass would see that we're calling
60         * from a boot class' native method, and so wouldn't look for the class
61         * we're trying to look up in CLASSPATH. Unfortunately it needs to,
62         * because the "am" classes are not boot classes.
63         *
64         * The easiest fix is to call FindClass here, early on before we start
65         * executing boot class Java code and thereby deny ourselves access to
66         * non-boot classes.
67         */
68        char* slashClassName = toSlashClassName(mClassName.string());
69        mClass = env->FindClass(slashClassName);
70        if (mClass == NULL) {
71            ALOGE("ERROR: could not find class '%s'\n", mClassName.string());
72        }
73        free(slashClassName);
74
75        mClass = reinterpret_cast<jclass>(env->NewGlobalRef(mClass));
76    }
77
78    virtual void onStarted()
79    {
80        sp<ProcessState> proc = ProcessState::self();
81        ALOGV("App process: starting thread pool.\n");
82        proc->startThreadPool();
83
84        AndroidRuntime* ar = AndroidRuntime::getRuntime();
85        ar->callMain(mClassName, mClass, mArgs);
86
87        IPCThreadState::self()->stopProcess();
88        hardware::IPCThreadState::self()->stopProcess();
89    }
90
91    virtual void onZygoteInit()
92    {
93        sp<ProcessState> proc = ProcessState::self();
94        ALOGV("App process: starting thread pool.\n");
95        proc->startThreadPool();
96    }
97
98    virtual void onExit(int code)
99    {
100        if (mClassName.isEmpty()) {
101            // if zygote
102            IPCThreadState::self()->stopProcess();
103            hardware::IPCThreadState::self()->stopProcess();
104        }
105
106        AndroidRuntime::onExit(code);
107    }
108
109
110    String8 mClassName;
111    Vector<String8> mArgs;
112    jclass mClass;
113};
114
115}
116
```

查看AndroidRunTime.cpp

```java
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
1057{
1058    ALOGD(">>>>>> START %s uid %d <<<<<<\n",
1059            className != NULL ? className : "(unknown)", getuid());
1060
1061    static const String8 startSystemServer("start-system-server");
1062
1063    /*
1064     * 'startSystemServer == true' means runtime is obsolete and not run from
1065     * init.rc anymore, so we print out the boot start event here.
1066     */
1067    for (size_t i = 0; i < options.size(); ++i) {
1068        if (options[i] == startSystemServer) {
1069           /* track our progress through the boot sequence */
1070           const int LOG_BOOT_PROGRESS_START = 3000;
1071           LOG_EVENT_LONG(LOG_BOOT_PROGRESS_START,  ns2ms(systemTime(SYSTEM_TIME_MONOTONIC)));
1072        }
1073    }
1074
1075    const char* rootDir = getenv("ANDROID_ROOT");
1076    if (rootDir == NULL) {
1077        rootDir = "/system";
1078        if (!hasDir("/system")) {
1079            LOG_FATAL("No root directory specified, and /android does not exist.");
1080            return;
1081        }
1082        setenv("ANDROID_ROOT", rootDir, 1);
1083    }
1084
1085    //const char* kernelHack = getenv("LD_ASSUME_KERNEL");
1086    //ALOGD("Found LD_ASSUME_KERNEL='%s'\n", kernelHack);
1087
1088    /* start the virtual machine */
1089    JniInvocation jni_invocation;
1090    jni_invocation.Init(NULL);
1091    JNIEnv* env;
        //创建虚拟机，看下面2.2
1092    if (startVm(&mJavaVM, &env, zygote) != 0) {
1093        return;
1094    }
1095    onVmCreated(env);
1096
1097    /*
1098     * Register android functions.
1099     */
        //JNI方法注册，看下面2.3
1100    if (startReg(env) < 0) {
1101        ALOGE("Unable to register all android natives\n");
1102        return;
1103    }
1104
1105    /*
1106     * We want to call main() with a String array with arguments in it.
1107     * At present we have two arguments, the class name and an option string.
1108     * Create an array to hold them.
1109     */
1110    jclass stringClass;
1111    jobjectArray strArray;
1112    jstring classNameStr;
1113
1114    stringClass = env->FindClass("java/lang/String");
1115    assert(stringClass != NULL);
1116    strArray = env->NewObjectArray(options.size() + 1, stringClass, NULL);
1117    assert(strArray != NULL);
1118    classNameStr = env->NewStringUTF(className);
1119    assert(classNameStr != NULL);
1120    env->SetObjectArrayElement(strArray, 0, classNameStr);
1121
1122    for (size_t i = 0; i < options.size(); ++i) {
1123        jstring optionsStr = env->NewStringUTF(options.itemAt(i).string());
1124        assert(optionsStr != NULL);
1125        env->SetObjectArrayElement(strArray, i + 1, optionsStr);
1126    }
1127
1128    /*
1129     * Start VM.  This thread becomes the main thread of the VM, and will
1130     * not return until the VM exits.
1131     */
        将"com.android.internal.os.ZygoteInit"转换为"com/android/internal/os/ZygoteInit"
        因为传入进来的是com.android.internal.os.ZygoteInit
1132    char* slashClassName = toSlashClassName(className != NULL ? className : "");
1133    jclass startClass = env->FindClass(slashClassName);
1134    if (startClass == NULL) {
1135        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
1136        /* keep going */
1137    } else {
            //进入java的世界了
1138        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
1139            "([Ljava/lang/String;)V");
1140        if (startMeth == NULL) {
1141            ALOGE("JavaVM unable to find main() in '%s'\n", className);
1142            /* keep going */
1143        } else {
1144            env->CallStaticVoidMethod(startClass, startMeth, strArray);
1145
1146#if 0
1147            if (env->ExceptionCheck())
1148                threadExitUncaughtException(env);
1149#endif
1150        }
1151    }
1152    free(slashClassName);
1153
1154    ALOGD("Shutting down VM\n");
1155    if (mJavaVM->DetachCurrentThread() != JNI_OK)
1156        ALOGW("Warning: unable to detach main thread\n");
1157    if (mJavaVM->DestroyJavaVM() != 0)
1158        ALOGW("Warning: VM did not shut down cleanly\n");
1159}
```
###### 2.2 startVm

只是列出了一些常用的参数,是不是很熟悉，这些都可以在配置文件里面看到
```java

...................
parseRuntimeOption("dalvik.vm.heapstartsize", heapstartsizeOptsBuf, "-Xms", "4m");
716    parseRuntimeOption("dalvik.vm.heapsize", heapsizeOptsBuf, "-Xmx", "16m");
717
718    parseRuntimeOption("dalvik.vm.heapgrowthlimit", heapgrowthlimitOptsBuf, "-XX:HeapGrowthLimit=");
719    parseRuntimeOption("dalvik.vm.heapminfree", heapminfreeOptsBuf, "-XX:HeapMinFree=");
720    parseRuntimeOption("dalvik.vm.heapmaxfree", heapmaxfreeOptsBuf, "-XX:HeapMaxFree=");
721    parseRuntimeOption("dalvik.vm.heaptargetutilization",
722                       heaptargetutilizationOptsBuf,
723                       "-XX:HeapTargetUtilization=");
724
...................
/*
1009     * Initialize the VM.
1010     *
1011     * The JavaVM* is essentially per-process, and the JNIEnv* is per-thread.
1012     * If this call succeeds, the VM is ready, and we can start issuing
1013     * JNI calls.
1014     */
        //去创建jvm
1015    if (JNI_CreateJavaVM(pJavaVM, pEnv, &initArgs) < 0) {
1016        ALOGE("JNI_CreateJavaVM failed\n");
1017        return -1;
1018    }
1019


```
###### 2.3 startReg
```java
int AndroidRuntime::startReg(JNIEnv* env)
{
    //设置线程创建javaCreateThreadEtc
    androidSetCreateThreadFunc((android_create_thread_fn) javaCreateThreadEtc);

    env->PushLocalFrame(200);
    //注册jni方法
    if (register_jni_procs(gRegJNI, NELEM(gRegJNI), env) < 0) {
        env->PopLocalFrame(NULL);
        return -1;
    }
    env->PopLocalFrame(NULL);
    return 0;
}
```
###### 2.4 register_jni_procs
注册系统里面的jni方法, 以便上层Api可以调用到Native方法，这里举一个例子
```java
extern int register_android_hardware_Camera(JNIEnv *env);
REG_JNI(register_android_hardware_Camera)
```
extern 是c++中的关键字,代表着不用引入.h 或者 .cpp文件，也可以调用到.意思是别的地方已经定义过了  
//举一个列子camera
register_android_hardware_Camera 这个方法会在下面这个文件里面注册  
/frameworks/base/core/jni/android_hardware_Camera.cpp

```java
6int register_android_hardware_Camera(JNIEnv *env)
1127{
        //注册成员变量
1128    field fields_to_find[] = {
1129        { "android/hardware/Camera", "mNativeContext",   "J", &fields.context },
1130        { "android/hardware/Camera$CameraInfo", "facing",   "I", &fields.facing },
1131        { "android/hardware/Camera$CameraInfo", "orientation",   "I", &fields.orientation },
1132        { "android/hardware/Camera$CameraInfo", "canDisableShutterSound",   "Z",
1133          &fields.canDisableShutterSound },
1134        { "android/hardware/Camera$Face", "rect", "Landroid/graphics/Rect;", &fields.face_rect },
1135        { "android/hardware/Camera$Face", "leftEye", "Landroid/graphics/Point;", &fields.face_left_eye},
1136        { "android/hardware/Camera$Face", "rightEye", "Landroid/graphics/Point;", &fields.face_right_eye},
1137        { "android/hardware/Camera$Face", "mouth", "Landroid/graphics/Point;", &fields.face_mouth},
1138        { "android/hardware/Camera$Face", "score", "I", &fields.face_score },
1139        { "android/hardware/Camera$Face", "id", "I", &fields.face_id},
1140        { "android/graphics/Rect", "left", "I", &fields.rect_left },
1141        { "android/graphics/Rect", "top", "I", &fields.rect_top },
1142        { "android/graphics/Rect", "right", "I", &fields.rect_right },
1143        { "android/graphics/Rect", "bottom", "I", &fields.rect_bottom },
1144        { "android/graphics/Point", "x", "I", &fields.point_x},
1145        { "android/graphics/Point", "y", "I", &fields.point_y},
1146    };
1147
1148    find_fields(env, fields_to_find, NELEM(fields_to_find));
1149
1150    jclass clazz = FindClassOrDie(env, "android/hardware/Camera");
1151    fields.post_event = GetStaticMethodIDOrDie(env, clazz, "postEventFromNative",
1152                                               "(Ljava/lang/Object;IIILjava/lang/Object;)V");
1153
1154    clazz = FindClassOrDie(env, "android/graphics/Rect");
1155    fields.rect_constructor = GetMethodIDOrDie(env, clazz, "<init>", "()V");
1156
1157    clazz = FindClassOrDie(env, "android/hardware/Camera$Face");
1158    fields.face_constructor = GetMethodIDOrDie(env, clazz, "<init>", "()V");
1159
1160    clazz = env->FindClass("android/graphics/Point");
1161    fields.point_constructor = env->GetMethodID(clazz, "<init>", "()V");
1162    if (fields.point_constructor == NULL) {
1163        ALOGE("Can't find android/graphics/Point()");
1164        return -1;
1165    }
1166
1167    // Register native functions
        // camMethods 注册方法，java层的方法和native层方法对应
1168    return RegisterMethodsOrDie(env, "android/hardware/Camera", camMethods, NELEM(camMethods));
1169}

//注册方法
static const JNINativeMethod camMethods[] = {
1025  { "getNumberOfCameras",
1026    "()I",
1027    (void *)android_hardware_Camera_getNumberOfCameras },
1028  { "_getCameraInfo",
1029    "(ILandroid/hardware/Camera$CameraInfo;)V",
1030    (void*)android_hardware_Camera_getCameraInfo },
1031  { "native_setup",
1032    "(Ljava/lang/Object;IILjava/lang/String;)I",
1033    (void*)android_hardware_Camera_native_setup },
1034  { "native_release",
1035    "()V",
1036    (void*)android_hardware_Camera_release },
      }
```


这个地方真正的进入了java的世界,接着是各种系统服务的创建
###### 2.5 zygote main方法执行
在/frameworks/base/core/jni/AndroidRuntime.cpp 里面的start方法会通过反射开始zygote
```java
/*
1129     * Start VM.  This thread becomes the main thread of the VM, and will
1130     * not return until the VM exits.
1131     */
1132    char* slashClassName = toSlashClassName(className != NULL ? className : "");
1133    jclass startClass = env->FindClass(slashClassName);
1134    if (startClass == NULL) {
1135        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
1136        /* keep going */
1137    } else {
            //找到类里面的main函数，进入java的世界
            //http://androidxref.com/9.0.0_r3/xref/frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
1138        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
1139            "([Ljava/lang/String;)V");
1140        if (startMeth == NULL) {
1141            ALOGE("JavaVM unable to find main() in '%s'\n", className);
1142            /* keep going */
1143        } else {
                //c++调用
1144            env->CallStaticVoidMethod(startClass, startMeth, strArray);
1145
1146#if 0
1147            if (env->ExceptionCheck())
1148                threadExitUncaughtException(env);
1149#endif
1150        }
1151    }

/frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
public static void main(String argv[]) {
751        ZygoteServer zygoteServer = new ZygoteServer();
752
753        // Mark zygote start. This ensures that thread creation will throw
754        // an error.
755        ZygoteHooks.startZygoteNoThreadCreation();
756
757        // Zygote goes into its own process group.
758        try {
759            Os.setpgid(0, 0);
760        } catch (ErrnoException ex) {
761            throw new RuntimeException("Failed to setpgid(0,0)", ex);
762        }
763
764        final Runnable caller;
765        try {
766            // Report Zygote start time to tron unless it is a runtime restart
767            if (!"1".equals(SystemProperties.get("sys.boot_completed"))) {
768                MetricsLogger.histogram(null, "boot_zygote_init",
769                        (int) SystemClock.elapsedRealtime());
770            }
771
772            String bootTimeTag = Process.is64Bit() ? "Zygote64Timing" : "Zygote32Timing";
773            TimingsTraceLog bootTimingsTraceLog = new TimingsTraceLog(bootTimeTag,
774                    Trace.TRACE_TAG_DALVIK);
775            bootTimingsTraceLog.traceBegin("ZygoteInit");
776            RuntimeInit.enableDdms();
777
778            boolean startSystemServer = false;
779            String socketName = "zygote";
780            String abiList = null;
781            boolean enableLazyPreload = false;
782            for (int i = 1; i < argv.length; i++) {
783                if ("start-system-server".equals(argv[i])) {
                       //这个地方设置为true，是在native zygote 进程里面孵化出来的
784                    startSystemServer = true;
785                } else if ("--enable-lazy-preload".equals(argv[i])) {
786                    enableLazyPreload = true;
787                } else if (argv[i].startsWith(ABI_LIST_ARG)) {
788                    abiList = argv[i].substring(ABI_LIST_ARG.length());
789                } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
790                    socketName = argv[i].substring(SOCKET_NAME_ARG.length());
791                } else {
792                    throw new RuntimeException("Unknown command line argument: " + argv[i]);
793                }
794            }
795
796            if (abiList == null) {
797                throw new RuntimeException("No ABI list supplied.");
798            }
799             
               //注册监听，以后通过这个这个socket来创建新的进程
800            zygoteServer.registerServerSocketFromEnv(socketName);
801            // In some configurations, we avoid preloading resources and classes eagerly.
802            // In such cases, we will preload things prior to our first fork.
803            if (!enableLazyPreload) {
804                bootTimingsTraceLog.traceBegin("ZygotePreload");
805                EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_START,
806                    SystemClock.uptimeMillis());
                   //加载一些新的资源
807                preload(bootTimingsTraceLog);
808                EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_END,
809                    SystemClock.uptimeMillis());
810                bootTimingsTraceLog.traceEnd(); // ZygotePreload
811            } else {
812                Zygote.resetNicePriority();
813            }
814
815            // Do an initial gc to clean up after startup
816            bootTimingsTraceLog.traceBegin("PostZygoteInitGC");
817            gcAndFinalize();
818            bootTimingsTraceLog.traceEnd(); // PostZygoteInitGC
819
820            bootTimingsTraceLog.traceEnd(); // ZygoteInit
821            // Disable tracing so that forked processes do not inherit stale tracing tags from
822            // Zygote.
823            Trace.setTracingEnabled(false, 0);
824
825            Zygote.nativeSecurityInit();
826
827            // Zygote process unmounts root storage spaces.
828            Zygote.nativeUnmountStorageOnInit();
829
830            ZygoteHooks.stopZygoteNoThreadCreation();
831
832            if (startSystemServer) {
833                Runnable r = forkSystemServer(abiList, socketName, zygoteServer);
834
835                // {@code r == null} in the parent (zygote) process, and {@code r != null} in the
836                // child (system_server) process.
837                if (r != null) {
838                    r.run();
839                    return;
840                }
841            }
842
843            Log.i(TAG, "Accepting command socket connections");
844
845            // The select loop returns early in the child process after a fork and
846            // loops forever in the zygote.
847            caller = zygoteServer.runSelectLoop(abiList);
848        } catch (Throwable ex) {
849            Log.e(TAG, "System zygote died with exception", ex);
850            throw ex;
851        } finally {
852            zygoteServer.closeServerSocket();
853        }
854
855        // We're in the child process and have exited the select loop. Proceed to execute the
856        // command.
857        if (caller != null) {
858            caller.run();
859        }
860    }


static void preload(TimingsTraceLog bootTimingsTraceLog) {
124        Log.d(TAG, "begin preload");
125        bootTimingsTraceLog.traceBegin("BeginIcuCachePinning");
126        beginIcuCachePinning();
127        bootTimingsTraceLog.traceEnd(); // BeginIcuCachePinning
128        bootTimingsTraceLog.traceBegin("PreloadClasses");
           //加载/system/etc/preloaded-classes 这个列表里面的类，会用到Class.forName方法
           Class.forName(xxx.xx.xx);的作用是要求JVM查找并加载指定的类，也就是说JVM会执行该类的静态代码段
129        preloadClasses();
130        bootTimingsTraceLog.traceEnd(); // PreloadClasses
131        bootTimingsTraceLog.traceBegin("PreloadResources");
132        preloadResources();
133        bootTimingsTraceLog.traceEnd(); // PreloadResources
134        Trace.traceBegin(Trace.TRACE_TAG_DALVIK, "PreloadAppProcessHALs");
135        nativePreloadAppProcessHALs();
136        Trace.traceEnd(Trace.TRACE_TAG_DALVIK);
137        Trace.traceBegin(Trace.TRACE_TAG_DALVIK, "PreloadOpenGL");
138        preloadOpenGL();
139        Trace.traceEnd(Trace.TRACE_TAG_DALVIK);
           //一些动态库
140        preloadSharedLibraries();
141        preloadTextResources();
142        // Ask the WebViewFactory to do any initialization that must run in the zygote process,
143        // for memory sharing purposes.
144        WebViewFactory.prepareWebViewInZygote();
145        endIcuCachePinning();
146        warmUpJcaProviders();
147        Log.d(TAG, "end preload");
148
149        sPreloadComplete = true;
150    }
```
这里引用gityuan先生的图，侵权请联系我
zygote进程内加载了preload()方法中的所有资源，当需要fork新进程时，采用copy on write技术，如下：
![avatar](https://github.com/skypx/BlogResource/raw/master/androidsystem/zygote_fork.jpg)

###### 2.6 forkSystemServer方法
```java
private static Runnable forkSystemServer(String abiList, String socketName,
658            ZygoteServer zygoteServer) {
659        long capabilities = posixCapabilitiesAsBits(
660            OsConstants.CAP_IPC_LOCK,
661            OsConstants.CAP_KILL,
662            OsConstants.CAP_NET_ADMIN,
663            OsConstants.CAP_NET_BIND_SERVICE,
664            OsConstants.CAP_NET_BROADCAST,
665            OsConstants.CAP_NET_RAW,
666            OsConstants.CAP_SYS_MODULE,
667            OsConstants.CAP_SYS_NICE,
668            OsConstants.CAP_SYS_PTRACE,
669            OsConstants.CAP_SYS_TIME,
670            OsConstants.CAP_SYS_TTY_CONFIG,
671            OsConstants.CAP_WAKE_ALARM,
672            OsConstants.CAP_BLOCK_SUSPEND
673        );
674        /* Containers run without some capabilities, so drop any caps that are not available. */
675        StructCapUserHeader header = new StructCapUserHeader(
676                OsConstants._LINUX_CAPABILITY_VERSION_3, 0);
677        StructCapUserData[] data;
678        try {
679            data = Os.capget(header);
680        } catch (ErrnoException ex) {
681            throw new RuntimeException("Failed to capget()", ex);
682        }
683        capabilities &= ((long) data[0].effective) | (((long) data[1].effective) << 32);
684
685        /* Hardcoded command line to start the system server */
           //system_server的group id，uid名字
686        String args[] = {
687            "--setuid=1000",
688            "--setgid=1000",
689            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1023,1024,1032,1065,3001,3002,3003,3006,3007,3009,3010",
690            "--capabilities=" + capabilities + "," + capabilities,
691            "--nice-name=system_server",
692            "--runtime-args",
693            "--target-sdk-version=" + VMRuntime.SDK_VERSION_CUR_DEVELOPMENT,
694            "com.android.server.SystemServer",
695        };
696        ZygoteConnection.Arguments parsedArgs = null;
697
698        int pid;
699
700        try {
701            parsedArgs = new ZygoteConnection.Arguments(args);
702            ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
703            ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);
704
705            boolean profileSystemServer = SystemProperties.getBoolean(
706                    "dalvik.vm.profilesystemserver", false);
707            if (profileSystemServer) {
708                parsedArgs.runtimeFlags |= Zygote.PROFILE_SYSTEM_SERVER;
709            }
710
711            /* Request to fork the system server process */
               //fork子进程，用于运行system_server
712            pid = Zygote.forkSystemServer(
713                    parsedArgs.uid, parsedArgs.gid,
714                    parsedArgs.gids,
715                    parsedArgs.runtimeFlags,
716                    null,
717                    parsedArgs.permittedCapabilities,
718                    parsedArgs.effectiveCapabilities);
719        } catch (IllegalArgumentException ex) {
720            throw new RuntimeException(ex);
721        }
722
723        /* For child process */
724        if (pid == 0) {
725            if (hasSecondZygote(abiList)) {
726                waitForSecondaryZygote(socketName);
727            }
728
729            zygoteServer.closeServerSocket();
730            return handleSystemServerProcess(parsedArgs);
731        }
732
733        return null;
734    }
```

在zygote的main方法中会执行循环操作，一方面保证zygote不退出，一方面监听是否有创建进程的消息
```java
Runnable runSelectLoop(String abiList) {
174        ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
175        ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();
176
177        fds.add(mServerSocket.getFileDescriptor());
178        peers.add(null);
179
180        while (true) {
181            StructPollfd[] pollFds = new StructPollfd[fds.size()];
182            for (int i = 0; i < pollFds.length; ++i) {
183                pollFds[i] = new StructPollfd();
184                pollFds[i].fd = fds.get(i);
185                pollFds[i].events = (short) POLLIN;
186            }
187            try {
                   //处理轮询状态，当pollFds有事件到来则往下执行，否则阻塞在这里,我也没太看懂这个地方
                   //暂且理解为类似loop那种机制吧，如果有客户端请求就会向下走
188                Os.poll(pollFds, -1);
189            } catch (ErrnoException ex) {
190                throw new RuntimeException("poll failed", ex);
191            }
192            for (int i = pollFds.length - 1; i >= 0; --i) {
193                if ((pollFds[i].revents & POLLIN) == 0) {
194                    continue;
195                }
196
197                if (i == 0) {
                    即fds[0]，代表的是sServerSocket，则意味着有客户端连接请求；
                    // 则创建ZygoteConnection对象,并添加到fds。
198                    ZygoteConnection newPeer = acceptCommandPeer(abiList);
199                    peers.add(newPeer);
200                    fds.add(newPeer.getFileDesciptor());
201                } else {
202                    try {
203                        ZygoteConnection connection = peers.get(i);
                           会调用到Zygote里面的方法 Zygote.forkAndSpecialize,
                           nativeForkAndSpecialize
204                        final Runnable command = connection.processOneCommand(this);
205
206                        if (mIsForkChild) {
207                            // We're in the child. We should always have a command to run at this
208                            // stage if processOneCommand hasn't called "exec".
209                            if (command == null) {
210                                throw new IllegalStateException("command == null");
211                            }
212
213                            return command;
214                        } else {
215                            // We're in the server - we should never have any commands to run.
216                            if (command != null) {
217                                throw new IllegalStateException("command != null");
218                            }
219
220                            // We don't know whether the remote side of the socket was closed or
221                            // not until we attempt to read from it from processOneCommand. This shows up as
222                            // a regular POLLIN event in our regular processing loop.
223                            if (connection.isClosedByPeer()) {
224                                connection.closeSocket();
225                                peers.remove(i);
226                                fds.remove(i);
227                            }
228                        }
229                    } catch (Exception e) {
230                        if (!mIsForkChild) {
231                            // We're in the server so any exception here is one that has taken place
232                            // pre-fork while processing commands or reading / writing from the
233                            // control socket. Make a loud noise about any such exceptions so that
234                            // we know exactly what failed and why.
235
236                            Slog.e(TAG, "Exception executing zygote command: ", e);
237
238                            // Make sure the socket is closed so that the other end knows immediately
239                            // that something has gone wrong and doesn't time out waiting for a
240                            // response.
241                            ZygoteConnection conn = peers.remove(i);
242                            conn.closeSocket();
243
244                            fds.remove(i);
245                        } else {
246                            // We're in the child so any exception caught here has happened post
247                            // fork and before we execute ActivityThread.main (or any other main()
248                            // method). Log the details of the exception and bring down the process.
249                            Log.e(TAG, "Caught post-fork exception in child process.", e);
250                            throw e;
251                        }
252                    } finally {
253                        // Reset the child flag, in the event that the child process is a child-
254                        // zygote. The flag will not be consulted this loop pass after the Runnable
255                        // is returned.
256                        mIsForkChild = false;
257                    }
258                }
259            }
260        }
261    }
```

###### 总结部分

这里引用gityuan先生的图  
![avatar](https://github.com/skypx/BlogResource/raw/master/androidsystem/zygote_start.jpg)

然后这个地方总结的很好:
1. 解析init.zygote.rc中的参数，创建AppRuntime并调用AppRuntime.start()方法；  
2. 调用AndroidRuntime的startVM()方法创建虚拟机，再调用startReg()注册JNI函数；  
3. 通过JNI方式调用ZygoteInit.main()，第一次进入Java世界；  
4. registerZygoteSocket()建立socket通道，zygote作为通信的服务端，用于响应客户端请求；  
5. preload()预加载通用类、drawable和color资源、openGL以及共享库以及WebView，用于提高app启动效率；  
6. zygote完毕大部分工作，接下来再通过startSystemServer()，fork得力帮手system_server进程，也是上层framework的运行载体。  
7. zygote功成身退，调用runSelectLoop()，随时待命，当接收到请求创建新进程请求时立即唤醒并执行相应工作。  

在次感谢gityuan先生。
