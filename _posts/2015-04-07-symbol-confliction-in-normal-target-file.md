---
layout: post
title:  "普通目标文件的符号重定义处理策略"
category: [compile]
tags: [g++, linking, symbol]
---

#### 一、什么是普通目标文件


静态链接器ld可以将一组可重定位目标文件链接成一个可执行目标文件。
其中可重定位目标文件有三种，分别是目标文件（.o）、静态链接库（.a）和动态链接库（.so）。
本文所指的普通目标文件特殊“目标文件（.a）”

#### 二、什么是符号

符号是指代码中的变量与函数。  
[链接中的符号](http://windmissing.github.io/compile/2015-04/symbols-in-linking.html)

#### 三、强符号与弱符号

只有外部符号和可引出符号才有强符号和弱符号的概念。  
 - 强符号  
（1）函数
（2）已经初始化的全局变量  
 - 弱符号  
（1）未初始化的全局变量


#### 四、符号重定义的处理策略

链接器使用强弱符号规则来决定怎么处理符号重定义的问题。  
（1）至多有一个强符号存在，若出现多个强符号，链接器就会报错  
（2）如果有一个强符号和多个弱符号，那么选择强符号  
（3）如果有多个弱符号，会选择占用地址空间较大的那个弱符号，不报错  
由此可见，为了及时地发现重定义引入的问题，尽量把符号都定义成强符号。  
