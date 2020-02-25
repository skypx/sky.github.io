---
layout: post
title:  mk 文件里面的重复copy问题
category: Android
tags: mk 文件
---
* content
{:toc}

##### 问题

> 我们都知道在mk文件里面做copy操作可以通过 **PRODUCT_COPY_FILES** 这个变量来做copy操作  
由于项目需要我需要对同一个文件，名字相同，内容不同的文件进行copy,类似这样

```java
a.mk -> copy a.txt  
b.mk -> copy a.txt  

c.mk  
   -- include a.mk  
   -- include b.mk  
```

我想当然的以为肯定在b.mk里面文件会覆盖掉a.mk文件里面的东西, 因为mk文件执行是树形结构从上到下  
执行，本以为b.mk文件里面的会覆盖掉a.mk, 编译，刷机, 验证结果，目瞪狗呆!为什么是a.mk文件里面  
的文件会覆盖掉b.mk文件


##### 思考
1. 脑子抽疯, 莫非mk从下向上编译，验证结果，在mk里面加log，明显是从上到下的执行.  
2. 莫非针对这个copy有去重的定义？这个想法靠谱，我们平时写代码的时候自己也会有去除重复的习惯.  

##### 找代码
```java
这个目录下 /build/core/Makefile
```
```java
# filter out the duplicate <source file>:<dest file> pairs.
unique_product_copy_files_pairs :=
$(foreach cf,$(PRODUCT_COPY_FILES), \
    $(if $(filter $(unique_product_copy_files_pairs),$(cf)),,\
        $(eval unique_product_copy_files_pairs += $(cf))))
```

```java
unique_product_copy_files_destinations :=
$(foreach cf,$(unique_product_copy_files_pairs), \
    $(eval _src := $(call word-colon,1,$(cf))) \
    $(eval _dest := $(call word-colon,2,$(cf))) \
    $(call check-product-copy-files,$(cf)) \
    $(if $(filter $(unique_product_copy_files_destinations),$(_dest)), \
        $(info PRODUCT_COPY_FILES $(cf) ignored.), \
        $(eval _fulldest := $(call append-path,$(PRODUCT_OUT),$(_dest))) \
        $(if $(filter %.xml,$(_dest)),\
            $(eval $(call copy-xml-file-checked,$(_src),$(_fulldest))),\
            $(eval $(call copy-one-file,$(_src),$(_fulldest)))) \
        $(eval ALL_DEFAULT_INSTALLED_MODULES += $(_fulldest)) \
        $(eval unique_product_copy_files_destinations += $(_dest))))
```

从去除重复的算法来看，是从第一个字符串开始，如果目标中没有就添加，如果已经有就不做任何处理，因此只有最先描述的目标有效  

至此明白了为什么，只有第一个copy的有效. 因为如果检测到已经有了这个文件的话，就不会对第二个文件做处理了.

PS: 如何在mk里面加入log,
```java
输出信息的方式为::
$(warning xxxxx) 或者 $(error xxxxx)

输出变量的方式为::
$(warning  $(XXX))

如果是输出error的话::
如果是$(error xxxxx)将会停止编译
```
