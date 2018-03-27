---
title: "问题排查利器之-JVM动态追踪工具BTrace"
date: 2018-03-20T14:18:25+08:00
categories: ["技术"]
tags: ["问题排查","Java"]
thumbnail: "images/xiaodongjiangchenwu.jpg"
draft: false
---
很多时候怀疑线上运行的代码有问题但又苦于日志打的不够详细的时候，BTrace作为基于ASM实现的代码动态跟踪工具能很快排上用场，在不需要重启java进程的情况下，通过脚本动态切入到嫌疑代码块，快速定位问题。<!--more-->

#### 实现原理：
Attach API + BTrace脚本解析引擎 + ASM + JDK Instumentation。

#### 使用BTrace的典型场景：
1. 没打日志，想知道我这个方法的调用参数和返回结果？
2. 某个方法变慢，想知道该接口方法调用的耗时或者该方法内调用的多个子方法的耗时？
3. 某个方法抛异常时的传入参数？
4. 谁调用了某个方法，具体调用链？

#### BTrace下载地址：
http://github.com/btraceio/btrace 可以直接下载它的release包，开箱即用。

#### BTrace简单用法：
```
sh ../bin/btrace -u java进程ID XXX.java
```
如果要把输出结果写到文件，可以：
```
sh ../bin/btrace -u -o btrace.log java进程ID XXX.java
```
-o 这个参数会把文件输出到被trace的java进程的启动目录，而不是当前目录。而且更坑的是：执行过一次-o之后，再执行btrace不加-o 也不会再输出回console，直到应用重启。所以还不如直接重定向到文件：
```
sh ../bin/btrace -u java进程ID XXX.java > btrace.log
```

如果脚本里使用到了jdk外的类，需要通过classpath传入依赖，注意这里的cp只是供BTrace编译时使用，真正运行的时候由于已经Attach到JVM里，所以用的是目标JVM的classpath：
```
sh ../bin/btrace -u -o btrace.log -cp .:guava-18.0.jar java进程ID XXX.java
```
#### 知识点-拦截时机：
Btrace里拦截时机以下几种，通过Location关键字来定义：具体是：
Kind.Entry：默认是Entry，就是方法刚进入时拦截。 
Kind.Return：如果想获得函数的返回结果或执行时间，需要定义为：Kind.Return
Kind.CALL：分析方法中调用其它方法的执行情况，有类似分布耗时的功效
Kind.LINE：设置line，监控代码是否执行到指定行的位置
Kind.Throw：异常抛出时拦截。
Kind.Catch：异常被捕获时拦截。
Kind.Error:异常没被捕获被抛出方法外时拦截。
在拦截函数的参数定义里注入一个Throwable的参数，代表异常。
public static void onBind(Throwable exception, @Duration long duration)

#### 知识点-预编译：
BTrace可以实时编译.java的脚本文件，但可以通过btacec预编译来避免执行时才发现有问题。
```
sh ../bin/btracec XXX.java
```

#### 特别注意：
BTrace为了安全考虑，trace脚本诸多限制，比如：
1. 不允许调用任何类的任何方法。只能调用BTraceUtils 里的工具方法和脚本里定义的static方法
2. 不允许创建对象
3. 不允许for循环

除非在脚本注解里使用 @BTrace(unsafe=true)，同时运行时使用 -u 参数 来规避限制。


#### <font color=green>示例：查看某个方法的调用耗时：</font>
注意：这里的Duration是纳秒，需要转换一下比较好看。另外，可以用fqn参数打印出方法全路径名。

```
import com.sun.btrace.annotations.*;
import static com.sun.btrace.BTraceUtils.*;

@BTrace(unsafe=true)
public class GetImageCost {
    @OnMethod(clazz="cn.sina.as.service.impl.AttachStorageServiceImpl", method="getAttachStorageFromNormal", location = @Location(Kind.RETURN))
    public static void printGetImageTime(@ProbeClassName String probeClass, @ProbeMethodName(fqn = true) String probeMethod,  @Duration long duration) {
        println("=======ProbeClass:" + probeClass +  ",probeMethod:" + probeMethod + ",duration:" + (duration / 100000) + "ms");
    }
}
```
#### <font color=green>示例：获取某个方法里所有调用子方法里耗时大于100ms的：</font>
```
import com.sun.btrace.annotations.*;
import static com.sun.btrace.BTraceUtils.*;

@BTrace
public class PrintGenerateThumbPicCost {
    @OnMethod(clazz="cn.sina.as.service.impl.AttachStorageServiceImpl", method="getImage",location = @Location(value = Kind.CALL, clazz = "/.*/", method = "/.*/", where = Where.AFTER))
    public static void printCompressedImageTime(@Self Object self, @TargetInstance Object instance, @TargetMethodOrField(fqn = true) String method,  @Duration long duration) {
    if (duration > 100*100000) {
        println("=======invoke method:" + method + ",duration:" + (duration /100000) + "ms");
        }
    }
}
```
#### <font color=green>示例：获取某个方法执行时的线程栈：</font>
```
import com.sun.btrace.annotations.*;
import static com.sun.btrace.BTraceUtils.*;

@BTrace
public class PrintThreadStack {
    @OnMethod(clazz="cn.sina.as.service.impl.AttachStorageServiceImpl", method="getImage")
    public static void printThreadStack() {
        println("thread dump where getImage invoked!");
        jstack();
    }
}
```

#### <font color=green>示例：获取某个方法的入参和返回</font>
sh ../bin/btrace -u -cp .:attachment-common-1.1.16.jar 9 PrintParamAndReturn.java
当前对象的引用，参数列表，返回值，这3个是按顺序定义用@Self 注释的this， 完整的参数列表，以及用@Return 注释的返回值。

```
import com.sun.btrace.annotations.*;
import static com.sun.btrace.BTraceUtils.*;
import java.lang.reflect.Field;
import cn.sina.as.constant.ImageType;
import cn.sina.as.model.AttachStorage;
import com.sun.btrace.AnyType;
import cn.sina.as.model.AuthUser;
import cn.sina.as.model.DownloadContext;

@BTrace(unsafe=true)
public class PrintParamAndReturn {

    @OnMethod(clazz="cn.sina.as.service.impl.AttachStorageServiceImpl", method="getAttachStorageFromNormal", location = @Location(Kind.RETURN))
    public static void printParamAndReturn(@Self Object self, long fid, ImageType imageType, AuthUser user, DownloadContext context,  @Return AnyType result) {
        println("method params: " + fid);
        println("method params: " + user.getUid());
        printFields(result);
        println("method return: " + str(get(field("cn.sina.as.model.AttachStorage", "filename"), result)));
    }
}
```
注意：参数列表要么都不要，要么都不能缺，否则会进入静默状态。
另外，某些场景下，我们只需要追踪某一两个简单的参数，不需要其他非JDK类型的参数，这是可以使用AnyType来定义非JDK类型的参数，从而不需要引入其他第三方依赖。（另外，不同参数的多个同名方法可以用AnyType[] 数组来匹配和打印出具体参数值，不过不知道哪里有问题没调通。)
sh ../bin/btrace 9 PrintParamAndReturn.java

```
import com.sun.btrace.annotations.*;
import static com.sun.btrace.BTraceUtils.*;
import java.lang.reflect.Field;
import com.sun.btrace.AnyType;

@BTrace
public class PrintParamAndReturn {

    @OnMethod(clazz="cn.sina.as.service.impl.AttachStorageServiceImpl", method="getAttachStorageFromNormal", location = @Location(Kind.RETURN))
    public static void printParamAndReturn(@Self Object self, long fid, AnyType imageType, AnyType authUser, AnyType downloadContext,  @Return AnyType result) {
        println("method params: " + fid);
        printFields(result);
    }
}
```

#### <font color=green>示例：每10s打印一次某个方法被调用的总次数和慢查询次数。</font>

使用@OnTimer来定时触发，结合一些工具类。sh ../bin/btrace 9 PrintPerfCount.java

```
import com.sun.btrace.annotations.*;
import static com.sun.btrace.BTraceUtils.*;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.Map;

@BTrace
public class PrintPerfCount {

    private static Map<String, AtomicInteger> histo = newHashMap();

    @OnMethod(clazz="cn.sina.as.service.impl.AttachStorageServiceImpl", method="getAttachStorageFromNormal", location = @Location(Kind.RETURN))
    public static void printInvokeCount(@ProbeClassName String probeClass, @ProbeMethodName(fqn = true) String probeMethod,  @Duration long duration) {
        if((duration / 100000) > 1000) {
            AtomicInteger slowCount = get(histo, "slowCount");
            if (slowCount == null) {
                slowCount = newAtomicInteger(1);
                put(histo, "slowCount", slowCount);
            } else {
                incrementAndGet(slowCount);
            }
        }

        AtomicInteger totalCount = get(histo, "totalCount");
        if (totalCount == null) {
            totalCount = newAtomicInteger(1);
            put(histo, "totalCount", totalCount);
        } else {
            incrementAndGet(totalCount);
        }
    }

    @OnTimer(1000 * 10)
    public static void count() {
        if (Collections.size(histo) != 0) {
           printNumberMap("Invoke TotalCount And SlowCount:", histo);
        }
    }
}
```

#### 其他用法：
按接口，父类，Annotation定位：
比如我想匹配所有的Filter类，在接口或基类的名称前面，加个+ 就行
@OnMethod(clazz="+com.vip.demo.Filter", method="doFilter")

#### 其他用法：
也可以按类或方法上的annotaiton匹配，前面加上@就行
@OnMethod(clazz="@javax.jws.WebService", method="@javax.jws.WebMethod")

#### 其他用法：
拦截构造函数，构造函数的名字是 <init>
@OnMethod(clazz="java.net.ServerSocket", method="<init>")

#### 其他用法：
拦截静态内部类的写法，是在类与内部类之间加上"$"，​静态函数中，instance的值为空。如果想获得执行时间，必须把Where定义成AFTER。如果想获得执行时间，必须 把Where定义成AFTER。
@OnMethod(clazz="com.vip.MyServer$MyInnerClass", method="hello")

*****
<font color=gray>参考：
白衣大大的详细介绍：http://calvin1978.blogcn.com/articles/btrace1.html
Instrumentation介绍： http://www.ibm.com/developerworks/cn/java/j-lo-jse61/
BTrace使用小结：https://lfckop.github.io/btrace/
阿里出品的类似工具greys-anatomy：https://github.com/oldmanpushcart/greys-anatomy
</font>
