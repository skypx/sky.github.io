---
layout: post
title: Camera API1 - API2
category: Camera
tags: Camera API 区别
---
* content
{:toc}

# API 概述

Android5.0之后，增加了Cameraapi2接口，api2和api1在调用流程上差异很大，功能上api2提供了更加细致的控制功能，api1子系统被设计为具备高级控制功能的黑盒，能够发出请求，但无法控制图像的缓冲区和元数据，无法再帧层面控制传感器的特性，无法通过访问和修改元数据信息（如3A信息）对捕获的帧应用任何增强功能，重新设计api2旨在大幅提高应用程序控制摄像头子系统的能力，性能也有一定的提升。

# API1 和 API2的流程对比

![avatar](https://github.com/skypx/BlogResource/raw/master/camera/ape_fwk_camera.png)

* 从图中可以看出在API1中java-> jni -> camera service -> Hal, 但是在API2中 java -> camera service -> Hal
  为什么呢? 这个地方我们可以单独放在后面讲
  就是说java不通过jni可以直接和c++通信. 类似aidl的方式, 链接在下面
  <https://android.googlesource.com/platform/system/tools/aidl/+/brillo-m10-dev/docs/aidl-cpp.md>

# Java 层代码比较
# Camera API1 主要类

* Camera api1调用流程比较直观，也利于理解一些，应用程序实例化camera这一个类就可以控制摄像头子系统
  1. Camera，api1主要类。通过类的这些startPreview、takePicture、AutoFocus等标准的接口实现
  2. Parameters，参数设置，通过camera内部类Parameters操作实现，如设置帧率，设置图片格式，预览尺寸等
  3. CameraInfo，元数据获取，通过camera内部类CameraInfo实现，这里只能获取不能修改

* Camera API2 主要类
  1. Camera api2在应用层的调用要复杂一些，定义的类也比较多，这也说明对摄像头子系统控制的更加细腻了
  2. CameraManager：摄像头管理器。这是一个全新的系统管理器，专门用于检测系统摄像头、打开系统摄像头。除此之外，调用CameraManager的getCameraCharacteristics(String)方法即可获取指定摄像头的相关特性
  3. CameraCharacteristics：摄像头特性。该对象通过CameraManager来获取，用于描述特定摄像头所支持的各种特性
  4. CameraDevice：代表系统摄像头。该类的功能类似于早期的Camera类
  5. CameraCaptureSession：这是一个非常重要的API，当程序需要预览、拍照时，都需要先通过该类的实例创建Session。而且不管预览还是拍照， 也都是由该对象的方法进行控制的，其中控制预览的方法为setRepeatingRequest()；控制拍照的方法为capture()
  6. 应用程序对整个摄像头管道的控制能力得到提升。每个拍摄请求会产生一个带有拍摄元数据的结果对象和一些图像数据缓 冲区

# FW代码
  后续分析
