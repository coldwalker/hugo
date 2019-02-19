---
title: "Java垃圾回收浅析-GC方式介绍"
date: 2019-02-19T17:00:00+08:00
categories: ["技术"]
tags: ["GC","Java"]
thumbnail: "images/gc_intro/changlonghaiyangwangguoyanhua.jpg"
draft: false
---
### 为什么需要GC？
当程序创建对象、数组等引用类型实体时，系统都会在堆内存中为之分配一块内存区，对象就保存在这块内存区中，当这块内存不再被任何引用变量引用时，这块内存就变成垃圾，等待垃圾回收机制进行回收。在C和C++中，垃圾的回收是由程序员来手动执行，虽然实时性比较好，但由于内存分配和回收代码繁琐，较容易出错，内存泄露问题比较常见。于是一些语言如Java、python、Go等，通过程序实现了自动化内存管理来解决垃圾回收问题，从而降低了在这些语言中内存使用的门槛。另外，由于GC有一定的滞后性，在一些实时性较高或者内存吃紧的（单片机开发）软件中，还是有不少采用人工回收垃圾的案例。

*GC的一些特点：*
1. GC只负责回收内存中的对象，不回收物理资源（如文件流、socket等）。
2. 程序无法精确控制GC的运行，GC只会在合适的时候进行。

<i>PS：1其实也是由于2的GC的不确定性导致的，比如一个socket占用了某个端口，当使用完后GC并不会立刻回收这个socket，当需要再用到这个端口的socket时，就会报错。</i>

### GC实现的两种常见方式
#### 引用计数（Reference Counting）
引用计数的实现比较简单，核心就是给每一个对象增加一个引用计数器，当另一个对象引用当前对象时就给当前对象的引用计数+1，当有对象不再引用当前对象时，就将引用计数-1，当前对象的引用计数变成0时，递归地将该对象引用的子对象的引用计数-1，并把当前对象使用的内存区域释放到空闲链表中。
![6efac91db597f7d968d53f68c99e32d4.gif](/images/gc_intro/refcount.gif)


*引用计数算法的优点：*

1. 引用归0时可立即回收垃圾，实时性好。
2. 应用执行和垃圾回收“自然、依次”发生，GC开销分布在各次应用执行中，最大暂停时间消减（虽然GC次数增加）。
3. 执行效率高，“事件触发”机制避免了整堆扫描标记的耗时。

*引用计数算法的缺点：*

1. 循环引用问题无法解决，导致垃圾回收不彻底。
2. 需要额外的存储空间来存放每一个对象的引用计数。
3. 频繁的计数增减带来的高并发和本身需要的操作原子性的开销较大。
4. 引用计数归0时，可能会级联删除大量对象，造成GC耗时不平稳。

<i>以上可知，基于引用计数的GC在实时性和执行效率方面有较大优势，类似 python、perl、swift等语言都使用引用计数算法来实现GC。但由于引用计数GC存在高并发时的性能较低等问题，因此，对性能要求较高的系统在实现时一般不会采用引用计数算法，而是使用和应用隔离的“跟踪回收”方式来实现GC。</i>

#### 跟踪回收
跟踪回收是一种独立于应用程序外的GC方式，定期运行来检查垃圾。跟踪回收通过被称为“GC ROOTS”的对象开始，不断通过深度或者广度遍历的方式来跟踪标记所有”可达的对象“，然后其他“GC ROOTS不可达对象”的未标记对象就被视为垃圾，会被统一回收。跟踪回收是目前被广泛使用的技术，如：Java、Go、.net等都使用跟踪的方式来进行垃圾回收。

*跟踪回收的GC方式在具体的算法实现上，主要包括以下几种垃圾回收算法：*
##### 标记-清除（mark-sweep)
标记清除算法主要分两个阶段：

1. 第一阶段，从GC ROOTS开始进行遍历，标记所有可直达或间接到达的对象为“使用中”。
2. 第二阶段，扫描整体内存，对上一阶段标记的“使用中”的对象进行回收。
![3ded3d0575a91f8dfde31dc446642d56.jpeg](/images/gc_intro/A73996FF-5E0A-42D5-AE6B-73D25A1FD000.jpg)

原始的标记-清除算法有不少问题，比如：

1. GC期间，整个业务线程都会被挂起，暂停时间较长。
2. 内存容易出现碎片，多次GC后可用连续内存问题明显。
3. GC的STW（Stop The World）的时间和heap大小正相关。

因此在标记-清除算法的基础上，又产生了很多基于标记-清除的衍生算法来优化这些问题。比如：
###### 并发的标记-清除算法
标记-清除过程实际上大部分时间Collector（GC线程）是可以和Mutator（应用程序线程）并发执行，大部分的工作都在并发阶段完成，真正需要暂停应用线程的阶段只需要来解决并发过程中变化的那部分对象即可，从而并不需要整个回收阶段都block住应用线程。比如CMS垃圾回收器除了Init-Mark阶段和Final-Mark阶段外，其他阶段都是可以并发执行的。
###### 标记--整理算法
针对内存容易出现碎片的问题，衍生的Mark-Compact算法通过将sweep过程替换成compact过程，每次GC都会对内存进行一次移动整理。

*标记-整理算法一般分3部分：*

a. 先在mark阶段标记存活对象。
b. 遍历heap计算出存活对象将要移动到的新位置。
c. 将存活对象真正移动到新位置并更新存活对象中被指向移动对象的指针。
![a0da00148bc1a9638f5ea4256f3c0b1e.jpeg](/images/gc_intro/CD89A12E-1A37-4269-B102-66ADB27404BB.jpg)

由过程可知，标记-整理算法的compact阶段的耗时是和存活对象的多少成正比的。而且虽然解决了碎片的问题，但实际需要多次遍历heap、移动对象、更新指针，所以耗时上会比标记-清除算法要更长一些。所以默认情况下一些垃圾回收器CMS并不直接使用mark-compact算法，而是当由于存在碎片导致发生Full GC时才会使用基于mark-compact算法来进行内存整理。

>PS：当然从另一个方面看，mark-compact阶段由于解决了内存碎片问题，在应用线程申请内存的时候就可以用“指针碰撞”（pointer bumping）方式来快速完成内存分配，如果碎片较多，“指针碰撞”就会较大概率导致多次指针挪动，就不适合了，这样就只能使用分配速度更慢的类似freelist的方式来进行内存的管理和分配。

##### 复制-收集（Copying GC)
由于标记-清除算法存在碎片化的问题，标记-整理算法又由于需要多次遍历heap效率较低，因此“复制-收集”算法采用了一个空间换时间的方式来解决上面的问题。“复制-收集”算法的工作过程如下：

1. 将可用内存分成大小相等的两块，from space和 to space。
2. 同一时刻只有from space会被使用。
3. GC时，将from space内存活的对象拷贝到to space
4. 拷贝完成后，from space和to space交换身份
![1290c3eb4092bd52de293fbe1be152cb.jpeg](/images/gc_intro/E583CCB7-47DF-4CE4-8945-8AC2E8135706.jpg)

>由“复制-收集”算法的实现可知，虽然这种算法比较简单高效，而且没有碎片问题。但一个比较明显的代价就是：应用程序实际可用的内存会被减半。另外，“复制-收集”算法很大一部分工作在于将存活对象移动到to space，因此它的时间开销和存活对象的数量成正比。

<i>上面介绍了主流几种“跟踪回收”类GC的具体实现算法，每一种算法都有其擅长和不足的地方，因此在实际的GC算法选择中，会根据不同的场景和特性选择不同的算法实现，从而将每种算法“扬长避短”。后面我会再继续分析现代化的GC如何进一步优化这些算法过程。</i>

***

##### 垃圾回收器
GC算法是内存回收的方法论，垃圾收集器是内存回收的具体实现。新生代垃圾收集器有Serial、ParNew、Parallel Scavenge、G1，属于老年代的垃圾收集器有CMS、Serial Old、Parallel Old和G1。其中的G1是一种既可以对新生代对象也可以对老年代对象进行回收的垃圾收集器。大部分垃圾回收器之间能相互配合来发挥各自的优势。

![85a94c88d8d3662165484268f0449723.png](/images/gc_intro/6073827-904b301be72b9f5a.jpg.png)

###### Serial收集器
Serial收集器是最古老、最基本的一种垃圾回收器，采用单线程来进行垃圾回收。新生代回收采用复制算法，回收的整个过程会暂停业务线程（STW）。

>_Serial收集器的特性总结：_
收集过程：暂停所有线程
算法：复制算法
优点：简单高效，拥有很高的单线程收集效率
应用：Client模式下的默认新生代收集器
参数控制：-XX:+UseSerialGC

![56c88d3322821a6d6af5909213e888a5.png](/images/gc_intro/6073827-42feae7608310929.png)

###### ParNew收集器
ParNew收集器也是一个新生代垃圾回收器，可以看成是Serial收集器的多线程版本，也是采用复制算法来进行收集，回收整个过程也是暂停业务线程（STW）的，但由于是多线程并行执行的，所以在多CPU机器上一般会比Serial收集器快很多。

>_ParNew收集器的特性总结：_
收集过程：暂停所有线程
算法：复制算法
优点：在CPU多的情况下，拥有比Serial更好的效果。单CPU环境下Serial效果更好
应用：许多运行在Server模式下的虚拟机中首选的新生代收集器
参数控制：
-XX:+UseParNewGC
-XX:ParallelGCThreads (限制线程数量)

![97ff89bcbbc18cd50ba5287850435f43.png](/images/gc_intro/img_97ff89bcbbc18cd50ba5287850435f43.png)

###### Parallel Scavenge收集器
和ParNew收集器类似，Parallel Scavenge收集器也是一个新生代垃圾回收器，但和ParNew收集器不同的是Parallel Scavenge收集器更多关注的是可控制的吞吐量，支持自适应调节策略。

>吞吐量 = 运行用户代码的时间/(运行用户代码的时间+垃圾收集时间)

Parallel Scavenge收集器提供几个参数控制垃圾回收的执行：

-XX:MaxGCPauseMillis
最大垃圾回收停顿时间。这个参数的原理是空间换时间，收集器会控制新生代的区域大小，从而尽可能保证回收少于这个最大停顿时间。简单的说就是回收的区域越小，那么耗费的时间也越小。所以这个参数并不是设置得越小越好。设太小的话，新生代空间会太小，从而更频繁的触发GC。

-XX:GCTimeRatio
垃圾回收时间与总时间占比（默认是99%）。这个是吞吐量的倒数，原理和MaxGCPauseMillis相同。

-XX:UseAdaptiveSizePolicy
开启动态自适应调节策略，虚拟机会根据当前系统的运行情况收集性能监控信息，动态调整上面两个参数以提供最合适的停顿时间或者最大吞吐量。

>_Parallel Scavenge收集器的特性总结：_
收集过程：暂停所有线程
算法：复制算法
优点：在CPU多的情况下，拥有比Serial更好的效果；对暂停时间和吞吐量能精细化控制。
应用：指定使用
参数控制：-XX:+UseParallelGC 


###### Serial Old收集器
老年代的收集器，与上面的Serial一样是单线程回收，不同的是算法用的是标记-整理（Mark-Compact），整个过程和Serial收集器一样都会暂停业务线程。这个收集器的主要意义也是在于给Client模式下的虚拟机使用。
![0207d0a7741701cf070520c4f97f065e.png](/images/gc_intro/serial-old.png)


###### Parallel Old收集器
Parallel Old收集器是Parallel Scavenge收集器的老年代版本，JDK 1.6开始推出的，它使用多线程和“标记-整理”算法进行垃圾回收。通常与Parallel Scavenge收集器配合使用，“吞吐量优先”收集器是这个组合的特点，在注重吞吐量和CPU资源敏感的场合，都可以使用这个组合。
![754b629778258dd09ede56a2a9f304b2.png](/images/gc_intro/parallel-old.png)

###### CMS收集器
CMS（Concurrent Mark Sweep）收集器是一种以获取最短回收停顿时间为目标的多线程并发收集器，基于“标记-清除”算法实现。CMS垃圾回收器将垃圾回收分成多个阶段，某些阶段可以和业务线程并发执行，这些并发阶段主要来做一些准备工作，这些准备工作能缩小最终的暂停阶段（Init-Mark和Remark）的时间，而且在暂停业务线程的两个阶段也由于多线程并行暂停时间控制也较好，是目前对用户响应时间敏感的且堆大小适中（太大、太小都不好）的应用广泛采用的年老代垃圾回收器。默认配合ParNew回收器作为新生代垃圾回收器。

>_CMS收集器的7个阶段：_
初始标记（CMS initial mark）
并发标记（CMS concurrent mark）
并发预清理（CMS concurrent-preclean）
并发可取消的预清理 （CMS concurrent-abortable-preclean）
重新标记（CMS remark）
并发清除（CMS concurrent sweep）
并发重置（CMS concurrent-reset）

![cms_process.png](/images/gc_intro/cms_process.jpg)

<i>PS：这个盗的图稍微有点问题，从jdk 1.8开始，initial mark阶段也是多线程并行执行的，可以通过-XX:+CMSParallelInitialMarkEnabled参数控制。</i>

> _CMS收集器的特性总结：_
收集过程：部分阶段和业务线程并发执行，部分阶段暂停所有业务线程
算法：标记-清除算法
优点：并发收集、低停顿
缺点：清除算法会产生空间碎片、并发阶段会降低吞吐量、无法即时处理并发阶段产生的“浮动垃圾”
应用：指定使用
参数控制：
-XX:+UseConcMarkSweepGC（是否采用CMS回收器）
-XX:+UseCMSInitiatingOccupancyOnly（JVM是否基于运行时收集的数据来启动CMS垃圾收集周期还是只根据初始设置来启动）
-XX:CMSInitiatingOccupancyFraction=80 （old区在使用了n%的后开始启动CMS backgroud GC）
-XX:ConcGCThreads=4 （并发CMS阶段采用多线程时使用的线程数）
 -XX:+CMSConcurrentMTEnabled （并发阶段是否采用多线程）
 -XX:+CMSParallelInitialMarkEnabled （初始标记阶段是否采用多线程并行执行）
 -XX:+CMSParallelRemarkEnabled （重新标记阶段是否采用多线程并行执行）
 -XX:+UseCMSCompactAtFullCollection （是否允许Full GC时采用整理算法）
 -XX:CMSFullGCsBeforeCompaction=0 （采用整理算法前需要Full GC次数达到一定阈值）
  -XX:+CMSClassUnloadingEnabled （GC过程也对永久代进行垃圾回收）
  -XX:ExplicitGCInvokesConcurrent （允许显示的系统GC调用）
  -XX:+CMSScavengeBeforeRemark （CMS重新标记阶段之前是否尝试对年轻代进行一次回收，能有效降低remark的时间）
  -XX:+CMSIncrementalMode （增量模式，基本不开启，cpu资源少时可尝试）
  -XX:+ExplicitGCInvokesConcurrentAndUnloadsClasses （当有系统GC调用时，永久代也被包括进CMS垃圾回收的范围内）
  
###### G1收集器
G1（Garbage First）收集器在JDK 1.7版本正式启用，在JDK 9中，G1被提议设置为默认垃圾收集器。应用在多处理器和大容量内存环境中，在实现高吞吐量的同时，尽可能的满足垃圾收集暂停时间的要求。G1收集器的设计目标是取代CMS收集器，它同CMS相比，在以下方面表现的更出色： 
  
  * G1是一个有整理内存过程的垃圾收集器，不会产生很多内存碎片。
  * G1的Stop The World(STW)更可控，G1在停顿时间上添加了预测机制，用户可以指定期望停顿时间。
 
与前面介绍回收器只能工作在年轻代或者年老代不同的是，G1收集器将整个Java堆划分为多个大小相等的独立区域（Region），虽然还保留有新生代和老年代的概念，但新生代和老年代不再是物理隔阂了，它们都是一部分（可以不连续）Region的集合。G1的新生代收集跟ParNew类似，当新生代占用达到一定比例的时候，开始出发收集。和CMS类似，G1收集器收集老年代对象会有短暂停顿。

 ![ba1807ea0d2b5304a7ad7ba2eb73f1f1.png](/images/gc_intro/2184951-715388c6f6799bd9.png)
  
>每个Region被标记了E、S、O和H，说明每个Region在运行时都充当了一种角色，其中H是以往算法中没有的，它代表Humongous，这表示这些Region存储的是巨型对象（humongous object，H-obj），当新建对象大小超过Region大小一半时，直接在新的一个或多个连续Region中分配，并标记为H。


G1收集器有三种GC模式：

* Young GC：发生在年轻代的Region，所有eden类型的Region被分配完时，触发Young GC，将存活对象从eden的Region拷贝到survior类型的Region或者晋升到old Region，然后根据历史统计信息和用户定义的暂停时间来动态调整eden Region和survivor Region的个数，从而尽量让下一次GC的回收时间满足用户期望。和ParNew一样YGC采用的“复制-收集”算法。

* Mixed GC：STW阶段，类似CMS一样当old类型的Region占用达到一定阈值的时候，G1会触发一次全局“标记”动作，统计得出收集收益高的老年代Region。“标记”工作完成后，在每次YGC之后或者再次发生Mixed GC前，JVM会检查整个堆的垃圾占比是否达到G1HeapWastePercent的阈值，如果达到了，下次就会触发一次Mixed GC（<i>一次标记后可能会分多次进行Mixed GC</i>）。Mixed GC并不仅仅是一次老年代GC，不仅进行正常的YGC回收整个young区，同时也会选择部分old Region也进行回收，和YGC一样，Mixed GC也是采用的“复制-收集”算法作为清除策略，因此整个G1垃圾回收不会存在和CMS一样的内存碎片问题。Mixed GC步骤包括 “标记”和“回收”两部分。 

* Full gc：如果对象内存分配速度过快，mixed gc来不及回收，导致老年代被填满，就会触发一次full gc，G1的full gc算法就是单线程执行的serial old gc，会导致异常长时间的暂停时间，需要进行不断的调优，尽可能的避免full gc。 
  
  其中YGC和ParNew等回收器的过程类似，比较简单，采用的是"复制-收集"算法，Mixed GC部分过程和CMS也比较相似，略有不同，Mixed GC分为"标记"和"回收"两大阶段。
  
  Mixed GC的“标记”过程如下：
  
  1. initial mark（初始标记）：会暂停所有线程，标记出所有可以直接从GC roots可以到达的对象，这是在Young GC的暂停收集阶段顺带进行的。因为 Young GC 是需要 stop-the-world 的，所以并发周期直接重用这个阶段，虽然会增加 CPU 开销，但是停顿时间只是增加了一小部分。
  2. concurrent-root-region-scan（扫描根引用区）：找出所有的GC Roots的Region, 然后从这些Region开始标记可到达的对象，是一个并发阶段。
  3. concurrent marking（并发标记）：和CMS的并发标记阶段类似，这个阶段G1递归寻找整个堆的存活对象，是和业务线程并发执行的。
  4. remark（最终标记）：STW的阶段，完成最后的存活对象标记。使用了比 CMS 收集器更加高效的 snapshot-at-the-beginning (SATB) 算法。
  5. clean up（清理阶段）：marking的最后一个阶段，G1统计各个Region的活跃性，完全没有存活对象的Region直接放入空闲可用Region列表中，然后会找出mixed GC的Region候选列表。
  
  Mixed GC的“回收”过程如下：
  
  1. 根据历史数据和“用户期望暂停时间”预测本次收集需要选择的年轻代的Region数量并进行年轻代回收。
  2. 选取标记阶段之前标记出来的老年代的垃圾最多的部分区块，结合用户“期望”的GC Pause耗时相关，选取部分Region进行回收，整个老年代标记完后可能会分多次Mixed GC执行，直到标记后的垃圾分区占比低于“堆废物占比”，之后再恢复到常规的年轻代垃圾收集，然后当堆的使用满足“触发标记周期阈值”会最终再次启动并发周期，进行下一轮循环。

*_G1常用参数_*
```
-XX:G1HeapRegionSize=n

设置的 G1 区域的大小。值是 2 的幂，范围是 1 MB 到 32 MB 之间。目标是根据最小的 Java 堆大小划分出约 2048 个区域。

-XX:MaxGCPauseMillis=200

为所需的最长暂停时间设置目标值。默认值是 200 毫秒。指定的值不适用于您的堆大小。

-XX:G1NewSizePercent=5

设置要用作年轻代大小最小值的堆百分比。默认值是 Java 堆的 5%。这是一个实验性的标志。有关示例，请参见“如何解锁实验性虚拟机标志”。此设置取代了 -XX:DefaultMinNewGenPercent 设置。Java HotSpot VM build 23 中没有此设置。

-XX:G1MaxNewSizePercent=60

设置要用作年轻代大小最大值的堆大小百分比。默认值是 Java 堆的 60%。这是一个实验性的标志。有关示例，请参见“如何解锁实验性虚拟机标志”。此设置取代了 -XX:DefaultMaxNewGenPercent 设置。Java HotSpot VM build 23 中没有此设置。

-XX:ParallelGCThreads=n

设置 STW 工作线程数的值。将 n 的值设置为逻辑处理器的数量。n 的值与逻辑处理器的数量相同，最多为 8。

如果逻辑处理器不止八个，则将 n 的值设置为逻辑处理器数的 5/8 左右。这适用于大多数情况，除非是较大的 SPARC 系统，其中 n 的值可以是逻辑处理器数的 5/16 左右。

-XX:ConcGCThreads=n

设置并行标记的线程数。将 n 设置为并行垃圾回收线程数 (ParallelGCThreads) 的 1/4 左右。

-XX:InitiatingHeapOccupancyPercent=45

设置触发标记周期的 Java 堆占用率阈值。默认占用率是整个 Java 堆的 45%。

-XX:G1MixedGCLiveThresholdPercent=65

为混合垃圾回收周期中要包括的旧区域设置占用率阈值。默认占用率为 65%。这是一个实验性的标志。有关示例，请参见“如何解锁实验性虚拟机标志”。此设置取代了 -XX:G1OldCSetRegionLiveThresholdPercent 设置。Java HotSpot VM build 23 中没有此设置。

-XX:G1HeapWastePercent=10

设置您愿意浪费的堆百分比。如果可回收百分比小于堆废物百分比，Java HotSpot VM 不会启动混合垃圾回收周期。默认值是 10%。Java HotSpot VM build 23 中没有此设置。

-XX:G1MixedGCCountTarget=8

设置标记周期完成后，对存活数据上限为 G1MixedGCLIveThresholdPercent 的旧区域执行混合垃圾回收的目标次数。默认值是 8 次混合垃圾回收。混合回收的目标是要控制在此目标次数以内。Java HotSpot VM build 23 中没有此设置。

-XX:G1OldCSetRegionThresholdPercent=10

设置混合垃圾回收期间要回收的最大旧区域数。默认值是 Java 堆的 10%。Java HotSpot VM build 23 中没有此设置。

-XX:G1ReservePercent=10

设置作为空闲空间的预留内存百分比，以降低目标空间溢出的风险。默认值是 10%。增加或减少百分比时，请确保对总的 Java 堆调整相同的量。Java HotSpot VM build 23 中没有此设置。
``` 
* * *

#### 延伸知识点1：哪些对象可以作为 GC Roots 对象呢？
             
1. 虚拟机栈（栈帧中的本地变量表）中引用的对象。
2. 方法区中类静态属性引用的对象。
3. 方法区中常量引用的对象。
4. 本地方法栈中 JNI （即 native 方法）引用的对象。
5. 分代回收算法中非当前GC年代的其他对象。


#### 延伸知识点2：如何识别“垃圾对象”？
##### *引用*
在JVM里，使用的“跟踪回收”算法从GC ROOTS出发，按照广度或者深度方式遍历所有与GC ROOTS可达的对象，根据引用类型和GC类型（是否Full GC），将某些对象标记为“使用中”，然后回收其他不在“使用中”的对象。
Java中，所有对象引用都被标识为五种类型：强引用（StrongReference）、软引用（SoftReference）、弱引用（WeakReference）、幻象引用（PhantomReference）、Final引用（FinalReference）。
![a5b1bd167692a29a29d2e09a75211e0c.png](/images/gc_intro/1119937-20190130111221088-1473128563.png)
      
##### GC如何识别和处理引用？
JVM垃圾回收器硬编码识别SoftReference，WeakReference，PhantomReference等这些具体的类，GC过程中，识别查找到引用对象没有被其他强引用使用的Reference，然后添加到一个pending列表，这个pending列表由GC来维护，通过Reference类的一个静态的pending变量（链表头）和一个实例变量discovered（链表下一节点）来实现；Reference有一个高优先级的ReferenceHandler线程，这个线程不停的从pending列表中取出待处理的Reference进行处理：有的放到Reference各自的ReferenceQueue队列里供使用者进行处理（如：PhantomReference和WeakReference）、有的直接调用固定的处理方法进行清理（如：Cleaner）。

_先看下Reference类的重要部分：_
```
public abstract class Reference<T> {
    private T referent; // 引用所指向的真实对象
    volatile ReferenceQueue<? super T> queue; //引用处理列表
    /* When active:   next element in a discovered reference list maintained by GC (or this if last)
     *     pending:   next element in the pending list (or null if last)
     *   otherwise:   NULL
     */
    transient private Reference<T> discovered;  /* used by VM ，指向pending链表下一个节点*/ 
      * References to this list, while the Reference-handler thread removes
     * them.  This list is protected by the above lock object. The
     * list uses the discovered field to link its elements.
     */
    private static Reference<Object> pending = null;  //静态的链表头
    
    private static class ReferenceHandler extends Thread {

        private static void ensureClassInitialized(Class<?> clazz) {
            try {
                Class.forName(clazz.getName(), true, clazz.getClassLoader());
            } catch (ClassNotFoundException e) {
                throw (Error) new NoClassDefFoundError(e.getMessage()).initCause(e);
            }
        }

        static {
            // pre-load and initialize InterruptedException and Cleaner classes
            // so that we don't get into trouble later in the run loop if there's
            // memory shortage while loading/initializing them lazily.
            ensureClassInitialized(InterruptedException.class);
            ensureClassInitialized(Cleaner.class);
        }

        ReferenceHandler(ThreadGroup g, String name) {
            super(g, name);
        }

        public void run() {
            while (true) { //死循环执行
                tryHandlePending(true);
            }
        }
    }
    
    static boolean tryHandlePending(boolean waitForNotify) {
        Reference<Object> r;
        Cleaner c;
        try {
            synchronized (lock) {
                if (pending != null) {
                    r = pending;
                    // 'instanceof' might throw OutOfMemoryError sometimes
                    // so do this before un-linking 'r' from the 'pending' chain...
                    c = r instanceof Cleaner ? (Cleaner) r : null;
                    // unlink 'r' from 'pending' chain
                    pending = r.discovered;
                    r.discovered = null;
                } else {
                    // The waiting on the lock may cause an OutOfMemoryError
                    // because it may try to allocate exception objects.
                    if (waitForNotify) {
                        lock.wait();
                    }
                    // retry if waited
                    return waitForNotify;
                }
            }
        } catch (OutOfMemoryError x) {
            // Give other threads CPU time so they hopefully drop some live references
            // and GC reclaims some space.
            // Also prevent CPU intensive spinning in case 'r instanceof Cleaner' above
            // persistently throws OOME for some time...
            Thread.yield();
            // retry
            return true;
        } catch (InterruptedException x) {
            // retry
            return true;
        }

        // Fast path for cleaners
        if (c != null) {
            c.clean();
            return true;
        }

        ReferenceQueue<? super Object> q = r.queue;
        if (q != ReferenceQueue.NULL) q.enqueue(r);
        return true;
    }

   //类加载完就起好最高优先级的ReferenceHandler
   static {
        ThreadGroup tg = Thread.currentThread().getThreadGroup();
        for (ThreadGroup tgn = tg;
             tgn != null;
             tg = tgn, tgn = tg.getParent());
        Thread handler = new ReferenceHandler(tg, "Reference Handler");
        /* If there were a special system-only priority greater than
         * MAX_PRIORITY, it would be used here
         */
        handler.setPriority(Thread.MAX_PRIORITY);
        handler.setDaemon(true);
        handler.start();

        // provide access in SharedSecrets
        SharedSecrets.setJavaLangRefAccess(new JavaLangRefAccess() {
            @Override
            public boolean tryHandlePendingReference() {
                return tryHandlePending(false);
            }
        });
    }
    // ...
}
```

ReferenceQueue可以由使用方通过Reference的构造方法指定传入，如果没有指定，从pending链表取出的Reference都enqueue到全局的一个ENQUEUED队列中。

_ReferenceQueue的实现：_
```
public class ReferenceQueue<T> {

    /**
     * Constructs a new reference-object queue.
     */
    public ReferenceQueue() { }

    private static class Null<S> extends ReferenceQueue<S> {
        boolean enqueue(Reference<? extends S> r) {
            return false;
        }
    }

    static ReferenceQueue<Object> NULL = new Null<>();
    static ReferenceQueue<Object> ENQUEUED = new Null<>();

    static private class Lock { };
    private Lock lock = new Lock();
    private volatile Reference<? extends T> head = null;
    private long queueLength = 0;

    boolean enqueue(Reference<? extends T> r) { /* Called only by Reference class */
        synchronized (lock) {
            // Check that since getting the lock this reference hasn't already been
            // enqueued (and even then removed)
            ReferenceQueue<?> queue = r.queue;
            if ((queue == NULL) || (queue == ENQUEUED)) {
                return false;
            }
            assert queue == this;
            r.queue = ENQUEUED;
            r.next = (head == null) ? r : head;
            head = r;
            queueLength++;
            if (r instanceof FinalReference) {
                sun.misc.VM.addFinalRefCount(1);
            }
            lock.notifyAll();
            return true;
        }
    }
}
```
##### *引用种类*
###### 强引用
最常见的引用方式。如：
```
Object obj = new Object();
```
上面的obj就是一个指向Object对象的强引用
强引用如果有到GC ROOTS的路径，那么这个引用指向的对象在GC时不能被回收。

>在实际使用中，除了强引用，可能还需要一些其他特殊类型的引用，比如有些缓存对象，是可以在内存不足时来回收的，这样通过丰富的引用类型，能让内存在实际使用时更灵活，整体业务稳定性更好。jdk 1.2后，对引用概念进行了扩充，增加了其他4种类型的引用，在java.lang.ref包下，分别为：SoftReference、WeakReference、PhantomReference、FinalReference。

###### 软引用（SoftReference）
```
SoftReference<String> str = new SoftReference<String>("abc");
```
软引用是比强引用稍弱的一种引用，普通的GC并不会回收软引用，只有在即将OOM的时候（也就是最后一次Full GC）的时候才会回收软引用指向的对象。所以，软引用比较适合用来实现不是特别重要的缓存，比如guava cache就支持软引用类型的存储值。

```
//如果只是普通增量GC，不回收软引用
if (!gch->incremental_collection_will_fail(false /* don't consult_young */)) {
gch->do_collection(false            /* full */,
                       false            /* clear_all_soft_refs */,
                       size             /* size */,
                       is_tlab          /* is_tlab */,
                       number_of_generations() - 1 /* max_level */);
...
//否则，触发Full GC，但还不回收软引用
} else {
    gch->do_collection(true             /* full */,
                       false            /* clear_all_soft_refs */,
                       size             /* size */,
                       is_tlab          /* is_tlab */,
                       number_of_generations() - 1 /* max_level */);
}
...
//如果还是内存不够时，会触发一次回收所有软引用的Full GC，再不行就OOM
gch->do_collection(true             /* full */,
                       true             /* clear_all_soft_refs */,
                       size             /* size */,
                       is_tlab          /* is_tlab */,
                       number_of_generations() - 1 /* max_level */);
  }

```

###### 弱引用（WeakReference）
弱引用比软引用的引用级别更低一些，GC时，如果一个对象从GC ROOTS出发，只有弱引用指向没有其他（强引用或软引用）指向时，这个对象就会在本次GC被回收掉。

弱引用最常见的使用情景是WeakHashMap，WeakHashMap里面的Entry是一个弱引用，这个弱引用指向Map的Key，如果这个Key没有被其他”强引用“或者”软引用“引用时，GC会干掉这个Key对象，同时将这个Entry对象放入WeakHashMap的ReferenceQueue中等待被处理，当WeakHashMap的get、put等方法被调用时，会通过expungeStaleEntries方法把这个ReferenceQueue的Entry对象的value置空并调整Entry链表摘取当前Entry，这样下次GC时就能回收掉value的对象了。
>一般在需要控制内存使用但又想尽量用到更多内存的场景下使用。比如tomcat的ConcurrentCache就用到了WeakHashMap来作为分级缓存，可以在内存充足的情况下，缓存尽量多的数据，同时又不会导致OOM。

```
 private static class Entry<K,V> extends WeakReference<Object> implements Map.Entry<K,V> {
        V value;
        final int hash;
        Entry<K,V> next;

        /**
         * Creates new entry.
         */
        Entry(Object key, V value,
              ReferenceQueue<Object> queue,
              int hash, Entry<K,V> next) {
            super(key, queue);
            this.value = value;
            this.hash  = hash;
            this.next  = next;
        }
}

/**
     * Expunges stale entries from the table.
     */
    private void expungeStaleEntries() {
        for (Object x; (x = queue.poll()) != null; ) {
            synchronized (queue) {
                @SuppressWarnings("unchecked")
                    Entry<K,V> e = (Entry<K,V>) x;
                int i = indexFor(e.hash, table.length);

                Entry<K,V> prev = table[i];
                Entry<K,V> p = prev;
                while (p != null) {
                    Entry<K,V> next = p.next;
                    if (p == e) {
                        if (prev == e)
                            table[i] = next;
                        else
                            prev.next = next;
                        // Must not null out e.next;
                        // stale entries may be in use by a HashIterator
                        e.value = null; // Help GC
                        size--;
                        break;
                    }
                    prev = p;
                    p = next;
                }
            }
        }
    }
```

###### 幻象引用（PhantomReference）
幻象引用是比弱引用的级别更低的一种，和软引用以及弱引用不同的是幻影引用指向的对象没有其他强引用、软引用指向时不会自动被GC清理。
_PhantomReference的回收处理过程如下：_
GC时，高优先级的ReferenceHandler线程将这个PhantomReference放到它自身的静态ReferenceQueue中，然后PhantomReference的实现子类一般会有一个线程在不断轮询这个ReferenceQueue，从queue中取出PhantomReference并调用它自己实现的清理方法来释放它所指向对象的占用的一些特定资源并把PhantomReference自身从ReferenceQueue中干掉，这样下次GC时这个幻象引用本身和它指向的对象也能够被GC掉。

_实际场景中，一般幻象引用用于对象需要被清理时，除了指向的对象本身外，还需要额外释放这个对象占用的其他资源的场景。比如DB连接池使用PhantomReference来释放底层的socket资源，比如DirectByteBuffer使用PhantomReference来释放底层占用的堆外内存。_

###### 举例：
*DirectByteBuffer使用PhantomReference管理堆外内存释放的过程如下：*
>每个DirectByteBuffer在生成时会绑定一个Cleaner对象，这个Cleaner对象是一个PhantomReference，当JVM GC时发现那些除了Cleaner幻象引用外已没有其他引用的DirectByteBuffer时，就会把这些Cleaner对象放到Reference这个类的pending列表里，Reference类维护了一条ReferenceHandler的高优先级线程，这条线程会不断去轮询待处理的pending列表，如果是Cleaner对象就调用这个对象的clean方法进行清理（_这里需要注意的是：Cleaner是一种特殊的PhantomReference，它实际的清理工作是由ReferenceHandler线程直接执行的，不需要自己再维护一个清理的线程_），clean方法里其实是调用初始化Cleaner时绑定的Deallocator间接使用unsafe.freeMemory来进行堆外内存的释放和Bits里全局堆外内存使用量的更新。

###### Final引用（PhantomReference）
* FinalReference & Finalizer
在java.lang.ref包下，除了上面四种引用，还有一个非公开的FinalReference引用以及它的一个子类Finalizer。实际上，FinalReference 代表的正是 Java 中的强引用，如这样的代码 :
```
Bean bean = new Bean();
```
在虚拟机的实现过程中，实际采用了 FinalReference 类对其进行引用。而 Finalizer，除了作为一个实现类外，更是在虚拟机中实现一个 FinalizerThread，以使虚拟机能够在所有的强引用被解除后实现内存清理。

Finalizer的工作过程：
GC过程中，当一个强引用对象bean没有了引用被标记为可回收时，如果这个bean对象的类定义了finalize方法，那么这个对象被绑定到一个Finalizer引用上，这个Finalizer引用会被前面讲过的ReferenceHandler线程将引用自身加入到Finalizer静态的ReferenceQueue中，同时Finalizer对象自带的静态的优先级为8（比普通线程优先级高但比ReferenceHandler优先级低）的FinalizerThread线程会轮询这个ReferenceQueue中的Finalizer引用，然后调用它的runFinalizer方法，最终调到了绑定的那个bean对象的finalize方法，当finalize方法的逻辑都执行完后，这个bean对象才会在下次GC时被回收。

*这里有几个需要注意的点：*
1. 在CPU资源比较紧张的情况下，由于FinalizerThread线程优先级较低，可能由于得不到时间片而导致finalize的方法执行延迟或者缓慢。最终可能导致bean对象真正占用的资源释放不确定性提高，另外也可能会导致由于bean对象无法回收导致Full GC甚至OOM。
2. 底层的bean对象至少需要2次GC才会被回收。第一次GC只是标记后将处理任务提交给FinalizerThread线程去执行，当FinalizerThread执行完finalize方法后才会在下次GC时回收bean对象，FinalizerThread执行期间可能经历多次GC。
3. 和PhantomReference不同，由于Finalizer引用最终的释放依赖对象的finalize方法的实现，在finalize里实际上可以访问到引用的对象本身，所以如果在finalize方法里让其他对象又引用了当前对象，这样会导致这个本应该被回收的对象复活。

>高优先级的ReferenceHandler线程将Finalizer引用加入到Finalizer类的静态ReferenceQueue中
```
static boolean tryHandlePending(boolean waitForNotify) {
    ...
    ReferenceQueue<? super Object> q = r.queue;
    if (q != ReferenceQueue.NULL) q.enqueue(r);
}
```
>FinalizerThread线程处理：
1. 将当前Finalizer引用从ReferenceQueue里删除
2. 执行Finalizer引用的runFinalizer来触发bean对象的清除操作
```
private static class FinalizerThread extends Thread {
...
public void run() {
    if (running) return;
    for (;;) {
                try {
                    Finalizer f = (Finalizer)queue.remove();
                    f.runFinalizer(jla);
                } catch (InterruptedException x) {
                    // ignore and continue
                }
            }
}
```
>jla.invokeFinalize实际调用了bean对象的finalize方法

```
private void runFinalizer(JavaLangAccess jla) {
        synchronized (this) {
            if (hasBeenFinalized()) return;
            remove();
        }
        try {
            Object finalizee = this.get();
            if (finalizee != null && !(finalizee instanceof java.lang.Enum)) {
                jla.invokeFinalize(finalizee);

                /* Clear stack slot containing this variable, to decrease
                   the chances of false retention with a conservative GC */
                finalizee = null;
            }
        } catch (Throwable x) { }
        super.clear();
    }
```
>finalize方法里将其他引用指向bean对象本身，导致bean对象复活
```
public void finalize() {
    Other.ref = this;
}
```
>而在PhantomReference实现中，由于get方法默认返回null，因此PhantomReference创建后就不能再访问它的referent，因此不存在死对象复活的问题。所以，尽量使用PhantomReference来替代原来Finalizer的功能。
```
public class PhantomReference<T> extends Reference<T> {
    public T get() {
        return null;
    }
}
```

##### GC过程是如何具体处理引用?
GC时，各垃圾收集器都会在过程中，先找到当前回收代中那些”指向的referent除了当前引用外没有其他强引用使用了”的引用，然后对所有类型的Reference（soft、weak、phantom、final）进行进一步筛选和排除处理，最后，在GC的最后阶段，将待处理的Reference入队到Reference的pendingList等待ReferHandler线程来善后。


###### 发现引用
_“引用发现”这个过程的主要工作：_找出当前回收代中的这些引用：“指向的referent没有被其它强引用所使用”。_
如ParNew垃圾回收器中的Ref"发现-标记"过程（parNewGeneration.cpp）：

>先处理roots触发，引用到的对象都拷贝到to space，然后通过par_scan_state.evacuate_followers_closure().do_void()在to space里遍历全部年轻代存活对象。</i>

```
gch->gen_process_roots(_gen->level(),
                         true,  // Process younger gens, if any,
                                // as strong roots.
                         false, // no scope; this is parallel code
                         GenCollectedHeap::SO_ScavengeCodeCache,
                         GenCollectedHeap::StrongAndWeakRoots,
                         &par_scan_state.to_space_root_closure(),
                         &par_scan_state.older_gen_closure(),
                         &cld_scan_closure);

  par_scan_state.end_strong_roots();

  // "evacuate followers".
  par_scan_state.evacuate_followers_closure().do_void();
 ```
>EvacuateFollowersClosureGeneral::do_void方法主要执行父类DefNewGeneration的oop_since_save_marks_iterate方法。

```
void DefNewGeneration::                                         \
oop_since_save_marks_iterate##nv_suffix(OopClosureType* cl) {   \
  cl->set_generation(this);                                     \
  eden()->oop_since_save_marks_iterate##nv_suffix(cl);          \
  to()->oop_since_save_marks_iterate##nv_suffix(cl);            \
  from()->oop_since_save_marks_iterate##nv_suffix(cl);          \
  cl->reset_generation();                                       \
  save_marks();                                                 \
}
```
>.space的oop_since_save_marks_iterate会调用每一个对象的oop_iterate
```
void ContiguousSpace::                                                    \
oop_since_save_marks_iterate##nv_suffix(OopClosureType* blk) {            \
  HeapWord* t;                                                            \
  HeapWord* p = saved_mark_word();                                        \
  assert(p != NULL, "expected saved mark");                               \
                                                                          \
  const intx interval = PrefetchScanIntervalInBytes;                      \
  do {                                                                    \
    t = top();                                                            \
    while (p < t) {                                                       \
      Prefetch::write(p, interval);                                       \
      debug_only(HeapWord* prev = p);                                     \
      oop m = oop(p);                                                     \
      p += m->oop_iterate(blk);                                           \
    }                                                                     \
  } while (t < top());                                                    \
                                                                          \
  set_saved_mark_word(p);                                                 \
}
```
>.oop是堆上对象的基类，它的oop_iterate实际上会调到对应类的oop_oop_iterate##nv_suffix方法，对于引用类型的对象，会走到InstanceRefKlass的op_oop_iterate##nv_suffix的方法。
```
inline int oopDesc::oop_iterate(OopClosureType* blk) {                     \
  SpecializationStats::record_call();                                      \
  return klass()->oop_oop_iterate##nv_suffix(this, blk);               \
}
```
>InstanceRefKlass的oop_oop_iterate##nv_suffix会调用InstanceRefKlass_SPECIALIZED_OOP_ITERATE这个语句块。
```
int InstanceRefKlass::                                                          \
oop_oop_iterate##nv_suffix(oop obj, OopClosureType* closure) {                  \
  /* Get size before changing pointers */                                       \
  SpecializationStats::record_iterate_call##nv_suffix(SpecializationStats::irk);\
                                                                                \
  int size = InstanceKlass::oop_oop_iterate##nv_suffix(obj, closure);           \
                                                                                \
  if (UseCompressedOops) {                                                      \
    InstanceRefKlass_SPECIALIZED_OOP_ITERATE(narrowOop, nv_suffix, contains);   \
  } else {                                                                      \
    InstanceRefKlass_SPECIALIZED_OOP_ITERATE(oop, nv_suffix, contains);         \
  }                                                                             \
}
```
>在InstanceRefKlass_SPECIALIZED_OOP_ITERATE这个语句块中，这里会执行最终的"引用发现"discover_reference方法。
```
#define InstanceRefKlass_SPECIALIZED_OOP_ITERATE(T, nv_suffix, contains)        \
  ...
  if (!referent->is_gc_marked() && (rp != NULL) &&                            \
      rp->discover_reference(obj, reference_type())) {                        \
      return size;                                                              \
  } 
  ...
```
>discover_reference方法中，对"发现"的各种引用，会真正操作Reference的discovered字段来维护"发现"的引用链表。
```
bool ReferenceProcessor::discover_reference(oop obj, ReferenceType rt) {
...
//只收集那些“指向的referent没有被其它强引用所使用的引用”
// We only discover references whose referents are not (yet)
  // known to be strongly reachable.
  if (is_alive_non_header() != NULL) {
    verify_referent(obj);
    if (is_alive_non_header()->do_object_b(java_lang_ref_Reference::referent(obj))) {
      return false;  // referent is reachable
    }
  }
...
//将找到的各种引用加入到收集的引用列表
if (_discovery_is_mt) {
    add_to_discovered_list_mt(*list, obj, discovered_addr);
  } else {
    // We do a raw store here: the field will be visited later when processing
    // the discovered references.
    oop current_head = list->head();
    // The last ref must have its discovered field pointing to itself.
    oop next_discovered = (current_head != NULL) ? current_head : obj;

    assert(discovered == NULL, "control point invariant");
    oop_store_raw(discovered_addr, next_discovered);
    list->set_head(obj);
    list->inc_length(1);

    if (TraceReferenceGC) {
      gclog_or_tty->print_cr("Discovered reference (" INTPTR_FORMAT ": %s)",
                                (void *)obj, obj->klass()->internal_name());
    }
```
###### 处理引用
针对之前"发现"的引用，GC过程还会通过ReferenceProcessor的process\_discovered\_references来对所有类型的Reference（soft、weak、phantom、final）进行处理，处理步骤分成3个阶段，主要工作：过滤软引用、过滤referent存活的引用 、

还是是ParNew回收器为例，GC时，在完成“引用发现”后，会通过process_discovered_references方法对引用进行处理，最后GC完成内存回收后再将Reference加入到pending列表中以便后续处理。

```
void ParNewGeneration::collect(bool   full,
                               bool   clear_all_soft_refs,
                               size_t size,
                               bool   is_tlab) {
   ...
   //处理引用
   if (rp->processing_is_mt()) {
    ParNewRefProcTaskExecutor task_executor(*this, thread_state_set);
    stats = rp->process_discovered_references(&is_alive, &keep_alive,
                                              &evacuate_followers, &task_executor,
                                              _gc_timer, gc_tracer.gc_id());
      } else {
        thread_state_set.flush();
        gch->set_par_threads(0);  // 0 ==> non-parallel.
        gch->save_marks();
        stats = rp->process_discovered_references(&is_alive, &keep_alive,
                                                  &evacuate_followers, NULL,
                                                  _gc_timer, gc_tracer.gc_id());
      }
   ...
   //通过将当前引用列表附到原来的pending链表以便ReferenceHandler线程的善后处理
    rp->set_enqueuing_is_done(true);
      if (rp->processing_is_mt()) {
        ParNewRefProcTaskExecutor task_executor(*this, thread_state_set);
        rp->enqueue_discovered_references(&task_executor);
      } else {
        rp->enqueue_discovered_references(NULL);
      }
      ...
    }
```
>处理”被发现“的引用的方法process_discovered_references主要实现在referenceProcessor.cpp的process_discovered_reflist中，包括三个阶段。
1. 第一个阶段只处理软引用：因为软引用普通GC时是不能回收处理的，所以需要从discovered链表中移除所有不存活但是还不能被回收的软引用；
2. 第二阶段处理所有引用：移除所有那些referent还存活的引用。
3. 第三阶段处理剩下引用的referent：根据clear_referent的值决定是否将对referent的引用解除，方便下一次GC时回收referent。 （比如Final和Phantom是先不解除引用的，因为后续还要用；Weak是可以解除的）。
```
ReferenceProcessor::process_discovered_reflist(
  DiscoveredList               refs_lists[],
  ReferencePolicy*             policy,
  bool                         clear_referent,
  BoolObjectClosure*           is_alive,
  OopClosure*                  keep_alive,
  VoidClosure*                 complete_gc,
  AbstractRefProcTaskExecutor* task_executor)
{...
    // Phase 1 (soft refs only):
  // . Traverse the list and remove any SoftReferences whose
  //   referents are not alive, but that should be kept alive for
  //   policy reasons. Keep alive the transitive closure of all
  //   such referents.
  if (policy != NULL) {
    if (mt_processing) {
      RefProcPhase1Task phase1(*this, refs_lists, policy, true /*marks_oops_alive*/);
      task_executor->execute(phase1);
    } else {
      for (uint i = 0; i < _max_num_q; i++) {
        process_phase1(refs_lists[i], policy,
                       is_alive, keep_alive, complete_gc);
      }
    }
  } else { // policy == NULL
    assert(refs_lists != _discoveredSoftRefs,
           "Policy must be specified for soft references.");
  }

  // Phase 2:
  // . Traverse the list and remove any refs whose referents are alive.
  if (mt_processing) {
    RefProcPhase2Task phase2(*this, refs_lists, !discovery_is_atomic() /*marks_oops_alive*/);
    task_executor->execute(phase2);
  } else {
    for (uint i = 0; i < _max_num_q; i++) {
      process_phase2(refs_lists[i], is_alive, keep_alive, complete_gc);
    }
    
     // Phase 3:
  // . Traverse the list and process referents as appropriate.
  if (mt_processing) {
    RefProcPhase3Task phase3(*this, refs_lists, clear_referent, true /*marks_oops_alive*/);
    task_executor->execute(phase3);
  } else {
    for (uint i = 0; i < _max_num_q; i++) {
      process_phase3(refs_lists[i], clear_referent,
                     is_alive, keep_alive, complete_gc);
    }
  }
```
>处理完引用后，GC回收完内存后，在结束阶段会通过ReferenceProcessor的enqueue_discovered_references方法来将处理过的引用通过pending链表来进行入队列，这样ReferenceHandler才能把队列里的Reference对象从pending链表取出后写入到ReferenceQueue或者进行clean（Cleaner）操作。enqueue_discovered_references会根据是否使用压缩指针选择不同的enqueue_discovered_ref_helper()模板函数。
```
bool ReferenceProcessor::enqueue_discovered_references(AbstractRefProcTaskExecutor* task_executor) {
  NOT_PRODUCT(verify_ok_to_handle_reflists());
  if (UseCompressedOops) {
    return enqueue_discovered_ref_helper<narrowOop>(this, task_executor);
  } else {
    return enqueue_discovered_ref_helper<oop>(this, task_executor);
  }
}
```
>pending_list_addr是Reference类的pending链表的首元素地址，enqueue_discovered_reflists过程会把符合的引用加入到这个链表，ReferenceHandler则从pending链表取出引用后放入ReferenceQueue或直接处理（Cleaner）。
```
bool enqueue_discovered_ref_helper(ReferenceProcessor* ref,
                                   AbstractRefProcTaskExecutor* task_executor) {
  T* pending_list_addr = (T*)java_lang_ref_Reference::pending_list_addr();
  T old_pending_list_value = *pending_list_addr;
   oopDesc::bs()->write_ref_field(pending_list_addr, oopDesc::load_decode_heap_oop(pending_list_addr));
  ref->enqueue_discovered_reflists((HeapWord*)pending_list_addr, task_executor);
  ...
  ref->disable_discovery();
  return old_pending_list_value != *pending_list_addr;
}
```
>多线程和单线程处理方式，由-XX:+ParallelRefProcEnabled控制，默认单线程，实际处理代码在enqueue_discovered_reflist中，主要逻辑如下：
```
void ReferenceProcessor::enqueue_discovered_reflist(DiscoveredList& refs_list,
                                                    HeapWord* pending_list_addr) {
    if (pending_list_uses_discovered_field()) { // New behavior
    // Walk down the list, self-looping the next field
    // so that the References are not considered active.
    while (obj != next_d) {
      obj = next_d;
      assert(obj->is_instanceRef(), "should be reference object");
      next_d = java_lang_ref_Reference::discovered(obj);
      if (TraceReferenceGC && PrintGCDetails) {
        gclog_or_tty->print_cr("        obj " INTPTR_FORMAT "/next_d " INTPTR_FORMAT,
                               (void *)obj, (void *)next_d);
      }
      assert(java_lang_ref_Reference::next(obj) == NULL,
             "Reference not active; should not be discovered");
      // Self-loop next, so as to make Ref not active.
      java_lang_ref_Reference::set_next_raw(obj, obj);
      if (next_d != obj) {
        oopDesc::bs()->write_ref_field(java_lang_ref_Reference::discovered_addr(obj), next_d);
      } else {
        // This is the last object.
        // Swap refs_list into pending_list_addr and
        // set obj's discovered to what we read from pending_list_addr.
        oop old = oopDesc::atomic_exchange_oop(refs_list.head(), pending_list_addr);
        // Need post-barrier on pending_list_addr. See enqueue_discovered_ref_helper() above.
        java_lang_ref_Reference::set_discovered_raw(obj, old); // old may be NULL
        oopDesc::bs()->write_ref_field(java_lang_ref_Reference::discovered_addr(obj), old);
      }
      ...
    }
```
###### 整个引用大致的处理流程（懒得画，盗个图）
![04dd63008299487f05620952f0bd5dff.png](/images/gc_intro/image-20180819185503111.png)


* * *
_参考：_
[垃圾回收统一理论](http://120.52.51.16/www.cs.virginia.edu/~cs415/reading/bacon-garbage.pdf)
[Introduction to Garbage Collection](http://www.math.grin.edu/~rebelsky/Courses/CS302/99S/Presentations/GC/)
[ R大：并发垃圾收集器（CMS）为什么没有采用标记-整理算法来实现？](https://hllvm-group.iteye.com/group/topic/38223)
[R大：请教Weak Reference及其在HotSpot GC中的行为](https://hllvm-group.iteye.com/group/topic/39402)
[垃圾优先型垃圾回收器调优](https://www.oracle.com/technetwork/cn/articles/java/g1gc-1984535-zhs.html)
[G1: One Garbage Collector To Rule Them All](https://www.infoq.com/articles/G1-One-Garbage-Collector-To-Rule-Them-All)
[JDK源码阅读-Reference](http://imushan.com/2018/08/19/java/language/JDK源码阅读-Reference/)



