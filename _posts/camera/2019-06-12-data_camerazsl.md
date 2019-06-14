---
layout: post
title: Camera ZSL 拍照
category: Camera
tags: Camera 拍照
---
* content
{:toc}

# 什么是ZSL
* ZSL (zero shutter lag)：零秒延迟

* 在日常生活中，使用手机camera拍照的时候往往会有一些延迟的体验。ZSL，就是为了消除这种延迟，提供一种“拍即视”的体验而被开发出来。

[ZSL链接](http://blog.chinaunix.net/uid-7213935-id-5753468.html)


# 针对API1来说
* 高通会提供关于ZSL的patch给厂商. 厂商通过设置 `setZSLMode` 默认是隐藏的方法. 具体对于队列的维护
  放在了Framework层去处理

# 针对API2来说
* Goolge建议对于缓存队列的处理放在App 层来处理 具体可以参考如下的链接
  [Camera2 ZSL](http://androidxref.com/9.0.0_r3/search?q=%22zsl%22&project=packages)

  这里要注意的是需要设置模板`TEMPLATE_ZERO_SHUTTER_LAG` 来通知HAL切换模板

# 具体的code方式

  1. 创建Session的方式 `createReprocessableCaptureSessionByConfigurations`
  ```java
  camera.createReprocessableCaptureSessionByConfigurations(inputConfiguration, outputs,sessionListener, handler);
  ```

  2. 设置`InputConfiguration`, 还有`ImageWrite`
  ```java
  InputConfiguration inputConfiguration = new InputConfiguration
                                Size.width(), Size.height(),
                                ImageFormat.YUV_420_888;
  ```
  3. 创建新的ImageReader来配置和InputConfiguration一样的参数
  ```java
  ImageReader mZslImageReader = new ImageReader(Size.width(), Size.height(),
                                ImageFormat.PRIVATE);
  ```
  4. 把zslIamgereader传入setRepeatingBurst, 这样就可以获取这个流了
  ```java
  CaptureRequest.Builder reqBuilder = cameraDevice.createCaptureRequest(templateType);
  reqBuilder.addTarget(mZslImageReader.getSurface);
  mCaptureSession.setRepeatingBurst(reqBuilder....);
  ```
  5. 设置ZslImageReader的callback. onImageAvailable 可用的时候放入对列
  ```java
  private final LinkedBlockingQueue<Image> mQueue = new LinkedBlockingQueue<Image>();
  mQueue.add(Image)
  ```
  [参考CTS](http://androidxref.com/9.0.0_r3/xref/cts/tests/camera/utils/src/android/hardware/camera2/cts/CameraTestUtils.java#286)
  
  6. 设置ImageWriter
  ```java
  这个surface其实就是设置inputconfig的时候底层创建的surface
  ImageWriter mImageWriter = ImageWriter.newInstance(session.getInputSurface(), 2);
  ```
  7. 把LinkedBlockingQueue里面的image送给下面去重新操作, 同时把result也放入进去
  ```java
  mImageWriter.queueInputImage(mQueue.getImage());//伪代码
  captureRequestBuilder = mCameraDevice.
                                createReprocessCaptureRequest(mQueue.getResult());
  ```
  8. 重新去拍照, 用图片的ImageReader
  ```java
  captureRequestBuilder.addTarget(mCaptureImageReader.getSurface());
  mCaptureSession.capture(captureRequestBuilder.build(),
                                captureRequestCallback, mCameraDeviceHandler.getDeviceThreadHandler());
  ```

  >我整理了一张图片大致流程是这样的
  ![avatar](https://github.com/skypx/BlogResource/raw/master/camera/zslnew.png)



  ## 参考了CTS相关代码
  [CameraCts - CameraTestUtils](http://androidxref.com/9.0.0_r3/xref/cts/tests/camera/utils/src/android/hardware/camera2/cts/CameraTestUtils.java)

  [CameraCts - ReprocessCaptureTest](http://androidxref.com/9.0.0_r3/xref/cts/tests/camera/src/android/hardware/camera2/cts/ReprocessCaptureTest.java)

# 总结
* 不管是用API1 的方式还是API2的方式去ZSL拍照. 都是为了提升拍照的速度, 和普通的拍照的较大区别是先处理和后处理
  普通的拍照是把图片在HAL层先处理白平衡.降噪等的处理. 而ZSL拍照是先获取YUV数据,然后在进行InputConfig的后处理
  其实ImageWrite write到了InputConfig里面

  ## 下步计划, 抽出snapdragon, camera2 ZSL 部分
