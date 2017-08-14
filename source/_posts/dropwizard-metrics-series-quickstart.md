title: metrics系列之quickstart
date: 2017-01-16
tags:
 - metrics
categories:
 - 翻译文章
---


dropwizard/metrics

Metrics是一个Java库，它能在你的生产环境中，为你的代码中提供无与伦比的洞察力。

Metrics提供了一个强大的工具包，用于测量生产环境中关键组件的行为。

它提供了常见模块支持，如：Jetty，Logback，Log4j，Apache HttpClient，Ehcache，JDBI，Jersey和报告后端（如Ganglia和Graphite）的公共库的模块，Metrics为您提供全栈可见性。

# QuickStarted

下面是我们开始一个quickstart的步骤

## Setting Up Maven

需要加入maven依赖：
```xml
<dependency>
    <groupId>io.dropwizard.metrics</groupId>
    <artifactId>metrics-core</artifactId>
    <version>${metrics.version}</version>
</dependency>
```

> 注意：你要确保metrics.version属性已经在你的pom文件中被定义，当前的版本是3.1.0

好的，现在是时候开始为你的程序提供指标化的数据了

<!--more-->

## Meters

一个Meter负责测量一类事件的速率（比如：请求速率/ requests per sencond）, 它提供了1分钟，5分钟，15分钟的平均值

```java
private final Meter requests = metrics.meter("requests");

public void handleRequest(Request request, Response response) {
    requests.mark();
    // etc
}
```
这个meter将会测量请求速率，单位为requests per seconds

## Console Reporter

就如听起来的一样，它会将结果汇报给标准输出，这个报告会每隔1秒钟打印一次
```java
ConsoleReporter reporter = ConsoleReporter.forRegistry(metrics)
       .convertRatesTo(TimeUnit.SECONDS)
       .convertDurationsTo(TimeUnit.MILLISECONDS)
       .build();
   reporter.start(1, TimeUnit.SECONDS);
```

so，完整的示例看起来是这个样子的：
```java 
package sample;
  import com.codahale.metrics.*;
  import java.util.concurrent.TimeUnit;

  public class GetStarted {
    static final MetricRegistry metrics = new MetricRegistry();
    public static void main(String args[]) {
      startReport();
      Meter requests = metrics.meter("requests");
      requests.mark();
      wait5Seconds();
    }

  static void startReport() {
      ConsoleReporter reporter = ConsoleReporter.forRegistry(metrics)
          .convertRatesTo(TimeUnit.SECONDS)
          .convertDurationsTo(TimeUnit.MILLISECONDS)
          .build();
      reporter.start(1, TimeUnit.SECONDS);
  }

  static void wait5Seconds() {
      try {
          Thread.sleep(5*1000);
      }
      catch(InterruptedException e) {}
  }
}
```

pom文件的定义：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>somegroup</groupId>
  <artifactId>sample</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <name>Example project for Metrics</name>

  <properties>
     <metrics.version>3.1.0</metrics.version>
  </properties>

  <dependencies>
    <dependency>
      <groupId>io.dropwizard.metrics</groupId>
      <artifactId>metrics-core</artifactId>
      <version>${metrics.version}</version>
    </dependency>
  </dependencies>
</project>
```

运行:
```
mvn package exec:java -Dexec.mainClass=sample.GetStarted
```

## The Registry

MetricsRegistry是整个项目的核心类，它是所有测量的容量，可以通过如下方式创建它：
```java
final MetricRegistry metrics = new MetricRegistry();
```

你可能想把这个实例集成到你的应用生命周期中（比如使用你的依赖注入），这没有任何问题,但是建议还是静态实例化是一个最佳实践。

## Gauges

一个gauges代表了一个瞬时度量值，例如：我们想要度量一个队列中的pending jobs数据量：
```java
public class QueueManager {
    private final Queue queue;

    public QueueManager(MetricRegistry metrics, String name) {
        this.queue = new Queue();
        metrics.register(MetricRegistry.name(QueueManager.class, name, "size"),
                         new Gauge<Integer>() {
                             @Override
                             public Integer getValue() {
                                 return queue.size();
                             }
                         });
    }
}
```
当这个gauge被度量时，它将返回这个队列中的job数量

每个metric在注册表(registry)中都有一个唯一的名字，这个名字一般使用.号分隔，如：things.count，又或者:"com.example.Thing.latency"。MetricRegistry有一个静态的方法用于构造这些命名：
```java
MetricRegistry.name(QueueManager.class, "jobs", "size")
```
以上的例子会返回: "com.example.QueueManager.jobs.size"

对于大多数queue或者像queue一样的数据结构，通常不希望只返回队列的大小，大多数java.util和java.util.concurrent包的size()实现都是O(n)的复杂度，这表示你的gauge将会比较慢（潜在地可能存在加锁）

## Counter

一个Counter实际上也是一个gauge，它是关于AtomicLong的一个泛型实例。你可以自增或者自减它的值，例如，我们可以以不同的方式来度量队列中的pending job：
```java
private final Counter pendingJobs = metrics.counter(name(QueueManager.class, "pending-jobs"));

public void addJob(Job job) {
    pendingJobs.inc();
    queue.offer(job);
}

public Job takeJob() {
    pendingJobs.dec();
    return queue.take();
}
```

每次该counter被计算时，它将会返回这个队列中的job数量

就上面看到的，counters的API是不同的，#counter(String) 取代了#register(String,Metrics)，然而你也可以使用register，然后创建一个你自己的Counter实例，但是要说一下的是，#counter(String) 帮你做了这一切，并且允许你通过该名称进行引用复用。

## Histograms

Histogram是一个流式数据的统计分布，除了比如：最大值，最小值，平均值等，它还支持中位数，75%，90%，95%，98%，99%和99.9%的百分线
```java
private final Histogram responseSizes = metrics.histogram(name(RequestHandler.class, "response-sizes"));

public void handleRequest(Request request, Response response) {
    // etc
    responseSizes.update(response.getContent().length);
}
```
该histogram将会度量响应的大小

## Timer

一个Timer提供了某个程序的耗时和调用速率
```java
private final Timer responses = metrics.timer(name(RequestHandler.class, "responses"));

public String handleRequest(Request request, Response response) {
    final Timer.Context context = responses.time();
    try {
        // etc;
        return "OK";
    } finally {
        context.stop();
    }
}
```
该timer会度量该段代码的调用速率以及调用的时耗。

## Health Checks

Metrics也提供了通过metrics-healthchecks模块集中式健康检测的能力
首先，创建一个健康检测注册表的实例：
```java
final HealthCheckRegistry healthChecks = new HealthCheckRegistry();
```
其次，实现一个健康检测的子类:
```java
public class DatabaseHealthCheck extends HealthCheck {
    private final Database database;

    public DatabaseHealthCheck(Database database) {
        this.database = database;
    }

    @Override
    public HealthCheck.Result check() throws Exception {
        if (database.isConnected()) {
            return HealthCheck.Result.healthy();
        } else {
            return HealthCheck.Result.unhealthy("Cannot connect to " + database.getUrl());
        }
    }
}
```
最后注册它：
```java
healthChecks.register("postgres", new DatabaseHealthCheck(database));
```

运行健康检测：
```java
final Map<String, HealthCheck.Result> results = healthChecks.runHealthChecks();
for (Entry<String, HealthCheck.Result> entry : results.entrySet()) {
    if (entry.getValue().isHealthy()) {
        System.out.println(entry.getKey() + " is healthy");
    } else {
        System.err.println(entry.getKey() + " is UNHEALTHY: " + entry.getValue().getMessage());
        final Throwable e = entry.getValue().getError();
        if (e != null) {
            e.printStackTrace();
        }
    }
}
```

>Metrics comes with a pre-built health check: ThreadDeadlockHealthCheck, which uses Java’s built-in thread deadlock detection to determine if any threads are deadlocked.

## Reporting Via JMX
通过JMX汇报指标
```java
final JmxReporter reporter = JmxReporter.forRegistry(registry).build();
reporter.start();
```

当上面的reporter开启后，你就可以通过JConsole或者VisualVM进行MBean查看了：
{% asset_img metrics-visualvm.png %}

## Reporting Via HTTP

提供一个Servlet，该Servlet提供了所有metrics的json表现形式。它同样会运行健康检测，打印thread dump，提供简单的ping pong响应用于负载均衡。（当然也有分开的单独的Servlet提供单独的功能，例如：MetricsServlet, HealthCheckServlet, ThreadDumpServlet, and PingServlet）


如果需要使用到上面的东西，需要以下依赖：
```xml
<dependency>
    <groupId>io.dropwizard.metrics</groupId>
    <artifactId>metrics-servlets</artifactId>
    <version>${metrics.version}</version>
</dependency>
```

## Other Reporting

除了以上的JMX和HTTP Reporter以外，还有以下一些Reporter:

- STDOUT, using ConsoleReporter from metrics-core
- CSV files, using CsvReporter from metrics-core
- SLF4J loggers, using Slf4jReporter from metrics-core
- Ganglia, using GangliaReporter from metrics-ganglia
- Graphite, using GraphiteReporter from metrics-graphite



