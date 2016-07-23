---
layout: post
title:  "Linux2.6系统调用"
category: [操作系统]
tags: [linux2.6]
---

#### 一、引入系统调用
##### 1.概念：
操作系统为在用户态运行的进程与硬件设备进行交到提供了一组接口。Linux通过向内核发出系统调用来实现这些接口
##### 2.作用：
对硬件设备操作的编程更容易  
提高了系统的安全性  
使程序更有可移植性  
##### 3.进入系统调用的两种方法
（1）int &0x80汇编指令  
（2）sysenter汇编指令  

#### 二、系统调用与API的区别
1.API只是一个函数定义，说明了如何获得一个给定的服务  
系统调用中通过软中断向内核发出一个明确的请求  
2.API可以不使用系统调用，也可以使用多个系统调用  
多个API也可以调用封装了不同功能的同一系统调用  
3.系统调用属于同内核  
用户态的库函数不属于内核  

#### 三、通过int $0x80汇编指令进入系统调用
1.切换到内核态  
2.把系统调用号和所需要寄存器（**不包括EFLAGS.CS.SS.EIP.ESP**）存入栈中  
3.在DS和ES中装入内核数据段和段选择符  
4.把thread_info的地址装入EBX  
5.调用与EAX中的所包含的系统调用号相对应的服务程序  
6.**服务程序结束**  
7.从EAX获得服务程序的返回值  
8.恢复保存在内核中的寄存器值  
9.重新开始执行用户态进程  

#### 四、通过sysenter汇编指令进入系统调用  
1.把系统调用号装入EAX  
2.把EBP.EDX.ECX保存到用户堆栈  
3.把用户堆栈的指令放入EBP  
4.把sysenter使用的三个特殊的寄存器的内容，分别装入CS.EIP.SS.ESP中  
5.CPU从用户态切换到内核态  
6.建立内核堆栈指针  
7.把用户数据段的段选择符、当前用户指针、EFLAGS、用户代码段的段选择符、退出系统调用时要执行的指令的地址，依次保存到内核堆栈中  
8.同“三的2-5步”  
9.**服务程序结束**  
10.获取服务程序的返回码  
11.从内核堆栈中取出第7步保存的内容  
12.切换到用户态sysexit  

#### 五、system_call()和sysenter_entry()是Linux中所有系统调用的入口点

#### 六、参数的传递
1.普通C函数的传递：把参数值写入活动的程序栈  
2.系统调用的参数传递：在系统调用前，参数被写入CPU寄存器，在调用系统调用服务程序前，把参数拷贝到内核堆栈中  
3.系统调用是一种横跨用户和内核两大陆地的特殊函数，所以既不能使用用户态栈，也不能使用内核态栈  
4.服务程序的返回值写入EAX中  