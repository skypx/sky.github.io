---
layout: post
title: Linux 与 Mac下的Terminal颜色修改
category: 工具
tags: Terminal
---
* content
{:toc}


* 默认的PS1 配置

代码块
英文状态下直接按左上角就可以，一定是英文状态
中文状态下shit+option+左上角，
```java
echo $PS1
\h:\W \u\$
```
![avatar](https://github.com/skypx/BlogResource/raw/master/other/normalterminal.png)

* 修改PS1的配置文件

1.  Linux系统中修改.bashrc, 添加下面的代码
```java
PS1='${debian_chroot:+($debian_chroot)}\[\e]2;\w\a\]\[\033[01;35m\]\u\[\033[01;37m\]@\[\033[01;34m\]\h \[\033[00m\]: \[\033[01;33m\]\w \[\e[00m\]\[\e[01;36m\][\t \d] \[\e[01;32m\]No.\#(\!)\n\[\e[01;31m\]\$ \[\033[01;37m\]'
```
2. Mac中修改.bash_profile. 同样添加上面的代码
下面的这是是我的配置，我只是把上面的代码颜色改了
```java
export PS1='${debian_chroot:+($debian_chroot)}\[\e]2;\w\a\]\[\033[01;35m\]\u\[\033[01;35m\]@\[\033[01;35m\]\h \[\033[01;35m\]: \[\e[01;36m\]\w \[\e[00m\]\[\e[01;36m\][\t \d] \[\e[01;32m\]No.\#(\!)\n\[\e[01;31m\]\$ \[\033[01;37m\]'
```
![avatar](https://github.com/skypx/BlogResource/raw/master/other/changeterminal.png)

这样就可以显示时间，和历史命令的是第几条的terminal了，是不是很酷
