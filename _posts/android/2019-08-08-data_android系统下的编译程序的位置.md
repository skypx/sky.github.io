---
layout: post
title: Android 系统下自定义的可程序的存放位置
category: Android
tags: android系统下的自定义二进制文件的存放位置
---
* content
{:toc}

>如何正确的存放自己编写的二进制文件的路径

### 前言

自己写了一个简单的c程序，然后放在系统的根目录下，执行mm编译，结果
```java
error: unused parameter 'argc' [-Werror,-Wunused-parameter]
```

仔细检查了程序，发现并没有错，只是有一个参数我并没有用到, 上网Google之
关键字 **Werror**

**Werror**  :: all warnings being treated as errors

所有的警告都被视为是错误信息

网上各种解决方案：只需要找到相应的Makefile，去掉编译选项中的 -Werror 即可。

可是我的makefile里面并没有加任何Werror的选项，这是怎么回事，那就在源码里面搜索
关键字吧，找到在一个.mk文件里面做了判断,类似这样::

if(什么目录下)
然后添加**-Werror**
else
不添加

恍然大悟,是我放的目录不对啊。不应该放在根目录下的, 该地方，放在vendor下，编译成功.

**由此得出结论，网上的未必靠谱，(只是给你提供了思路要去那个方向)要根据自己的代码实际情况去
看自己的code.**
