---
layout: post
title: Android 系统下的log文件
category: Android
tags: android系统下的 log文件
---
* content
{:toc}

>Android系统中的log都有那些，如何在JNI或者自己在Android系统下开发的程序中打印Log

### 前言

Android 系统中的Log分为两类，一种是java层的， 一种是Native层的，

## Java层的Log

**Log.d**  
**Log.e**  
**Log.v**  
**Log.i**  

在此不做过多赘述, 最终还是通过jni调到系统的

## Native层的Log, 一般JNI就是通过这种方式调用的

### __android_log_print方式
```java
1.源码位置,系统编译出liblog库
/system/core/liblog/include/android/log.h

2.在mk文件里面引用这个库文件
LOCAL_SHARED_LIBRARIES := \
	liblog

3.在c或者c++代码中引用 #include <android/log.h>

ANDROID_LOG_DEBUG log.h文件里面定义的枚举，可以直接用,
__android_log_print(ANDROID_LOG_ERROR, LOG_TAG, __VA_ARGS__)
__VA_ARGS__  这个是可变参数接收的类型

__android_log_print(ANDROID_LOG_DEBUG, "skyxiao", "test");
一般定义一个宏来表示可变参数，这个可以自己上网搜下，很简单的
```

### SLOGD的方式
```java
1.源码位置,系统编译出libcutils库文件
/system/core/include/log/log_system.h
2.引入头文件
#include <cutils/log.h>
3.SLOGD("xxxxx");
```

### ALOGE的方式
```java
定义宏
#define LOG_TAG "test"
为什么要这样定义？
参考源码:第二个参数是LOG_TAG 也可以不用定义
#ifndef ALOGE
242#define ALOGE(...) ((void)ALOG(LOG_ERROR, LOG_TAG, __VA_ARGS__))

#### 第一种方式:
上面的两种方式引入的库和头文件之后，可以直接用
ALOGE("xxxxx");
#### 第二种方式:
#include <utils/Log.h>
mk中添加
LOCAL_SHARED_LIBRARIES:= libcutils libutils liblog
ALOGE("xxxxx");
ALOGE(“%s(), %d”,__FUNCTION__,__LINE__);
可参考这个链接
<https://blog.csdn.net/u010164190/article/details/78659503>
```
