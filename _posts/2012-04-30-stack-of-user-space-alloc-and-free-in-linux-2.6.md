---
layout: post
title:  "Linux2.6用户空间堆栈区的分配与回收"
category: [操作系统]
tags: [linux2.6]
---

#### 1.sys_brk(新边界的线性地址)
（1）地址检查，地址不低于代码段的终点  
（2）与页面大小对齐  
（3）新地址 < 老边界 -----> 释放空间（见2）  
  新地址 > 老边界 -----> 申请空间（见8）  

#### 2.释放空间
（1）线性地址 -> 区间地址  
（2）预备一个新的区间结构（回收一个区间的一部分，可能导致一个区间变成两个区间）  
（3）把所有涉及到的区间移到一个临时队列  
（4）解除映射，释放页面（见3）  
（5）对vm_area_struct和mm_struct作出调整  
（6）释放页面表  

#### 3.依次处理PGD中所有的页目录项所指向的页目录表，处理方法（见4）

#### 4.依次处理页目录表中的所有页面项所指向的页面，处理方法（见5）

#### 5.释放某一页面
（1）将页面的页面表项清0  
（2）解除内存页面及盘上页面的使用（见6）  

#### 6.解除内存页面及盘上页面的使用
（1）若页面在内存中且为脏，则移入脏队列  
（2）若页面在内存中，则解除对内存页面的使用（见7）  
（3）解除对“交换设备”盘上页面的使用  

#### 7.解除对内存页面的使用
（1）将页面从换入/掏出队列中脱离  
（2）将页面从某个LRU队列中脱离  
（3）将队列从杂凑队列中脱离  
（4）释放页面  

#### 8.申请空间
（1）权限检查，越界检查，冲突检查，各种检查  
（2）申请一个新的区间，如果能合并，就合并  
（3）为新区间建立起对内存的映射  