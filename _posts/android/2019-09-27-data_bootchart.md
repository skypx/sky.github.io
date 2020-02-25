---
layout: post
title:  bootchart性能分析工具
category: Android
tags: bootchart
---
* content
{:toc}
#### bootchart简介
bootchart 可为整个系统提供所有进程的 CPU 和 I/O 负载细分。该工具不需要重建系统映像，
可以用作进入 systrace 之前的快速健全性检查。(systrace)以后在聊


#### 使用方法

首先在电脑上安装bootchart工具  
```java
sudo apt-get install bootchart
```

1. 启用bootchart

```java
adb shell 'touch /data/bootchart/enabled'
adb reboot
```

2. 获取图表  
第一种方式：  
进入源码目录, 启动下面的脚本，会生成一个bootchat.png的图片
$ANDROID_BUILD_TOP/system/core/init/grab-bootchart.sh

第二种方式：  
进入/data/bootchart/ 目录里面， 执行 tar -czf bootchart.tar * 命令，把下面的资源打包，  
然后pull出来, 接着用bootchat bootchart.tar 会自动生成一张bootchat.png图片

#### 下面是我从google pixel2手机上抓取的一个启动时间图
![avatar](https://github.com/skypx/BlogResource/raw/master/other/bootchart.png)

整个图表以时间线为横轴，图标上方为 CPU 和 磁盘的利用情况，下方是各进程的运行状态条，  
显示各个进程的开始时间与结束时间以及对 CPU、I/O 的利用情况，我们关心的各个进程的运行时间以及  
CPU 的使用情况，进而优化系统。  
这里统计的是开启init进程到启动完成的时间, 并没有kernel的启动时间.
同样有的设备可以通过boot_completed这个时间段来确定,adb shell输入一下命令，  
dmesg | grep -i boot_completed  
[   21.997797] init: processing action (sys.boot_completed=1) from (/init.rc:705)  
从上面的意思来看应该是从init进程到boot_completed的时间为22s左右.如果要统计开机时间，还要加上kernel的时间
