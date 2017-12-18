---
title: kafka顺序消息
date: 2017-01-17 12:49:43
tags:
- kafka
- 大数据
categories:
 - 原创文章
thumbnail:
---

顺序消息包括以下两方面：

- 全局顺序
- 局部顺序

## 全局顺序
全局顺序就目前的应用范围来讲，可以列举出来的也就限于binlog日志传输，如mysql binlog日志传输要求全局的顺序，不能有任何的乱序。这种的解决办法通常是最为保守的方式：
1. 全局使用一个生产者
2. 全局使用一个消费者（并严格到一个消费线程）
3. 全局使用一个分区（当然不同的表可以使用不同的分区或者topic实现隔离与扩展）

## 局部顺序
其实在大部分业务场景下，只需要保证消息局部有序即可，什么是局部有序？局部有序是指在某个业务功能场景下保证消息的发送和接收顺序是一致的。如：订单场景，要求订单的创建、付款、发货、收货、完成消息在同一订单下是有序发生的，即消费者在接收消息时需要保证在接收到订单发货前一定收到了订单创建和付款消息。

针对这种场景的处理思路是：针对部分消息有序（message.key相同的message要保证消费顺序）场景，可以在producer往kafka插入数据时控制，同一key分发到同一partition上面。因为每个partition是固定分配给某个消费者线程进行消费的，所以对于在同一个分区的消息来说，是严格有序的（在kafka 0.10.x以前的版本中，kafka因消费者重启或者宕机可能会导致分区的重新分配消费，可能会导致乱序的发生，0.10.x版本进行了优化，减少重新分配的可能性）。

## 注意事项

### 消息重试对顺序消息的影响

对于一个有着先后顺序的消息A、B，正常情况下应该是A先发送完成后再发送B，但是在异常情况下，在A发送失败的情况下，B发送成功，而A由于重试机制在B发送完成之后重试发送成功了。
这时对于本身顺序为AB的消息顺序变成了BA

### 消息producer发送逻辑的控制

消息producer在发送消息的时候，对于同一个broker连接是存在多个未确认的消息在同时发送的，也就是存在上面场景说到的情况，虽然A和B消息是顺序的，但是由于存在未知的确认关系，有可能存在A发送失败，B发送成功，A需要重试的时候顺序关系就变成了BA。简之一句就是在发送B时A的发送状态是未知的。
针对以上的问题，严格的顺序消费还需要以下参数支持：max.in.flight.requests.per.connection
这个参数官方文档的解释是：

>The maximum number of unacknowledged requests the client will send on a single connection before blocking. Note that if this setting is set to be greater than 1 and there are failed sends, there is a risk of message re-ordering due to retries (i.e., if retries are enabled).

大体意思是：

>在发送阻塞前对于每个连接，正在发送但是发送状态未知的最大消息数量。如果设置大于1，那么就有可能存在有发送失败的情况下，因为重试发送导致的消息乱序问题。
所以我们应该将其设置为1，保证在后一条消息发送前，前一条的消息状态已经是可知的。
