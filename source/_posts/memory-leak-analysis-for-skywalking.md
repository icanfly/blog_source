---
title: 一次Skywalking内存泄露的原因分析
tags:
  - skywalking
  - apm
  - 问题解析
  - 参与开源
categories:
  - 原创文章
originContent: ''
toc: false
date: 2019-01-31 11:21:57
author:
thumbnail:
blogexcerpt:
---

# 什么是skywalking

Skywalking 是一款分布式系统的应用程序性能监视工具(APM)，专为微服务、云本机架构和基于容器（Docker、K8s、Mesos）架构而设计。

详细的Skywalking介绍见：[Skywalking官网](http://skywalkking.apache.org)

# 遇到的问题场景

1. 公司Dev/Test环境
2. Collector因故宕机很长时间，约两周（无人维护监控）
3. 应用接入端agent内存暴涨导致大量应用内存溢出或告警
4. Skywalking版本：5.0.0-GA
<!-- more -->

# 问题现象

```java
ERROR 2018-12-03 09:50:47:931 AppAndServiceRegisterClient :  AppAndServiceRegisterClient execute fail.
org.apache.skywalking.apm.dependencies.io.grpc.StatusRuntimeException: UNAVAILABLE: io exception
        at org.apache.skywalking.apm.dependencies.io.grpc.stub.ClientCalls.toStatusRuntimeException(ClientCalls.java:222)

```
另一些应用报错：
```java
java.lang.OutOfMemoryError: GC overhead limit exceeded

```

```java
ERROR 2019-01-20 20:44:21:024 JVMService :  send JVM metrics to Collector fail. 
org.apache.skywalking.apm.dependencies.io.grpc.StatusRuntimeException: UNAVAILABLE: io exception
        at org.apache.skywalking.apm.dependencies.io.grpc.stub.ClientCalls.toStatusRuntimeException(ClientCalls.java:222)
        at org.apache.skywalking.apm.dependencies.io.grpc.stub.ClientCalls.getUnchecked(ClientCalls.java:203)
        at org.apache.skywalking.apm.dependencies.io.grpc.stub.ClientCalls.blockingUnaryCall(ClientCalls.java:132)
        at org.apache.skywalking.apm.network.proto.JVMMetricsServiceGrpc$JVMMetricsServiceBlockingStub.collect(JVMMetricsServiceGrpc.java:158)
        at org.apache.skywalking.apm.agent.core.jvm.JVMService$Sender.run(JVMService.java:143)
        at org.apache.skywalking.apm.util.RunnableWithExceptionProtection.run(RunnableWithExceptionProtection.java:36)
        at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
        at java.util.concurrent.FutureTask.runAndReset(FutureTask.java:308)
        at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.access$301(ScheduledThreadPoolExecutor.java:180)
        at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:294)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
        at java.lang.Thread.run(Thread.java:748)
Caused by: org.apache.skywalking.apm.dependencies.io.netty.channel.AbstractChannel$AnnotatedConnectException: 拒绝连接: /172.21.16.175:11800
        at sun.nio.ch.SocketChannelImpl.checkConnect(Native Method)
        at sun.nio.ch.SocketChannelImpl.finishConnect(SocketChannelImpl.java:717)
        at org.apache.skywalking.apm.dependencies.io.netty.channel.socket.nio.NioSocketChannel.doFinishConnect(NioSocketChannel.java:325)
        at org.apache.skywalking.apm.dependencies.io.netty.channel.nio.AbstractNioChannel$AbstractNioUnsafe.finishConnect(AbstractNioChannel.java:340)
        at org.apache.skywalking.apm.dependencies.io.netty.channel.nio.NioEventLoop.processSelectedKey(NioEventLoop.java:634)
        at org.apache.skywalking.apm.dependencies.io.netty.channel.nio.NioEventLoop.processSelectedKeysOptimized(NioEventLoop.java:581)
        at org.apache.skywalking.apm.dependencies.io.netty.channel.nio.NioEventLoop.processSelectedKeys(NioEventLoop.java:498)
        at org.apache.skywalking.apm.dependencies.io.netty.channel.nio.NioEventLoop.run(NioEventLoop.java:460)
        at org.apache.skywalking.apm.dependencies.io.netty.util.concurrent.SingleThreadEventExecutor$5.run(SingleThreadEventExecutor.java:884)
        at org.apache.skywalking.apm.dependencies.io.netty.util.concurrent.FastThreadLocalRunnable.run(FastThreadLocalRunnable.java:30)
        ... 1 more
Caused by: java.net.ConnectException: 拒绝连接
        ... 11 more

```

这些日志大量出现在应用日志中，从发生内存溢出以前很久时间一直持续。

通过jmap将java内存对象状态dump出来分析：

![image.png](https://i.loli.net/2019/01/31/5c52660b1228b.png)

发现存在大量skywalking对象占据内存，具体对象为：HpackHeaderField和ManagedChannelImpl。

# 问题分析

HpackHeaderField和ManagedChannelImpl均为处理gRPC的处理类，并根据内存泄露时报的错误来看，Collector挂掉了很久，一直在重试连接。查看skywalking的源码分析重连过程：

GRPCChannelManager

```java
@Override
    public void run() {
        logger.debug("Selected collector grpc service running, reconnect:{}.", reconnect);
        if (reconnect) {
            if (RemoteDownstreamConfig.Collector.GRPC_SERVERS.size() > 0) {
                String server = "";
                try {
                    int index = Math.abs(random.nextInt()) % RemoteDownstreamConfig.Collector.GRPC_SERVERS.size();
                    server = RemoteDownstreamConfig.Collector.GRPC_SERVERS.get(index);
                    String[] ipAndPort = server.split(":");
                    managedChannel = GRPCChannel.newBuilder(ipAndPort[0], Integer.parseInt(ipAndPort[1]))
                        .addManagedChannelBuilder(new StandardChannelBuilder())
                        .addManagedChannelBuilder(new TLSChannelBuilder())
                        .addChannelDecorator(new AuthenticationDecorator())
                        .build();

                    if (!managedChannel.isShutdown() && !managedChannel.isTerminated()) {
                        reconnect = false;
                        notify(GRPCChannelStatus.CONNECTED);
                    } else {
                        notify(GRPCChannelStatus.DISCONNECT);
                    }
                    return;
                } catch (Throwable t) {
                    logger.error(t, "Create channel to {} fail.", server);
                    notify(GRPCChannelStatus.DISCONNECT);
                }
            }

            logger.debug("Selected collector grpc service is not available. Wait {} seconds to retry", Config.Collector.GRPC_CHANNEL_CHECK_INTERVAL);
        }
    }

```

从上面的代码可知，当collector挂掉后，agent在尝试重连而一直连接不上时，会不断的创建ManagedChannel对象，查看gRPC的ManagedChannelOrphanWrapper源码：

```java
final class ManagedChannelOrphanWrapper extends ForwardingManagedChannel {
  private static final ReferenceQueue<ManagedChannelOrphanWrapper> refqueue =
      new ReferenceQueue<ManagedChannelOrphanWrapper>();
  // Retain the References so they don't get GC'd
  private static final ConcurrentMap<ManagedChannelReference, ManagedChannelReference> refs =
      new ConcurrentHashMap<ManagedChannelReference, ManagedChannelReference>();
  private static final Logger logger =
      Logger.getLogger(ManagedChannelOrphanWrapper.class.getName());

  private final ManagedChannelReference phantom;

  ManagedChannelOrphanWrapper(ManagedChannel delegate) {
    this(delegate, refqueue, refs);
  }

... 此后代码省略

```
上面有一句话明确提示：Retain the References so they don't get GC'd
同时也初始化了一些Netty相关的处理类，并且没有释放。

翻看了一些gRPC的文档，也是建议一定要显式的关闭channel。

修改了agent部分的源码，处理为在重连时关闭旧的Channel对象：

```java
@Override
    public void run() {
        logger.debug("Selected collector grpc service running, reconnect:{}.", reconnect);
        if (reconnect) {
            if (RemoteDownstreamConfig.Collector.GRPC_SERVERS.size() > 0) {
                String server = "";
                GRPCChannel oldChannel = managedChannel;
                try {
                    int index = Math.abs(random.nextInt()) % RemoteDownstreamConfig.Collector.GRPC_SERVERS.size();
                    server = RemoteDownstreamConfig.Collector.GRPC_SERVERS.get(index);
                    String[] ipAndPort = server.split(":");
                    managedChannel = GRPCChannel.newBuilder(ipAndPort[0], Integer.parseInt(ipAndPort[1]))
                        .addManagedChannelBuilder(new StandardChannelBuilder())
                        .addManagedChannelBuilder(new TLSChannelBuilder())
                        .addChannelDecorator(new AuthenticationDecorator())
                        .build();

                    if (!managedChannel.isShutdown() && !managedChannel.isTerminated()) {
                        reconnect = false;
                        notify(GRPCChannelStatus.CONNECTED);
                    } else {
                        notify(GRPCChannelStatus.DISCONNECT);
                    }
                    return;
                } catch (Throwable t) {
                    logger.error(t, "Create channel to {} fail.", server);
                    notify(GRPCChannelStatus.DISCONNECT);
                } finally {
                    if (oldChannel != null) {
                        oldChannel.shutdownNow();
                    }
                }
            }

            logger.debug("Selected collector grpc service is not available. Wait {} seconds to retry", Config.Collector.GRPC_CHANNEL_CHECK_INTERVAL);
        }
    }

```
打包推送至Dev环境进行观察，同时也模拟Collector的情况将Collector杀死。经过一段时间的观察，内存平稳无异常。

到此skywalking的特定情况下的内存泄漏问题得到解决，相关issue已由同事提交到skywalking官方。