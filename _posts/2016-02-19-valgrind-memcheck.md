---
layout: post
title:  "valgrind memcheck使用方法及效果"
category: linux
tags: [valgrind, memcheck]
---

#### 一、valgrind

##### 1. Valgrind是什么

Valgrind是运行在Linux上一套基于仿真技术的程序调试和分析工具，它包含一个内核──一个软件合成的CPU，和一系列的小工具，每个工具都可以完成一项任务──调试，分析，或测试等。Valgrind可以检测内存泄漏和内存违例，还可以分析cache的使用等

不管是使用哪个工具，valgrind在开始之前总会先取得对你的程序的控制权，从可执行关联库里读取调试信息。然后在valgrind核心提供的虚拟CPU上运行程序，valgrind会根据选择的工具来处理代码，该工具会向代码中加入检测代码，并把这些代码作为最终代码返回给valgrind核心，最后valgrind核心运行这些代码。

valgrind是高度模块化的，所以开发人员或者用户可以给它添加新的工具而不会损坏己有的结构。

<!-- more -->

##### 2. valgrind tool是什么

valgrind提供多种内存检测方法，用于检测不同的数据，满足不同的使用需求

可使用的工具如下：

（1）cachegrind是一个缓冲模拟器。它可以用来标出你的程序每一行执行的指令数和导致的缓冲不命中数。

（2）callgrind在cachegrind基础上添加调用追踪。它可以用来得到调用的次数以及每次函数调用的开销。作为对cachegrind的补充，callgrind可以分别标注各个线程，以及程序反汇编输出的每条指令的执行次数以及缓存未命中数。

（3）helgrind能够发现程序中潜在的条件竞争。

（4）lackey是一个示例程序，以其为模版可以创建你自己的工具。在程序结束后，它打印出一些基本的关于程序执行统计数据。

（5）massif是一个堆剖析器，它测量你的程序使用了多少堆内存。

（6）memcheck是一个细粒度的的内存检查器。

（7）none没有任何功能。它一般用于Valgrind的调试和基准测试。

##### 3. Valgrind怎么用

###### (1)安装

```
yum install valgrind
```

###### (2)运行

```
valgrind --tool=toolname program args
```
例如

```
valgrind --tool=memcheck ls -l
```

`--tool`选项，用于选择valgrind tool中的一种，后面接tool的名字。可以不加这个参考，则默认使用memcheck。

program选项，用于指定检测程序对象。valgrind对目标program的编译过程有些要求：

（1）打开调试模式（gcc编译器的-g选项）。如果没有调试信息，即使最好的valgrind工具也将中能够猜测特定的代码是属于哪一个函数。打开调试选项进行编译后再用valgrind检查，valgrind将会给出具体到某一行的详细报告。

（2）关闭编译优化选项(比如-O2或者更高的优化选项)。这些优化选项可能会使得memcheck提交错误的未初始化报告，因此，为了使得valgrind的报告更精确，在编译的时候最好不要使用优化选项。


#### memcheck

##### 1.valgrind memcheck是什么

memcheck是valgrind tool的一种，是一个细粒度的的内存检查器。它可以检测以下问题：
1）使用未初始化的内存

2）读/写已经被释放的内存

3）读/写内存越界

4）读/写不恰当的内存栈空间

5）内存泄漏

6）使用malloc/new/new[]和free/delete/delete[]不匹配。

7）src和dst的重叠

##### 2.使用未初始化的内存

###### 测试代码

在这里，我列举了四种使用未初始化内存的情况

```c++
#include <iostream>
using namespace std;

#include "string.h"

void test1()
{
    int a;
    cout<<a+3<<endl;
}

void test2()
{
    char *p1 = new char[50];
    char *p2 = new char[50];
    memcpy(p1, p2, 50);
    delete []p1;
    delete []p2;
}

void test3()
{
    char *p1 = new char[50];
    char *p2 = new char[50];
    memcpy(p1, p2, 50);
    cout<<p1[3]<<endl;
    delete []p1;
    delete []p2;
}

void test4()
{
    char p1[50];
    char p2[50];
    memcpy(p1, p2, 50);
    cout<<p1[3]<<endl;
}

int main()
{
    test1();
    test2();
    test3();
    test4();
    return 0;
}
```

###### 编译及运行

```
g++ -g -o uninit val-uninit.cpp
valgrind --track-origins=yes /home/vagrant/git_hub/windmissing.github.io/_posts/code/uninit
```

`--track-origins=yes`表示开启“使用未初始化的内存”的检测功能，并打开详细结果。如果没有这句话，默认也会做这方面的检测，但不会打印详细结果。

###### 检测结果

结果有点长，这里只是节选部分重点信息

```
==11190== Memcheck, a memory error detector
```
11190是valgrind的进程号，也是目标程序的进程号。它们是同一个进程。

```
==11190== Conditional jump or move depends on uninitialised value(s)

...

==11190==    by 0x4009C9: test1() (val-uninit.cpp:9)
==11190==    by 0x400B32: main (val-uninit.cpp:41)
```
这是检测到的第一处使用未初始化内存的地方，会打印完整的堆栈信息

```
==11190== Syscall param write(buf) points to uninitialised byte(s)

...

==11190==    by 0x400A9F: test3() (val-uninit.cpp:26)
==11190==    by 0x400B3C: main (val-uninit.cpp:43)
```
test1和teset3都被检测出来了，test2和test4没有


###### 检测结果解读

(1)未初始化的栈空间和堆空间都可以被检测出来(test1/test3)

(2)如果只是申请了但没有使用，不会被检测(test2/test3)

(3)memcpy(p1, p2, 50);不会检测p2的空间是否初始化，但这样执行后p1仍然是未初始化的(test2)

(4)栈中的数组是默认初始化的

##### 3.读/写已经被释放的内存

###### 测试代码

```c++
#include <iostream>
using namespace std;

int main()
{
    int *p = new int;
    delete p;

    *p = 3;
    return 0;
}
```

###### 编译及运行

```
g++ -g -o deleted val-deleted.cpp
valgrind /home/vagrant/git_hub/windmissing.github.io/_posts/code/deleted 
```

###### 检测结果

```
==11588== Invalid write of size 4
==11588==    at 0x400796: main (val-deleted.cpp:9)
==11588==  Address 0x5a15040 is 0 bytes inside a block of size 4 free'd
==11588==    at 0x4C2B131: operator delete(void*) (in /usr/lib64/valgrind/vgpreload_memcheck-amd64-linux.so)
==11588==    by 0x400791: main (val-deleted.cpp:7)
```
###### 检测结果解读

（1）“读/写已经被释放的内存”可以被检测出来

（2）打印的信息只会提示“释放内存”的代码，不会提示“使用释放的内存”的代码

##### 4.读/写内存越界

###### 测试代码

这里设计了三种访问越界的场景，分别测试访问栈内数组越界、访问堆中数组越界、函数传参导致数组长度退化三种情况

```c++
#include <iostream>
using namespace std;
#include "string.h"

void test1()
{
    int s[5] = {1, 2, 3, 4, 5};
    cout<<s[5]<<endl;
}

void test2()
{
    int *s = new int[5];
    memset(s, 0, sizeof(s));
    cout<<s[5]<<endl;
    delete []s;
}

void print(int *s, int id)
{
    cout<<s[id]<<endl;
}

void test3()
{
    int s[5] = {1, 2, 3, 4, 5};
    print(s, 5);
}
int main()
{
    test1();
    test2();
    test3();
    return 0;
}
```

###### 编译及运行

```
g++ -o outrange val-outrange.cpp 
valgrind /home/vagrant/git_hub/windmissing.github.io/_posts/code/outrange
```
###### 检测结果

关于test1

```
==24777== Use of uninitialised value of size 8

...

==24777==    by 0x400959: test1() (in /home/vagrant/git_hub/windmissing.github.io/_posts/code/outrange)
==24777==    by 0x400A53: main (in /home/vagrant/git_hub/windmissing.github.io/_posts/code/outrange)
```

关于test2

```
==24777== Invalid read of size 4
==24777==    at 0x40099D: test2() (in /home/vagrant/git_hub/windmissing.github.io/_posts/code/outrange)
==24777==    by 0x400A58: main (in /home/vagrant/git_hub/windmissing.github.io/_posts/code/outrange)
==24777==  Address 0x5a15054 is 0 bytes after a block of size 20 alloc'd
==24777==    at 0x4C2A7AA: operator new[](unsigned long) (in /usr/lib64/valgrind/vgpreload_memcheck-amd64-linux.so)
==24777==    by 0x40097A: test2() (in /home/vagrant/git_hub/windmissing.github.io/_posts/code/outrange)
==24777==    by 0x400A58: main (in /home/vagrant/git_hub/windmissing.github.io/_posts/code/outrange)
==24777==
```

关于test3

```
==24777== Conditional jump or move depends on uninitialised value(s)

...

==24777==    by 0x400A48: test3() (in /home/vagrant/git_hub/windmissing.github.io/_posts/code/outrange)
==24777==    by 0x400A5D: main (in /home/vagrant/git_hub/windmissing.github.io/_posts/code/outrange)
```

###### 检测结果解读

1.可以检测访问堆中数组越界的问题

2.不能检测访问栈中数组越界的问题

3.堆中数组因为参数传递而退化为指针后，不能检测出访问越界的问题

##### 5.内存泄漏

###### 测试代码

```c++
#include <iostream>
using namespace std;

void test1()
{
    int *p = new int;
}

void test2()
{
    int *p = new int;
    char *p2 = (char *)p;
    delete p2;
}

class father
{
    int *p;
public:
    father(){p = new int;}
    ~father(){delete p;}
};

class son : public father
{
    int *p2;
public:
    son(){p2 = new int;}
    ~son(){delete p2;}
};

void test3()
{
    father *p = new son;
    delete p;
};
int main()
{
    test1();
    test2();
    test3();
    return 0;
}
```

###### 编译及运行

```
g++ -g -o memleak val-memleak.cpp
valgrind --leak-check=full /home/vagrant/git_hub/windmissing.github.io/_posts/code/memleak
```
###### 检测结果

```
==29099== HEAP SUMMARY:
==29099==     in use at exit: 8 bytes in 2 blocks
==29099==   total heap usage: 5 allocs, 3 frees, 32 bytes allocated
==29099== 
==29099== 4 bytes in 1 blocks are definitely lost in loss record 1 of 2
==29099==    at 0x4C2A105: operator new(unsigned long) (in /usr/lib64/valgrind/vgpreload_memcheck-amd64-linux.so)
==29099==    by 0x400871: test1() (val-memleak.cpp:6)
==29099==    by 0x40090A: main (val-memleak.cpp:39)
==29099== 
==29099== 4 bytes in 1 blocks are definitely lost in loss record 2 of 2
==29099==    at 0x4C2A105: operator new(unsigned long) (in /usr/lib64/valgrind/vgpreload_memcheck-amd64-linux.so)
==29099==    by 0x4009CE: son::son() (val-memleak.cpp:28)
==29099==    by 0x4008C3: test3() (val-memleak.cpp:34)
==29099==    by 0x400914: main (val-memleak.cpp:41)
==29099== 
==29099== LEAK SUMMARY:
==29099==    definitely lost: 8 bytes in 2 blocks
==29099==    indirectly lost: 0 bytes in 0 blocks
==29099==      possibly lost: 0 bytes in 0 blocks
==29099==    still reachable: 0 bytes in 0 blocks
==29099==         suppressed: 0 bytes in 0 blocks
==29099== 
```

###### 检测结果解读

1.内存泄漏可以被检测出来

2.检测结果会给出详细的堆栈信息及行号

##### 6.使用malloc/new/new[]和free/delete/delete[]不匹配

###### 测试代码

```c++
#include <iostream>
using namespace std;

void test1()
{
    int *p = new int[5];
    delete p;
}

int main()
{
    test1();
}
```

###### 编译及运行

```
g++ -g -o mismatch val-mismatch.cpp 
valgrind --leak-check=full /home/vagrant/git_hub/windmissing.github.io/_posts/code/mismatch
```
###### 检测结果

```
==29260== Mismatched free() / delete / delete []
==29260==    at 0x4C2B131: operator delete(void*) (in /usr/lib64/valgrind/vgpreload_memcheck-amd64-linux.so)
==29260==    by 0x400791: test1() (val-mismatch.cpp:7)
==29260==    by 0x40079C: main (val-mismatch.cpp:12)
==29260==  Address 0x5a15040 is 0 bytes inside a block of size 20 alloc'd
==29260==    at 0x4C2A7AA: operator new[](unsigned long) (in /usr/lib64/valgrind/vgpreload_memcheck-amd64-linux.so)
==29260==    by 0x400781: test1() (val-mismatch.cpp:6)
==29260==    by 0x40079C: main (val-mismatch.cpp:12)
```
###### 检测结果解读

##### 7.src和dst的重叠

###### 测试代码

```c++
#include <iostream>
using namespace std;

#include "string.h"

void test1()
{
    char ch[10] = "abcdefghi";
    char *p1 = ch;
    char *p2 = ch + 3;
    memcmp(p1, p2, 5);
}

int main()
{
    test1();
    return 0;
}
```

###### 编译及运行

```
g++ -g -o overlap val-overlap.cpp
valgrind --leak-check=full /home/vagrant/git_hub/windmissing.github.io/_posts/code/overlap
```
###### 检测结果

```
==29405== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 2 from 2)
```

###### 检测结果解读

对于不涉及到写的情况，src和dst重叠不算是问题

#### 四、Valgrind的特点

##### 1.优点
（1）检测对象程序在编译时无须指定特别的选项，也不需要连接特别的函数库

（2）valgrind被设计成非侵入式的，它直接工作于可执行文件上，因此在检查前不需要重新编译、连接和修改你的程序。要检查一个程序很简单，只需要执行下面的命令就可以了

（3）valgrind模拟程序中的每一条指令执行，因此，检查工具和剖析工具不仅仅是对你的应用程序，还有对共享库，GNU C库，X的客户端库都起作用。

（4）能打印堆栈信息，具体到某一行

（5）合并重复的信息
 
##### 2.缺点
（1）不同工具间加入的代码变化非常的大。在每个作用域的末尾，memcheck加入代码检查每一片内存的访问和进行值计算，代码大小至少增加12倍，运行速度要比平时慢25到50倍。 

（2）程序运行结束后才会显示结果