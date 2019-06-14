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
  
  2. 设置`InputConfiguration`, 还有`ImageWrite`
  3. 在`onImageAvailable` 可用的时候把image图片放入队列中, 还有`TotalCaptureResult`放入对列中
  4. 如果拍照的时候.set`ImageWrite.queueInputImage`,  同时 cameradevice.createReprocessCaptureRequest(TotalCaptureResult)
  5. 这个时候去拍照, session.capture(..)

# 总结
* 不管是用API1 的方式还是API2的方式去ZSL拍照. 都是为了提升拍照的速度, 和普通的拍照的较大区别是先处理和后处理
  普通的拍照是把图片在HAL层先处理白平衡.降噪等的处理. 而ZSL拍照是先获取YUV数据,然后在进行InputConfig的后处理
  其实ImageWrite write到了InputConfig里面
