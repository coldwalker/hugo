---
title: "一个StackOverFlowError问题引出的JVM栈内存管理"
date: 2018-03-21T16:51:44+08:00
categories: ["技术"]
tags: ["问题排查","Java","JVM"]
thumbnail: "images/yalaxueshan.jpg"
draft: false
---

最近项目中在使用javacv时在加载jar中的so共享链接库时出现了StackOverFlowError，最终原因是由于jvm启动时指定的最大栈内存Xss的大小不够，从而导致线程栈溢出。<!--more-->
从-Xss256k调整为-Xss1m后问题解决。
*注意： JDK1.5开始，xss 64位默认1M, 32为默认512K。*


## 科普一下JVM栈内存的相关知识：
JVM的栈包括两部分，虚拟机栈和本地方法栈，各JVM的最终实现不一样，例如HotSpot是将两个栈合二为一，不区分。

### *Java虚拟机栈（Java Virtual Machine Stacks）*

线程私有的，生命周期与线程相同。每个方法在执行的同时，会创建一个栈帧（Stack Frame）。方法开始执行时，压入这个栈帧，方法执行完成，这个栈帧就出栈。递归调用方法如果递归的深度过深，就会出现栈溢出（StackOverflowError）异常，值得就是这个地方的栈溢出。如果这个区域允许动态扩展，但是无法申请到足够的内存，就会内存溢出（OutOfMemoryError）。
设置参数：
-Xss 调整栈内存容量

### *本地方法栈（Native Method Stack）*

线程私有。与上面提到的Java虚拟机栈（Java Virtual Machine Stacks）相对应，Java虚拟机栈为Java方法服务，本地方法栈为Native方法服务。Sun HotSpot将这两个栈合二为一。栈溢出和内存溢出规则也一样。
设置参数：
-Xoss 调整栈内存容量（Hot spot虚拟机无效）
虚拟机栈和本地方法栈的-Xss（-Xoss）参数影响了栈的深度，当抛出StackOverflowError异常说明-Xss参数设置过小。不停递归调用会抛出此异常。定义大量本地变量增加方法帧本地变量表的长度也会抛出此异常。

### 栈里面放啥？
栈中进行压栈出栈的叫栈帧，栈帧由三部分组成：局部变量区、操作数栈和帧数据区(保存一些数据来支持常量池解析、正常方法返回以及异常派发机制)。

####  局部变量区：
局部变量区是一组变量值存储空间，用于存放方法参数和方法内部定义的局部变量。在Java程序被编译为Class文件时，就在方法的Code属性的max\_locals数据项中确定了方法所需要分配的最大局部变量表的容量。局部变量表的容量以变量槽（Variable Slot）为最小单位（字节数组组成）。
JVM规范规定：
每个Slot都应该能存放一个boolean,byte,char,short,int,float,refrence,returnAddress类型的数据，对于非基本类型的局部变量，在局部变量区只存储引用，具体对应的值还是在堆中，通过reference表示。returnAddress类型是为字节码指令jsr、jsr\_w和ret服务的，它指向了一条字节码指令的地址。大部分资料表示Slot的长度为32位，不排除在64位系统中Slot的长度为64位（未考证），对于64位的数据类型，如long、double，虚拟机会以高位在前的方式为其分配两个连续的Slot空间。由于局部变量表建在线程的堆栈上，是线程私有的数据，无论读写两个连续的Slot是否是原子操作，都不会引起数据安全问题。虚拟机通过索引定位的方式使用局部变量表，索引值的范围是从0开始到局部变量表最大的Slot数量。如果32位数据类型的变量，索引N就代表了使用第N个Slot，如果是64位数据类型的变量，则说明要使用第N个和N+1两个Slot。对于非static方法，局部变量的第1个Slot默认为方法所属对象实例的引用，通过this访问。

#### 操作数栈：
操作数栈用于存放方法中计算时的中间结果，也是被组织成以字长为单位的数组。但不是通过数组下标来访问，而是通过压栈出栈来访问，操作数栈中存储数据的方式和在局部变量区中是一样的：如int、long、float、double、reference和returnType的存储。对于byte、short以及char类型的值在压入到操作数栈之前，也会被转换为int（原因是由于Java虚拟机缺乏对byte、short和char类型的，某些指令只支持int类型）。

#### 帧数据区：
除了局部变量区和操作数栈外，Java栈帧还需要一些数据来支持常量池解析、正常方法返回以及异常派发机制。这些数据都保存在Java栈帧的帧数据区中。

* 常量池解析存的是一个引用，这个引用指向栈帧当前运行方法所在类的常量池，通过这个引用支持动态链接。

* 除了处理常量池解析外，帧里的数据还要处理Java方法的正常结束和异常终止。如果一个方法有定义 try-catch 或者 try-finally 异常处理器，那么就会创建一个异常表，每个异常表入口包含四个信息：
	![stack_data_area](/images/stack_data_area.png)￼


	1. JVM 在 try 住的代码区间内如有异常抛出的话，就会在当前栈桢的异常表中，找到匹配类型的异常记录的入口指令号，然后跳到该指令处执行。异常指令块执行完后，再回来继续执行后面的代码。JVM 按照每个入口在表中出现的顺序进行检索，如果没有发现匹配的项，JVM 将当前栈帧从栈中弹出，再次抛出同样的异常。
	2. 当 JVM 弹出当前栈帧时，JVM 马上终止当前方法的执行，并且返回到调用本方法的方法中，但是并非继续正常执行该方法，而是在该方法中抛出同样的异常，这就使得 JVM 在该方法中再次执行同样的搜寻异常表的操作。
	3. 如果所有的栈帧都被弹出还没有找到匹配的异常处理器，那么这个线程就会终止。如果这个异常在最后一个非守护进程抛出（比如这个线程是主线程），那么也有会导致 JVM 进程终止。

* 方法出口：如果是通过return正常结束，则当前栈帧从Java栈中弹出，恢复发起调用的方法的栈。如果方法有返回值，JVM会把返回值压入到发起调用方法的操作数栈。

### 产生StackOverFlowError的原因有哪些？
1. 死循环本身是不会StackOverflow的，比较常见的比如无限递归。原则上循环嵌套次数本身是没有限制的，限制的是占用的栈空间，由于局部变量、操作数栈、帧数据区等都在栈内存中，因此也可能导致栈内存溢出。
2. 依赖JNI的本地方法依赖较多，导致需要load的本地代码空间要求更多，但StackShadowPages设置过小或Stack Space总的大小不够。

Say:
<font color=gray>A stack overflow in Java language code will normally result in the offending thread throwing java.lang.StackOverflowError. On the other hand, C and C++ write past the end of the stack and provoke a stack overflow. This is a fatal error which causes the process to terminate.

In the HotSpot implementation, Java methods share stack frames with C/C++ native code, namely user native code and the virtual machine itself. Java methods generate code that checks that stack space is available a fixed distance towards the end of the stack so that the native code can be called without exceeding the stack space. This distance towards the end of the stack is called “Shadow Pages.” The size of the shadow pages is between 3 and 20 pages, depending on the platform. This distance is tunable, so that applications with native code needing more than the default distance can increase the shadow page size. The option to increase shadow pages is -XX:StackShadowPages= n, where n is greater than the default stack shadow pages for the platform.</font>

******
Resource：
http://www.importnew.com/17770.html
https://github.com/bytedeco/javacpp-presets/wiki/Debugging-UnsatisfiedLinkError-and-StackOverflowError-on-Linux

