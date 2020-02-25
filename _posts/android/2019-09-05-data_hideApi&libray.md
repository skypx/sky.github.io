---
layout: post
title:  源码下调用Hiden Api以及对mk文件里面属性的一些思考
category: Android
tags: 源码下调用Hiden Api以及对mk文件里面属性的一些思考
---
* content
{:toc}

### 原因
>最近在做HIDL相关的工作，HIDL暂且不谈， 现在我编译完HIDL的接口之后会给上层App  
提供一个jar包，以供App来调用HIDL接口. 当时我的想法很简单，我编译完之后用studio来  
引入jar包。然后我就可以美滋滋的调用了.

### 美梦泡汤
熟悉的在out/target/...目录下找到生成的jar包.add library. 在类里面直接引用jar包   
里面的方法. 走起. run。。。。  
![avatar](https://github.com/skypx/BlogResource/raw/master/other/hiden.png)








恩? 怎么会找不到呢?看源码，看看这个究竟是一个什么类. 结果是一个hide的

![avatar](https://github.com/skypx/BlogResource/raw/master/other/hwbinderhide.png)

原来SDK里面并没有这个类，因为是hide的。是不会提供API的.那问题来了。应该怎么用呢.别人是怎么用hide的?  
google之，公认的方法:   
**1:** 强大的反射功能  
**2:** 编译framework jar生成 jar包   
**3:** 在系统下编译. 切配置mk文件   

第一种，第二种我就不多说了。我们就说下第三种.  
```java
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
LOCAL_PACKAGE_NAME := HIDLdemo
LOCAL_SRC_FILES := $(call all-java-files-under, src)
LOCAL_MODULE_TAGS :=optional
LOCAL_PROGUARD_ENABLED := disabled
LOCAL_STATIC_JAVA_LIBRARIES := android-support-v4
LOCAL_STATIC_JAVA_LIBRARIES += android-support-v7-appcompat
#LOCAL_STATIC_JAVA_LIBRARIES += android-support-v7-gridlayout
#LOCAL_STATIC_JAVA_LIBRARIES += android-support-v13
LOCAL_STATIC_JAVA_LIBRARIES += android.hardware.gunder-V1.0-java-static
#LOCAL_PREBUILT_STATIC_JAVA_LIBRARIES := pxwen:libs/classes.jar
LOCAL_DEX_PREOPT = false
LOCAL_RESOURCE_DIR := $(LOCAL_PATH)/res
LOCAL_RESOURCE_DIR += prebuilts/sdk/current/support/v7/appcompat/res
#LOCAL_RESOURCE_DIR += prebuilts/sdk/current/support/v7/gridlayout/res


#LOCAL_SDK_VERSION := current
LOCAL_CERTIFICATE := platform
#LOCAL_PRIVILEGED_MODULE := true
LOCAL_AAPT_FLAGS := --auto-add-overlay
LOCAL_AAPT_FLAGS += --extra-packages android.support.v7.appcompat
#LOCAL_STATIC_JAVA_LIBRARIES += android.hardware.fingerprint-V1.0-java-static
#LOCAL_SRC_FILES := $(call all-subdir-java-files)
include $(BUILD_PACKAGE)

```

主要在于这行
**LOCAL_SDK_VERSION := current**
这行加的话就会引用当前版本的sdk来进行编译，而不会引用framework下的去编译。去掉这行就可以引用hide的api了


然后对mk属性里面的一些思考
LOCAL_STATIC_JAVA_LIBRARIES 引用的静态库是从哪里来的？如何引用一个第三方的jar在mk里面?      
静态库是会打包进应用里面的.那就是说系统里面肯定会有这么一份的库文件来提供，那我们就去找例如   
android-support-v4，android-support-v7-appcompat

果不其然, 在Framework下是有这个文件的
<http://androidxref.com/8.1.0_r33/xref/frameworks/support/v4/>
![avatar](https://github.com/skypx/BlogResource/raw/master/other/supportv4.png)

同时查找out目录下的common文件夹里面查询v4.可以看到生成了jar文件. 而我们的LOCAL_STATIC_JAVA_LIBRARIES   
其实就是链接到了这个jar包
![avatar](https://github.com/skypx/BlogResource/raw/master/other/outsupportv4.png)

上面的是引用系统的，比如我们自己的第三方包怎么来引进来呢?

**LOCAL_STATIC_JAVA_LIBRARIES** 取jar库的别名，可以任意取值；

**LOCAL_PREBUILT_STATIC_JAVA_LIBRARIES** 指定prebuiltjar库的规则，格式：别名:jar文件路径。注意：别名一定要与LOCAL_STATIC_JAVA_LIBRARIES里所取的别名一致，且不含.jar；jar文件路径一定要是真实的存放第三方jar包的路径。

编译用 **BUILD_MULTI_PREBUILT**。

### 延伸点 关于Android下的编译指令
Android 系统提供了三种指令用于编译，他们分别为make、mmm、mm，这三个指令编译的优缺点如下：

1. **make**：不带任何参数，用于编译整个系统，编译时间比较长，除非是进行初次编译否则不建议此种做法；   
例如：make camera2  z这种模式对应于单个模块的编译。它的优点是：会把该模块依赖的其他模块一起跟着编译。例如：make libmedia 就会把libmedia依赖库全部编译好。当然缺点也会很明显，那就是它会搜索整个源码来定位MediaProvider 模块所使用的Android.mk文件。并且还要判断该模块依赖的其他模块是否有修改。所以编译时间比较长

2. **mmm**  packages/app/Camera2r:该命令编译指定目录下的目标模块，而不编译它所依赖其他模块。所以，若是初次编译，采用此种模式编译一个模块往往会报错，错误的原因就在于它依赖的其他模块没有一起编译。

3. **mm** 这种编译方式一般需要cd 进入packages/app/Camera2目录，然后执行mm指令。该命令会编译当前目录下的模块。它和mmm一样，只编译目标模块。mm和mmm编译的速度都很快。

如果用 make 模块名 这种方式会编译你在make依赖下的所有模块的. 下面我们做个实验编译packages/app/Camera2来做个实验
```java
LOCAL_STATIC_JAVA_LIBRARIES := android-support-v13
8LOCAL_STATIC_JAVA_LIBRARIES += android-ex-camera2-portability
9LOCAL_STATIC_JAVA_LIBRARIES += xmp_toolkit
10LOCAL_STATIC_JAVA_LIBRARIES += glide
11LOCAL_STATIC_JAVA_LIBRARIES += guava
12LOCAL_STATIC_JAVA_LIBRARIES += jsr305
```
之前我没有编译过Camera2相关的代码，我进入目录之后mm编译
![avatar](https://github.com/skypx/BlogResource/raw/master/other/camera2error.png)


然后我用make Camera2 编译，顺利编过.然后查看out目录下是否android-ex-camera2-portability文件生成
原来是没有的.

![avatar](https://github.com/skypx/BlogResource/raw/master/other/android-ex-camera2-portability.png)



### 总结
在mk里面要指定的依赖库一定在源码项目里面有的。即使没有我们也要创造条件给编译出来，这样我们的应用才能引用到.   
以前看mk的时候总是不知道为什么要这样加。这次算是有了一个初步的认识吧.未来我们要去探索更多的东西，编译流程   
以及mk文件
