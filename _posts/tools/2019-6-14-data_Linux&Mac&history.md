---
layout: post
title: Linux 和 Mac下快速搜索历史命令
category: 工具
tags: Linux 与 Mac 搜索历史命令
---
* content
{:toc}

> 在Linux 或者 Mac系统中如何快速搜索已经使用过的命令?

* 在Linux中我们可以用history查询已经使用过的命令

* 并且还可以用 Ctrl + R 来搜索已经用过的命令

* 我们用一种更快的方式来搜索, 你忘记了以前输入过的某个命令, 只记得开头的几个字母, 没关系
   下面我们看看怎么快速补全

1. 进入你的用户目录下.建立.inputrc文件
2. 输入以下内容

    ```java
    "\e[A": history-search-backward
    "\e[B": history-search-forward
    set show-all-if-ambiguous on
    set completion-ignore-case on
    ```
3. 保存, 执行 bind -f .inputrc

然后你随便输入几个命令. ls, pwd, cd, ....
这个时候想输入pwd, 只用输入p 然后按键盘上键, 命令自动补充完毕. 大大提升工作效率!!
