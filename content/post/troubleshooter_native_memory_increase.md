---
title: "Java堆外内存增长问题排查Case"
date: 2018-08-15T16:30:00+08:00
categories: ["技术"]
tags: ["问题排查","Java","JVM"]
thumbnail: "images/IMG_6127.JPG"
draft: false
---
最近排查一个线上java服务常驻内存异常高的问题，大概现象是：java堆Xmx配置了8G，但运行一段时间后常驻内存RES从5G逐渐增长到13G #补图#，导致机器开始swap从而服务整体变慢。
由于Xmx只配置了8G但RES常驻内存达到了13G，多出了5G堆外内存，经验上判断这里超出太多不太正常。

### 前情提要--JVM内存模型
开始逐步对堆外内存进行排查，首先了解一下JVM内存模型。根据JVM规范，JVM运行时数据区共分为虚拟机栈、堆、方法区、程序计数器、本地方法栈五个部分。
![o_dc695f48-4189-4fc7-b950-ed25f6c1521708518830](/images/o_dc695f48-4189-4fc7-b950-ed25f6c1521708518830.jpg)￼

* 虚拟机栈：每个线程有一个私有的栈，随着线程的创建而创建。栈里面存着的是一种叫“栈帧”的东西，每个方法会创建一个栈帧，栈帧中存放了局部变量表（基本数据类型和对象引用）、操作数栈、方法出口等信息。栈的大小可以固定也可以动态扩展。当栈调用深度大于JVM所允许的范围，会抛出StackOverflowError的错误，不过这个深度范围不是一个恒定的值。虚拟机栈除了上述错误外，还有另一种错误，那就是当申请不到空间时，会抛出 OutOfMemoryError。
* 本地方法栈：与虚拟机栈类似，区别是虚拟机栈执行java方法，本地方法站执行native方法。在虚拟机规范中对本地方法栈中方法使用的语言、使用方法与数据结构没有强制规定，因此虚拟机可以自由实现它。本地方法栈也可以抛出StackOverflowError和OutOfMemoryError。
* PC 寄存器，也叫程序计数器。可以看成是当前线程所执行的字节码的行号指示器。在任何一个确定的时刻，一个处理器（对于多内核来说是一个内核）都只会执行一条线程中的指令。因此，为了线程切换后能恢复到正确的执行位置，每条线程都需要一个独立的程序计数器，我们称这类内存区域为“线程私有”内存。倘若当前线程执行的是 JAVA 的方法，则该寄存器中保存当前执行指令的地址；倘若执行的是native 方法，则PC寄存器中为空。
* 堆内存。堆内存是 JVM 所有线程共享的部分，在虚拟机启动的时候就已经创建。所有的对象和数组都在堆上进行分配。这部分空间可通过 GC 进行回收。当申请不到空间时会抛出 OutOfMemoryError。
* 方法区也是所有线程共享。主要用于存储类的信息、常量池、静态变量、及时编译器编译后的代码等数据。方法区逻辑上属于堆的一部分，但是为了与堆进行区分，通常又叫“非堆”。 

#### 前情提要--PermGen（永久代）和 Metaspace（元空间）
PermGen space 和 Metaspace是HotSpot对于方法区的不同实现。在Java虚拟机（以下简称JVM）中，类包含其对应的元数据，比如类名,父类名,类的类型,访问修饰符,字段信息,方法信息,静态变量,常量,类加载器的引用,类的引用。在HotSpot JDK 1.8之前这些类元数据信息存放在一个叫永久代的区域（PermGen space），永久代一段连续的内存空间。在JDK 1.8开始，方法区实现采用Metaspace代替，这些元数据信息直接使用本地内存来分配。元空间与永久代之间最大的区别在于：元空间并不在虚拟机中，而是使用本地内存。

#### 堆外内存
java 8下是指除了Xmx设置的java堆（java 8以下版本还包括MaxPermSize设定的持久代大小）外，java进程使用的其他内存。主要包括：DirectByteBuffer分配的内存，JNI里分配的内存，线程栈分配占用的系统内存，jvm本身运行过程分配的内存，codeCache，java 8里还包括metaspace元数据空间。

#### 分析java堆
由于现象是RES比较高，先看一下java堆是否有异常。把java堆dump下来仔细排查一下，jmap -histo:live pid，发现整个堆回收完也才几百兆，远不到8G的Xmx的上限值，GC日志看着也没啥异常。基本排查java堆内存泄露的可能性。 
![](/images/15331172189388.jpg)￼

#### 分析DirectByteBuffer的占用
##### DirectByteBuffer简单了解
由于服务使用的RPC框架底层采用了Netty等NIO框架，会使用到DirectByteBuffer这种“冰山对象”，先简单排查一下。关于DirectByteBuffer先介绍一下：JDK 1.5之后ByteBuffer类提供allocateDirect(int capacity)进行堆外内存的申请，底层通过unsafe.allocateMemory(size)实现，会调用malloc方法进行内存分配。实际上，在java堆里是维护了一个记录堆外地址和大小的DirectByteBuffer的对象，所以GC是能通过操作DirectByteBuffer对象来间接操作对应的堆外内存，从而达到释放堆外内存的目的。但如果一旦这个DirectByteBuffer对象熬过了young GC到达了Old区，同时Old区一直又没做CMS GC或者Full GC的话，这些“冰山对象”会将系统物理内存慢慢消耗掉。对于这种情况JVM留了后手，Bits给DirectByteBuffer前首先需要向Bits类申请额度，Bits类维护了一个全局的totalCapacity变量，记录着全部DirectByteBuffer的总大小，每次申请，都先看看是否超限（堆外内存的限额默认与堆内内存Xmx设定相仿），如果已经超限，会主动执行Sytem.gc()，System.gc()会对新生代的老生代都会进行内存回收，这样会比较彻底地回收DirectByteBuffer对象以及他们关联的堆外内存。但如果启动时通过-DisableExplicitGC禁止了System.gc()，那么这里就会出现比较严重的问题，导致回收不了DirectByteBuffer底下的堆外内存了。所以在类似Netty的框架里对DirectByteBuffer是框架自己主动回收来避免这个问题。
![](/images/15331235138702.jpg)￼

##### DirectByteBuffer为什么要用堆外内存
DirectByteBuffer是直接通过native方法使用malloc分配内存，这块内存位于java堆之外，对GC没有影响；其次，在通信场景下，堆外内存能减少IO时的内存复制，不需要堆内存Buffer拷贝一份到直接内存中，然后才写入Socket中。所以DirectByteBuffer一般用于通信过程中作为缓冲池来减少内存拷贝。当然，由于直接用malloc在OS里申请一段内存，比在已申请好的JVM堆内内存里划一块出来要慢，所以在Netty中一般用池化的 PooledDirectByteBuf 对DirectByteBuffer进行重用进一步提升性能。
##### 如何排查DirectByteBuffer的使用情况
JMX提供了监控direct buffer的MXBean，启动服务时开启-Dcom.sun.management.jmxremote.port=9527 -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Djava.rmi.server.hostname=10.79.40.147，JMC挂上后运行一段时间，此时Xmx是8G的情况下整体RES逐渐增长到13G，MBean里找到java.nio.BufferPool下的direct节点，查看direct buffer的情况，发现总共才213M。为了进一步排除，在启动时通过-XX:MaxDirectMemorySize来限制DirectByteBuffer的最大限额，调整为1G后，进程整体常驻内存的增长并没有限制住，因此这里基本排除了DirectByteBuffer的嫌疑。
![](/images/15331238716181.jpg)￼
#### 使用NMT排查JVM原生内存使用
##### Native Memory Tracking（NMT）使用
NMT是Java7U40引入的HotSpot新特性，可用于监控JVM原生内存的使用，但比较可惜的是，目前的NMT不能监控到JVM之外或原生库分配的内存。java进程启动时指定开启NMT（有一定的性能损耗），输出级别可以设置为“summary”或“detail”级别。如：
```    
-XX:NativeMemoryTracking=summary 或者 -XX:NativeMemoryTracking=detail
```
开启后，通过jcmd可以访问收集到的数据。
```
jcmd <pid> VM.native_memory [summary | detail | baseline | summary.diff | detail.diff 
```

如：jcmd 11 VM.native_memory，输出如下：

```
Native Memory Tracking:

Total: reserved=12259645KB（保留内存）, committed=11036265KB （提交内存）
堆内存使用情况，保留内存和提交内存和Xms、Xmx一致，都是8G。
-                 Java Heap (reserved=8388608KB, committed=8388608KB)
                            (mmap: reserved=8388608KB, committed=8388608KB)
用于存储类元数据信息使用到的原生内存，总共12045个类，整体实际使用了79M内存。
-                     Class (reserved=1119963KB, committed=79751KB)
                            (classes #12045)
                            (malloc=1755KB #29277)
                            (mmap: reserved=1118208KB, committed=77996KB)
总共2064个线程，提交内存是2.1G左右，一个线程1M，和设置Xss1m相符。
-                    Thread (reserved=2130294KB, committed=2130294KB)
                            (thread #2064)
                            (stack: reserved=2120764KB, committed=2120764KB)
                            (malloc=6824KB #10341)
                            (arena=2706KB #4127)
JIT的代码缓存，12045个类JIT编译后代码缓存整体使用79M内存。
-                      Code (reserved=263071KB, committed=79903KB)
                            (malloc=13471KB #15191)
                            (mmap: reserved=249600KB, committed=66432KB)
GC相关使用到的一些堆外内存，比如GC算法的处理锁会使用一些堆外空间。118M左右。
-                        GC (reserved=118432KB, committed=118432KB)
                            (malloc=93848KB #453)
                            (mmap: reserved=24584KB, committed=24584KB)
JAVA编译器自身操作使用到的一些堆外内存，很少。
-                  Compiler (reserved=975KB, committed=975KB)
                            (malloc=844KB #1074)
                            (arena=131KB #3)
Internal：memory used by the command line parser, JVMTI, properties等。
-                  Internal (reserved=117158KB, committed=117158KB)
                            (malloc=117126KB #44857)
                            (mmap: reserved=32KB, committed=32KB)
Symbol：保留字符串（Interned String）的引用与符号表引用放在这里，17M左右
-                    Symbol (reserved=17133KB, committed=17133KB)
                            (malloc=13354KB #145640)
                            (arena=3780KB #1)
NMT本身占用的堆外内存，4M左右
-    Native Memory Tracking (reserved=4402KB, committed=4402KB)
                            (malloc=396KB #5287)
                            (tracking overhead=4006KB)
不知道啥，用的很少。
-               Arena Chunk (reserved=272KB, committed=272KB)
                            (malloc=272KB)
其他未分类的堆外内存占用，100M左右。
-                   Unknown (reserved=99336KB, committed=99336KB)
                            (mmap: reserved=99336KB, committed=99336KB)
```
* 保留内存（reserved）：reserved memory 是指JVM 通过mmaped PROT_NONE 申请的虚拟地址空间，在页表中已经存在了记录（entries），保证了其他进程不会被占用，且保证了逻辑地址的连续性，能简化指针运算。

* 提交内存（commited）：committed memory 是JVM向操做系统实际分配的内存（malloc/mmap）,mmaped PROT_READ | PROT\_WRITE,仍然会page faults，但是跟 reserved 不同，完全内核处理像什么也没发生一样。

**这里需要注意的是：由于malloc/mmap的lazy allocation and paging机制，即使是commited的内存，也不一定会真正分配物理内存。**

```
malloc/mmap is lazy unless told otherwise. Pages are only backed by physical memory once they're accessed.
```

*Tips：由于内存是一直在缓慢增长，因此在使用NMT跟踪堆外内存时，一个比较好的办法是，先建立一个内存使用基线，一段时间后再用当时数据和基线进行差别比较，这样比较容易定位问题。*

```
jcmd 11 VM.native_memory baseline
```
同时pmap看一下物理内存的分配，RSS占用了10G。

```
pmap -x 11 | sort -n -k3
```
![](/images/15332046869086.jpg)￼

运行一段时间后，做一下summary级别的diff，看下内存变化，同时再次pmap看下RSS增长情况。

```
jcmd 11 VM.native_memory summary.diff
Native Memory Tracking:

Total: reserved=13089769KB +112323KB, committed=11877285KB +117915KB

-                 Java Heap (reserved=8388608KB, committed=8388608KB)
                            (mmap: reserved=8388608KB, committed=8388608KB)

-                     Class (reserved=1126527KB +2161KB, committed=85771KB +2033KB)
                            (classes #12682 +154)
                            (malloc=2175KB +113KB #37289 +2205)
                            (mmap: reserved=1124352KB +2048KB, committed=83596KB +1920KB)

-                    Thread (reserved=2861485KB +94989KB, committed=2861485KB +94989KB)
                            (thread #2772 +92)
                            (stack: reserved=2848588KB +94576KB, committed=2848588KB +94576KB)
                            (malloc=9169KB +305KB #13881 +460)
                            (arena=3728KB +108 #5543 +184)

-                      Code (reserved=265858KB +1146KB, committed=94130KB +6866KB)
                            (malloc=16258KB +1146KB #18187 +1146)
                            (mmap: reserved=249600KB, committed=77872KB +5720KB)

-                        GC (reserved=118433KB +1KB, committed=118433KB +1KB)
                            (malloc=93849KB +1KB #487 +24)
                            (mmap: reserved=24584KB, committed=24584KB)

-                  Compiler (reserved=1956KB +253KB, committed=1956KB +253KB)
                            (malloc=1826KB +253KB #2098 +271)
                            (arena=131KB #3)

-                  Internal (reserved=203932KB +13143KB, committed=203932KB +13143KB)
                            (malloc=203900KB +13143KB #62342 +3942)
                            (mmap: reserved=32KB, committed=32KB)

-                    Symbol (reserved=17820KB +108KB, committed=17820KB +108KB)
                            (malloc=13977KB +76KB #152204 +257)
                            (arena=3844KB +32 #1)

-    Native Memory Tracking (reserved=5519KB +517KB, committed=5519KB +517KB)
                            (malloc=797KB +325KB #9992 +3789)
                            (tracking overhead=4722KB +192KB)

-               Arena Chunk (reserved=294KB +5KB, committed=294KB +5KB)
                            (malloc=294KB +5KB)

-                   Unknown (reserved=99336KB, committed=99336KB)
                            (mmap: reserved=99336KB, committed=99336KB
```
![](/images/15332669357421.jpg)￼

发现这段时间pmap看到的RSS增长了3G多，但NMT观察到的内存增长了不到120M，还有大概2G多常驻内存不知去向，因此也基本排除了由于JVM自身管理的堆外内存的嫌疑。

#### 排查Metaspace元空间的堆外内存占用
由于线上使用的是JDK8，前面提到，JDK8里的元空间实际上使用的也是堆外内存，默认没有设置元空间大小的情况下，元空间最大堆外内存大小和Xmx是一致的。JMC连上后看下内存tab下metaspace一栏的内存占用情况，发现元空间只占用不到80M内存，也排除了它的可能性。实在不放心的话可以通过-XX:MaxMetaspaceSize设置元空间使用堆外内存的上限。
![](/images/15332080269604.jpg)￼

#### gdb分析内存块内容
上面提到使用pmap来查看进程的内存映射，pmap命令实际是读取了/proc/pid/maps和/porc/pid/smaps文件来输出。发现一个细节，pmap取出的内存映射发现很多64M大小的内存块。这种内存块逐渐变多且占用的RSS常驻内存也逐渐增长到reserved保留内存大小，内存增长的2G多基本上也是由于这些64M的内存块导致的，因此看一下这些内存块里具体内容。

##### strace挂上监控下内存分配和回收的系统调用：

```
strace -o /data1/weibo/logs/strace_output2.txt -T -tt -e mmap,munmap,mprotect -fp 12
```
看内存申请和释放的情况：

```
cat ../logs/strace_output2.txt | grep mprotect | grep -v resumed | awk '{print int($4)}' | sort -rn | head -5

cat ../logs/strace_output2.txt | grep mmap | grep -v resumed | awk '{print int($4)}' | sort -rn | head -5

cat ../logs/strace_output2.txt | grep munmap | grep -v resumed | awk '{print int($4)}' | sort -rn | head -5
```
配合pmap -x 10看一下实际内存分配情况：
![](/images/15332094618358.jpg)￼

找一块内存块进行dump：

```
gdb --batch --pid 11 -ex "dump memory a.dump 0x7fd488000000 0x7fd488000000+56124000"
```
简单分析一下内容，发现绝大部分是乱码的二进制内容，看不出什么问题。
strings a.dump | less
或者： hexdump -C a.dump | less
或者： view a.dump

没啥思路的时候，随便搜了一下发现貌似很多人碰到这种64M内存块的问题([比如这里](https://yq.aliyun.com/articles/227924))，了解到glibc的内存分配策略在高版本有较大调整：

"从glibc 2.11（为应用系统在多核心CPU和多Sockets环境中高伸缩性提供了一个动态内存分配的特性增强）版本开始引入了per thread arena内存池，Native Heap区被打散为sub-pools ，这部分内存池叫做Arena内存池。也就是说，以前只有一个main arena，目前是一个main arena（还是位于Native Heap区） + 多个per thread arena，多个线程之间不再共用一个arena内存区域了，保证每个线程都有一个堆，这样避免内存分配时需要额外的锁来降低性能。main arena主要通过brk/sbrk系统调用去管理，per thread arena主要通过mmap系统调用去分配和管理。"

*<font color=red>"一个32位的应用程序进程，最大可创建 2 CPU总核数个arena内存池（MALLOC\_ARENA\_MAX），每个arena内存池大小为1MB，一个64位的应用程序进程，最大可创建 8 CPU总核数个arena内存池（MALLOC\_ARENA\_MAX），每个arena内存池大小为64MB"</font>*

##### ptmalloc2内存分配和释放
"当某一线程需要调用 malloc()分配内存空间时， 该线程先查看线程私有变量中是否已经存在一个分配区，如果存在， 尝试对该分配区加锁，如果加锁成功，使用该分配区分配内存，如果失败， 该线程搜索循环链表试图获得一个没有加锁的分配区。如果所有的分配区都已经加锁，那么 malloc()会开辟一个新的分配区，把该分配区加入到全局分配区循环链表并加锁，然后使用该分配区进行分配内存操作。在释放操作中，线程同样试图获得待释放内存块所在分配区的锁，如果该分配区正在被别的线程使用，则需要等待直到其他线程释放该分配区的互斥锁之后才可以进行释放操作。<font color=red>用户 free 掉的内存并不是都会马上归还给系统，ptmalloc2 会统一管理 heap 和 mmap 映射区域中的空闲的chunk，当用户进行下一次分配请求时， ptmalloc2 会首先试图在空闲的chunk 中挑选一块给用户，这样就避免了频繁的系统调用，降低了内存分配的开销</font>。"

##### ptmalloc2的内存收缩机制
"业务层调用free方法释放内存时，ptmalloc2先判断 top chunk 的大小是否大于 mmap 收缩阈值(默认为 128KB)，如果是的话，对于主分配区，则会试图归还 top chunk 中的一部分给操作系统。但是最先分配的 128KB 空间是不会归还的，ptmalloc 会一直管理这部分内存，用于响应用户的分配 请求;如果为非主分配区，会进行 sub-heap 收缩，将 top chunk 的一部分返回给操 作系统，如果 top chunk 为整个 sub-heap，会把整个 sub-heap 还回给操作系统。做 完这一步之后，释放结束，从 free() 函数退出。可以看出，收缩堆的条件是当前 free 的 chunk 大小加上前后能合并 chunk 的大小大于 64k，并且要 top chunk 的大 小要达到 mmap 收缩阈值，才有可能收缩堆。"

##### ptmalloc2的mmap分配阈值动态调整
"M_MMAP_THRESHOLD 用于设置 mmap 分配阈值，默认值为 128KB，ptmalloc 默认开启 动态调整 mmap 分配阈值和 mmap 收缩阈值。当用户需要分配的内存大于 mmap 分配阈值，ptmalloc 的 malloc()函数其实相当于 mmap() 的简单封装，free 函数相当于 munmap()的简单封装。相当于直接通过系统调用分配内存， 回收的内存就直接返回给操作系统了。因为这些大块内存不能被 ptmalloc 缓存管理，不能重用，所以 ptmalloc 也只有在万不得已的情况下才使用该方式分配内存。"

##### 业务特性和ptmalloc2内存分配的gap
当前业务并发较大，线程较多，内存申请时容易造成锁冲突申请多个arena，另外该服务涉及到图片的上传和处理，底层会比较频繁的通过JNI调用ImageIO的图片读取方法（com_sun_imageio_plugins_jpeg_JPEGImageReader_readImage），经常会向glibc申请10M以上的buffer内存，考虑到ptmalloc2的lazy回收机制和mmap分配阈值动态调整默认打开，对于这些申请的大内存块，使用完后仍然会停留在arena中不会归还，同时也比较难得到收缩的机会去释放（当前回收的chunk和top chunk相邻，且合并后大于64K）。因此在这种较高并发的多线程业务场景下，RES的增长也是不可避免。

##### 如何优化解决
##### 三种方案：
**第一种：**控制分配区的总数上限。默认64位系统分配区数为：cpu核数*8，如当前环境16核系统分配区数为128个，每个64M上限的话最多可达8G，限制上限后，后续不够的申请会直接走mmap分配和munmap回收，不会进入ptmalloc2的buffer池。
所以第一种方案调整一下分配池上限个数到4： 

```
export MALLOC_ARENA_MAX=4
```
**第二种：**之前降到ptmalloc2默认会动态调整mmap分配阈值，因此对于较大的内存请求也会进入ptmalloc2的内存buffer池里，这里可以去掉ptmalloc的动态调整功能。可以设置 M_TRIM_THRESHOLD，M_MMAP_THRESHOLD，M_TOP_PAD 和 M_MMAP_MAX 中的任意一个。这里可以固定分配阈值为128K，这样超过128K的内存分配请求都不会进入ptmalloc的buffer池而是直接走mmap分配和munmap回收（性能上会有损耗，当前环境大概10%）。：

```
export MALLOC_MMAP_THRESHOLD_=131072
export MALLOC_TRIM_THRESHOLD_=131072
export MALLOC_TOP_PAD_=131072
export MALLOC_MMAP_MAX_=65536   
```
**第三种：**使用tcmalloc来替代默认的ptmalloc2。google的tcmalloc提供更优的内存分配效率，性能更好，ThreadCache会阶段性的回收内存到CentralCache里。 解决了ptmalloc2中arena之间不能迁移导致内存浪费的问题。

##### tcmalloc安装使用
###### 1.实现原理
perf-tools实现原理是：在java应用程序运行时，当系统分配内存时调用malloc时换用它的libtcmalloc.so，也就是TCMalloc会自动替换掉glibc默认的malloc和free，这样就能做一些统计。使用TCMalloc（Thread-Caching Malloc）与标准的glibc库的malloc相比，TCMalloc在内存的分配上效率和速度要高，[==了解更多TCMalloc](http://shiningray.cn/tcmalloc-thread-caching-malloc.html)

###### 2. 安装和使用
###### 2.1 前置工具的安装

```
yum -y install gcc make
yum -y install gcc gcc-c++
yum -y perl
```
###### 2.2 libunwind
使用perf-tools的TCMalloc，在64bit系统上需要先安装libunwind（http://download.savannah.gnu.org/releases/libunwind/libunwind-1.2.tar.gz，只能是这个版本），这个库为基于64位CPU和操作系统的程序提供了基本的堆栈辗转开解功能，其中包括用于输出堆栈跟踪的API、用于以编程方式辗转开解堆栈的API以及支持C++异常处理机制的API，32bit系统不需安装。

```
tar zxvf libunwind-1.2.tar.gz
./configure
make
make install
make clean
```
###### 2.3 perf-tools
从https://github.com/gperftools/gperftools下载相应的google-perftools版本。

```
tar zxvf google-perftools-2.7.tar.gz
./configure
make
make install
make clean
#修改lc_config,加入/usr/local/lib(libunwind的lib所在目录)
echo "/usr/local/lib" > /etc/ld.so.conf.d/usr_local_lib.conf 
#使libunwind生效
ldconfig
```
###### 2.3.1 关于etc/ld.so.conf
这个文件记录了编译时使用的动态链接库的路径。默认情况下，编译器只会使用/lib和/usr/lib这两个目录下的库文件。
如果你安装了某些库，比如在安装gtk+-2.4.13时它会需要glib-2.0 >= 2.4.0,辛苦的安装好glib后没有指定 --prefix=/usr 这样glib库就装到了/usr/local下，而又没有在/etc/ld.so.conf中添加/usr/local/lib。
库文件的路径如 /usr/lib 或 /usr/local/lib 应该在 /etc/ld.so.conf 文件中，这样 ldd 才能找到这个库。在检查了这一点后，要以 root 的身份运行 /sbin/ldconfig。
将/usr/local/lib加入到/etc/ld.so.conf中，这样安装gtk时就会去搜索/usr/local/lib,同样可以找到需要的库
###### 2.3.2 关于ldconfig
ldconfig的作用就是将/etc/ld.so.conf列出的路径下的库文件 缓存到/etc/ld.so.cache 以供使用
因此当安装完一些库文件，(例如刚安装好glib)，或者修改ld.so.conf增加新的库路径后，需要运行一下/sbin/ldconfig
使所有的库文件都被缓存到ld.so.cache中，如果没做，即使库文件明明就在/usr/lib下的，也是不会被使用的

###### 2.4 为perf-tools添加线程目录
```
mkdir /data1/weibo/logs/gperftools/tcmalloc/heap
chmod 0777 /data1/weibo/logs/gperftools/tcmalloc/heap
```

###### 2.5 修改tomcat启动脚本
catalina.sh里添加：

```
ldconfig
export LD_PRELOAD=/usr/local/lib/libtcmalloc.so
export HEAPPROFILE=/data1/weibo/logs/gperftools/tcmalloc/heap
```
修改后重启tomcat的容器。

###### 2.5.1 关于LD_PRELOAD
LD_PRELOAD是Linux系统的一个环境变量，它可以影响程序的运行时的链接（Runtime linker），它允许你定义在程序运行前优先加载的动态链接库。这个功能主要就是用来有选择性的载入不同动态链接库中的相同函数。通过这个环境变量，我们可以在主程序和其动态链接库的中间加载别的动态链接库，甚至覆盖正常的函数库。一方面，我们可以以此功能来使用自己的或是更好的函数（无需别人的源码），而另一方面，我们也可以以向别人的程序注入程序，从而达到特定的目的。[更多关于LD_PRELOAD](http://www.cnblogs.com/net66/p/5609026.html)


**<font color=red>经验证上面三种方式都能有效解决常驻内存持续增长的问题。</font>**


参考：
http://www.infoq.com/cn/articles/Java-PERMGEN-Removed
https://docs.oracle.com/javase/specs/jvms/se8/html/index.html
https://docs.oracle.com/javase/8/docs/technotes/guides/vm/nmt-8.html
https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr022.html#BABCBGFA
http://zhanjindong.com/2016/03/02/jvm-memory-tunning-notes
http://www.cnblogs.com/duanxz/archive/2012/08/09/2630284.html
https://stackoverflow.com/questions/31173374/why-does-a-jvm-report-more-committed-memory-than-the-linux-process-resident-set
https://siddhesh.in/posts/malloc-per-thread-arenas-in-glibc.html
https://www.sourceware.org/systemtap/SystemTap_Beginners_Guide/using-systemtap.html
http://debuginfo.centos.org/7/x86_64/


    


