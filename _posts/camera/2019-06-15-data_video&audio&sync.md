---
layout: post
title: Video and Audio 的同步处理
category: Camera
tags: Video and Audio
---
* content
{:toc}

>前言之需要了解的东西

* MediaCodec
  MediaCodec类Android提供的用于访问低层多媒体编/解码器接口，它是Android低层多媒体架构的一部分，通常与MediaExtractor、MediaMuxer、AudioTrack结合使用，能够编解码诸如H.264、H.265、AAC、3gp等常见的音视频格式。广义而言，MediaCodec的工作原理就是处理输入数据以产生输出数据。具体来说，MediaCodec在编解码的过程中使用了一组输入/输出缓存区来同步或异步处理数据：首先，客户端向获取到的编解码器输入缓存区写入要编解码的数据并将其提交给编解码器，待编解码器处理完毕后将其转存到编码器的输出缓存区，同时收回客户端对输入缓存区的所有权；然后，客户端从获取到编解码输出缓存区读取编码好的数据进行处理，待处理完毕后编解码器收回客户端对输出缓存区的所有权。不断重复整个过程，直至编码器停止工作或者异常退出

* MediaMuxer
  就是对音频和视频的混合. 来生成MP4 或 其它格式的文件

没有数据源,比如是从surface出来的
MediaCodec is typically used like this in asynchronous mode:
```java
MediaCodec codec = MediaCodec.createByCodecName(name);
 MediaFormat mOutputFormat; // member variable
 codec.setCallback(new MediaCodec.Callback() {
  @Override
  void onInputBufferAvailable(MediaCodec mc, int inputBufferId) {
    ByteBuffer inputBuffer = codec.getInputBuffer(inputBufferId);
    // fill inputBuffer with valid data
    …
    codec.queueInputBuffer(inputBufferId, …);
  }

  @Override
  void onOutputBufferAvailable(MediaCodec mc, int outputBufferId, …) {
    ByteBuffer outputBuffer = codec.getOutputBuffer(outputBufferId);
    MediaFormat bufferFormat = codec.getOutputFormat(outputBufferId); // option A
    // bufferFormat is equivalent to mOutputFormat
    // outputBuffer is ready to be processed or rendered.
    …
    codec.releaseOutputBuffer(outputBufferId, …);
  }

  @Override
  void onOutputFormatChanged(MediaCodec mc, MediaFormat format) {
    // Subsequent data will conform to new format.
    // Can ignore if using getOutputFormat(outputBufferId)
    mOutputFormat = format; // option B
  }

  @Override
  void onError(…) {
    …
  }
 });
 codec.configure(format, …);
 mOutputFormat = codec.getOutputFormat(); // option B
 codec.start();
 // wait for processing to complete
 codec.stop();
 codec.release();
```

个人认为在有数据源的情况下用这个, 比如已经有了图片要合成视频之类的
MediaCodec is typically used like this in synchronous mode:
```java
MediaCodec codec = MediaCodec.createByCodecName(name);
 codec.configure(format, …);
 MediaFormat outputFormat = codec.getOutputFormat(); // option B
 codec.start();
 for (;;) {
  int inputBufferId = codec.dequeueInputBuffer(timeoutUs);
  if (inputBufferId >= 0) {
    ByteBuffer inputBuffer = codec.getInputBuffer(…);
    // fill inputBuffer with valid data
    …
    codec.queueInputBuffer(inputBufferId, …);
  }
  int outputBufferId = codec.dequeueOutputBuffer(…);
  if (outputBufferId >= 0) {
    ByteBuffer outputBuffer = codec.getOutputBuffer(outputBufferId);
    MediaFormat bufferFormat = codec.getOutputFormat(outputBufferId); // option A
    // bufferFormat is identical to outputFormat
    // outputBuffer is ready to be processed or rendered.
    …
    codec.releaseOutputBuffer(outputBufferId, …);
  } else if (outputBufferId == MediaCodec.INFO_OUTPUT_FORMAT_CHANGED) {
    // Subsequent data will conform to new format.
    // Can ignore if using getOutputFormat(outputBufferId)
    outputFormat = codec.getOutputFormat(); // option B
  }
 }
 codec.stop();
 codec.release();
```


>问题 请看下图, 从Hal上来的数据第一秒的时候是无效的video的, 结束的时候要求多录一秒钟

![avatar](https://github.com/skypx/BlogResource/raw/master/camera/video_audio_sync.png)

## 问题描述
正常情况下,如果录像开启的话,Video和Audio是同时传上来的, 然后同时合成, 类似mediarecorder
这样的话就不用管, 视频和音频的同步问题了. 因为基本不会有延迟,
但是假如开始的一秒钟我没有video数据.但是停止的时候是以audio停止的, 这样的话我的视频不就少了
一秒的时间, 并且前面的video数据是垃圾帧数据

## 思考,

那假如我多录一秒, 并且让前面的垃圾帧去掉在合成不就可以了,

## 核心思路(涉及公司代码, 不能上传code)
```java
if (mFirstVideoFrameTimeUs < 0) {
                //136899194595
                mFirstVideoFrameTimeUs = info.presentationTimeUs;
                //第一帧为0
                info.presentationTimeUs = 0;
//第一帧解码, return
                queueBuffer()
                return
            }

//每次来算时间的gap 136899227943 这个数在增大,帧的时间戳肯定是在增大
long timeSinceVideoStart = info.presentationTimeUs - mFirstVideoFrameTimeUs;
//33348 是33毫秒 比如1秒30帧, 1帧就是33毫秒左右
info.presentationTimeUs = timeSinceVideoStart - mVideoOffset;

接着我们就可以拿出来一副这样的数据
0         1         2          3          4
0         33348     66695      100042     133397
每一帧到第一帧的时间

```
为什么要计算时间, 因为我们记录了开始的时间和结束的时间,然后我们录了多长时间也知道,停止的时候我们不就可以达到多录的目的了.
