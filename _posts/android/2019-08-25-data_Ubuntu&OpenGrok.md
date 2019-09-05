---
layout: post
title:  Ubuntu下搭建OpenGrok浏览Android源码
category: Android
tags: opengrok浏览android源码
---
* content
{:toc}

>Ubuntu下用opengrok部署Android源码

>前言  
浏览代码的工具很多,像AndroidStudio, source insight等,但是像studio设置过于麻烦，并且还对内存有要求,source insight付费的且对新手不太友好.所以我们就用免费开源轻巧的神器.
opengrok,下面我们来一步一步来实现，保证是你看到的最详细的教程

### 1.下载Android源码参考下面的链接
Android 官方 <https://source.android.com/setup/build/downloading> 需要自带梯子  
国内清华大学镜像 <https://mirrors.tuna.tsinghua.edu.cn/help/AOSP/>  

下载完成之后随便起一个名字比如我的是**sourcecode19.02**

### 2.安装JDK
sudo apt-get install openjdk-8-jdk

### 3.下载tomcat
<https://tomcat.apache.org/download-90.cgi> 我是用的9.0的

### 4.下载opengrok
<https://github.com/oracle/opengrok/releases> 我是用的1.3.1

### 5.下载生成索引的工具(universal-ctags)
不要再使用 Exuberant ctags，因为已经不再维护更新，对于新版本的Opengrok支持度不好  

```java
1. git clone https://github.com/universal-ctags/ctags.git
2. 下载完成后，进入ctags文件夹，依次执行以下命令，完成编译和安装：
   ./autogen.sh
   ./configure
   make
   sudo make install
3. 使用which ctags查看安装路径，一会部署的时候要用到,我的路径是
   /usr/local/bin/ctags
```

### 7.解压opengrok
这里需要注意下，需要一个磁盘空间比较大的地方,因为一会生成索引的时候会在opengrok的目录里面生成.

### 8.在opengrok目录下建立目录
```java
mkdir src data etc
# src：源代码目录
# data：存放opengrok索引文件目录
# etc：放置configuration.xml的目录,xml文件由opengrok生成，我们只需要配置路径。
```

### 9.指定源码位置
原本需要把源码放到src下，因为我的android源代码放在了~/sourcecode19.02下，没必要把它整个移动到src下，所以在src下建了个sourcecode19.02的软连接指向它：
```java
cd src
ln -s ~/sourcecode19.02 sourcecode19.02
```

### 10.复制source.war到tomcat中
像这样
```java
cp ~/opengrok-1.2.2/lib/source.war /opt/tomcat8.5/webapps/
```

### 11.配置CONFIGURATION：
```java
在web.xml中修改configuration.xml的路径，  
vi ~/tomcat9.0/webapps/source/WEB-INF/web.xml  
<param-value>~/opengrok-1.3.1/etc/configuration.xml</param-value>
```

### 12.启动Tomcat
应该会看到如下的错误  
**opengrok configuration parameter has not been configured in web.xml**
这是因为我们没有生成索引,接下来我们来生成索引

### 13.执行如下命令
我是把下面的命令封装成了一个.sh文件，执行的时候尽量用sudo sh xxxx.sh执行，
```java
#!/bin/bash
java -Xmx8g \
-jar /home/pxwen/WorkStation/opengrok/opengrok-1.3.1/lib/opengrok.jar \
-c /usr/local/bin/ctags \
-s /home/pxwen/WorkStation/opengrok/opengrok-1.3.1/src -d /home/pxwen/WorkStation/opengrok/opengrok-1.3.1/data -H -P -S -G -v \
-W /home/pxwen/WorkStation/opengrok/opengrok-1.3.1/etc/configuration.xml -U http://localhost:8080/source \
--depth 100000 \
--progress \
-m 2048
```

### 14.打开http://localhost:8080/source/

![avatar](https://github.com/skypx/BlogResource/raw/master/other/opengrok.png)


### 15.多项目支持
同样的办法在src下建立软链接,然后执行上面生成索引的脚本. 等生成完了之后就可以看到了.   
删除项目的话，把刚才建立的软链接删掉，然后在重新生成索引.在浏览器看不到项目了.




### 遇到的问题
部署完成之后，我满心欢喜的打开浏览器，浏览着代码，舒服.就在我打开某个文件的时候突然   
![avatar](https://github.com/skypx/BlogResource/raw/master/other/opengrokerror.png)

心里一惊，不可能啊。难道没有部署成功,如果没有部署成功的话，那别的文件我也访问不了呀.  
那就是这个文件和别的文件不一样.鬼使神差的在目录下 **ls -l**  结果如下图.   
![avatar](https://github.com/skypx/BlogResource/raw/master/other/opengrokandroidbp.png)

原来这个文件是软链接. 难道所有软链接都不可用？不会呀，opengrok team肯定会考虑这个问题的   
查看文档.cd opengrok-1.3.1/lib$/   
**java -jar opengrok.jar**   
```java
--symlink /path/to/symlink
Allow this symlink to be followed. Option may be repeated.
By default only symlinks directly under source root directory
are allowed.
```
专门解决软链接的问题.

按照要求在刚才那个shell脚本里面加入这个两行
```java
--symlink /home/pxwen/WorkStation/opengrok/opengrok-1.3.1/src/sourcecode19.02/bootstrap.bash \
--symlink /home/pxwen/WorkStation/opengrok/opengrok-1.3.1/src/sourcecode19.02/Android.bp \
```

重新执行脚本生成index，完美.这两个文件可以访问了. 这里记住我说的是文件.还有文件夹呢.
文件夹里面的文件也不可以访问   
![avatar](https://github.com/skypx/BlogResource/raw/master/other/dirlink.png)

那是不是我把目录加如脚本里面也可以
```java
--symlink /home/pxwen/WorkStation/opengrok/opengrok-1.3.1/src/sourcecode19.02/build/tools/ \
```

重新执行脚本生成index, 结果文件夹里面的文件还是不可以访问。google了所有资料。没有找到解决办法.无奈，给opengrok team提交bug吧。让人家帮我指导下看是不是我哪里执行错了.

<https://github.com/oracle/opengrok/issues/2913>

结果,,,好像是opengrok的bug，应该是最新版本上的，他们已经复现了这个问题正在debug.


## 思考::::
我是用最用的tomcat和最新的opengrok部署的.不确定是否和这个有关系. 别人部署遇到这个问题了吗?   
还有我的android源码里面有好多软链接的形式. 不可能需要一行行的在脚本里面加。是否有其它的快捷方   
式。可以把我的项目里面的所有软链接都识别? 我们期待opengrok team 能尽快给我们反馈.
