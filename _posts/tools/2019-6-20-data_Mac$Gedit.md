---
layout: post
title: Mac 下安装gedit
category: 工具
tags: Mac Gedit
---
* content
{:toc}

>Mac 下安装 Gedit

## 1.可通过 <https://gedit.en.softonic.com/mac> 安装

  1. open -e .bash_profile
  2. alias gedit="sudo open -a gedit"
  3. source .bash_profile
  4. 如果出现打不开“gedit”，因为它来自身份不明的开发者。的问题：参考

     <https://jingyan.baidu.com/article/f71d60377960651ab741d140.html>

关于open 命令

* open，使用关联的程序打开文件，例：open a.txt会用文本编辑打开a.txt，open b.jpg会使用预览打开b.jpg
* open -e，强制使用文本编辑程序打开文件
* open -a，自行选择程序打开文件，例：open -a Preview b.jpg会使用预览打开b.jpg，另外使用此命令输入已安装的程序名可直接打开，而open则需要知道程序存放的路径才行，例：open -a Preview等同于open /Applications/Preview.app

但是我用这种方式安装之后打开的一次之后，在次打开的话一定要完全退出上次的软件， 所以很不舒服，下面介绍第二种方式安装

## 2.通过下面的连接安装

<http://macappstore.org/gedit/>

我并没有用**brew cask install gedit**这个命令安装
用**brew install gedit**这个命令，会安装很多依赖，大约5分钟左右

然后就可以gedit文件了，这个界面比通过[softonic](<https://gedit.en.softonic.com/mac>)这种方式的安装界面漂亮多了
