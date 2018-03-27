---
title: "服务稳定性提升之-熔断组件Hystrix"
date: 2018-03-21T15:43:14+08:00
categories: ["技术"]
tags: ["netflix","Java","服务稳定性","开源组件"]
thumbnail: "images/qieyifengjing.jpg"
draft: false
---
Hystrix是netflix公司开源的一个通用系统保护框架，提供服务对远程依赖的快速失败机制、requestCache支持、请求合并能力等。目前通讯这边已经大范围使用，并结合profile日志进行了一些简单的定制化，目前使用情况看，实用性和稳定性比较出色，接入成本也较低。有兴趣的组可以参考以下范例尝试接入使用。<!--more-->

### Hystrix的主要能力栈如下：

* 提供方法级别的Fail Fast的自动短路保护机制,设置时间内频繁出错或超时会自动短路该方法的调用。
* 提供熔断机制触发后的备选方法切入。例如rpc不可用时可自动切换成jar方式等。
* 提供并发访问控制。
* 提供超时控制。
* 提供平均访问耗时、qps、p99等详细的metrics数据，并已接入平台profile日志。
* 提供一定程度的requestCache支持，可酌情作为LocalCache使用。
* 提供有限的自动合并请求能力的支持，可根据业务场景酌情使用。

### How It Work?

* 如果在一定的窗口时间内(默认500ms)，出错比例达到多少(默认50%，前提还有需要满足一次的访问次数【默认20】才行)，该方法就进入短路状态。
* 短路状态下，所有访问都会立即返回或者走配置的对应的降级方法，不会真正访问远程资源。保护窗口期(默认5秒)过了之后，会再尝试放一次请求进入，如果依然错误，则继续保护状态，如果成功则放入请求。
* 独立线程池模式和信号量模式都支持超时控制，超时默认为1秒，超时立刻返回异常并算入异常计数。<font color=red>但需要注意，由于信号量模式下的超时控制是通过Hystrix本身提供的Timer来实现的，所以超时控制并不是很精确。</font>线程池大小默认为10，如果满了，同样立刻返回错误并记入异常计数。
_详细请参考：[Hystrix工作方式](https://github.com/Netflix/Hystrix/wiki/How-it-Works#flow-chart)_

### 演示示例
最佳实践中提供了hystrix的多种使用场景的演示，见failfast-hystrix模块。
该模块是一个基于spring boot搭建的restful风格的应用，模拟一个查询消息的服务。通过模块源码根目录下的Main可以启动服务，启动完毕后，通过本机8080端口配合apache benchmark进行演示。
service层的MessageService增加了基于注解@HystrixCommand方式的保护机制；
controller层MessageController作为请求入口，完成了多种调用示例的实现;
#### 具体详见代码。

* Demo1： ab -c 1 -n 100 "http://localhost:8080/msg/1/timeout"
模拟多并发访问消息ID为1的请求，该接口内部通过sleep超时响应，可以观察到所以请求在1s超时后进入熔断状态，其他后续请求直接进入fallback降级。

* Demo2： ab -c 1 -n 100 "http://localhost:8080/msg/2/exception"   ab -c 1 -n 100 "http://localhost:8080/msg/1/exception"
模拟两种请求，访问ID为1的请求,内部会抛出"非法参数异常",该异常在"屏蔽的白名单"中,因此不会进入熔断异常计数;
第二种请求,访问ID为2的消息,内部会抛出"运行时异常",该异常会计入熔断计数, 可以观察到所以请求在经过大概默认的500ms健康检查期后进入熔断状态。*

* Demo3： curl -s "http://localhost:8080/msg/1,2/ratelimiter"
模拟访问消息ID为1，2的请求，该接口限制了并发访问限制为1，可以观察到该请求只有一个可以成功，另一个进入fallback。*

* Demo4： curl -s "http://localhost:8080/msg/1/async"
模拟访问消息ID为1的请求，该接口内部为异步模式，适用于某些已经是异步访问的调用方式。*

* Demo5： curl -s "http://localhost:8080/msg/1/cache"
模拟访问消息ID为1的请求，在同一http请求对同一ID访问两次，实际底层是调用了一次。*


* Demo6： curl -s "http://localhost:8080/msg/1,2/collapse"
模拟合并请求的场景，该请求通过单查询接口逐条查询消息，在service层通过hystrix合并两个请求成一个多查询接口请求，只需要访问db一次。*



### 日志信息和实时监控dashboard
* 对于实时metrics数据，hystrix本身提供良好的访问接口，本模块中只是进行了简单的提取和封装并打印到日志文件中，目前默认是存放在profile日志中。通过将profile日志推送到平台日志系统，可以随时查看被hystrix保护的接口的历史和实时情况等。
* hystrix本身提供一个实时的监控dashboard，这是一个独立的war，需要单独部署，可实时展示被保护的方法的访问情况、熔断与否等。
	* 如果需要连接多台机器的实时数据，还需要通过turbine来进行聚合。
	* 实时dashboard需要在每台服务机上开启一个servlet( [hystrix-metrics-event-stream](https://github.com/Netflix/Hystrix/tree/master/hystrix-contrib/hystrix-metrics-event-stream))用于提供给dashboard程序定时拉取metrics数据。

![](https://github.com/Netflix/Hystrix/wiki/images/dashboard-annoted-circuit-640.png)

_具体使用情况请移步：[Hystrix实时dashboard](https://github.com/Netflix/Hystrix/wiki/Dashboard)_

### 一些坑
* @HystrixCommand注解的一些约束:
@HystrixCommand does NOT work, if the original call is made to the service method without direct @HystrixCommand. For example: the original call => service() => doSomething(). Here, service() does NOT has @HystrixCommand, but it calls another method doSomething(), which has @HystrixCommand.

```
public class A {

    public void service() {
        ...
        doSomething();
        ...
    }

    @HystrixCommand
    public void doSomething() {
        ...
    }
}
```
On the other side, if the original call invokes doSomething() directly, then @HystrixCommand works well.

I believe @spencergibb gave a very good direction for this issue: it is all about AOP rather than @HystrixCommand self. I did a very quick search about AOP, and found a very good Posted on 26th July 2009 by Denis Zhdanov. Please check it out via the link:
Spring AOP top problem #1 - [aspects are not applied](http://denis-zhdanov.blogspot.com/2009/07/spring-aop-top-problem-1-aspects-are.html)
相关链接：[Why @HystrixCommand did not work?](https://github.com/Netflix/Hystrix/issues/1020)

### 其他参考
* 使用线程的payload。 参考：[cost-of-threads](https://github.com/Netflix/Hystrix/wiki/How-it-Works#cost-of-threads)

* 为啥默认不通过AOP实现。（PS：如果想用注解AOP方式，需要用到它的子项目javanica，当前示例模块就是通过这种方式使用的，并没有使用Command模式自己写代码）。参考： [why not use AOP?](https://github.com/Netflix/Hystrix/wiki/FAQ%20:%20General)

* 详细配置项，需细看。 [详细配置](https://github.com/Netflix/Hystrix/wiki/Configuration)


