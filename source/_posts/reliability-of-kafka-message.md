---
title: kafka消息可靠性
date: 2017-01-15 20:09:04
tags:
- kafka
- 大数据
categories:
 - 原创文章
thumbnail:
---


kafka消息可靠性，这里涉及几个方面：

- 发送可靠性
- 接收可靠性
- 存储可靠性

# 发送可靠性

kafka新版client(0.10.x)使用java重新实现。使用的是异步方式发送消息，即消息提交给KafkaProducer的send方法后，实际上是将该消息放入了它本身的一个后台发送队列，然后再有一个后台线程不断地从队列中取出消息进行发送，发送成功后会回调send方法的callback（如果没有，就不用回调了）。

从以上的流程来看，kafka客户端的发送流程是一个异步化的流程，kafka客户端会累积一定量的消息后统一组装成一个批量消息发出，这个的触发条件是： 消息量达到了batch.size的大小或者等待批量的时间超过了linger.ms时间。

此外还要注意一下发送方消息的堆积问题，当程序的发送速率大于发送到broker的速率时，会产生消费在发送方堆积，堆积的策略控制主要由参数buffer.memory 以及max.block.ms。buffer.memory设置了可使用的buffer内存，max.block.ms是指在buffer满的情况下可以阻塞多长时间，超过这个时间则抛出异常。

## 消息失败重试

设置失败重试的次数为一个很大的数值,如Integer.MAX_VALUE，对应properties的设置为：

| 配置        | 默认值   | 建议值  |
| :--------: |:--------:| :-----:|
| retries	 | 0	    | Integer.MAX_VALUE |

## 消息异步转同步

对于消息异步转同步：使用future.get()等待消息发送返回结果,如：
```java
Future<RecordMetadata> future = producer.send(new ProducerRecord<String, String>("test.testTopic", "key","value"));
RecordMetadata metadata = future.get(); //等待发送结果返回
```
这种用法可能会导致性能下降比较厉害，也可以通过send(message,callback)的方式，在消息发送失败时通过callback记录失败并处理

## 顺序消息
kafka默认情况下是批量发送，批量发送存在消息积累再发送的过程，为了达到消息send后立刻发送到broker的要求，对应properties设置：

| 配置        | 默认值   | 建议值  |
| :--------: |:--------:| :-----:|
| max.in.flight.requests.per.connection	 | 5	    | 1 |

其中max.in.flight.requests.per.connection以及retries主要应用于顺序消息场景，顺序场景中需要设置为：
max.in.flight.requests.per.connection = 1

<!--more-->

综合以上配置示例：
```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("acks", "1"); //这里是只要求leader响应就OK，更高的要求则应该设置成"all"
props.put("retries", Integer.MAX_VALUE);
props.put("max.in.flight.requests.per.connection",1);
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer"); //这里是key的序列化类
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");//这里是value的序列化类

Producer<String, String> producer = new KafkaProducer<String,String>(props);
for (int i = 0; i < 1000000; i++) {
   Future<RecordMetadata> future = producer.send(new ProducerRecord<String, String>("test.testTopic","key","value"));
   RecordMetadata metadata = future.get(); //关键的这一步，同步等待发送完成
}
producer.close();
```

# 接收可靠性

新版的java客户端（0.10.0.0）已经变更接收线程为单线程接收处理。
同时客户端默认情况下是自动提交offset，这样可能存在消息丢失的可能性，比如客户端接收到一批消息并进行处理，在处理过程中达到了客户端offset定时提交的时间点，这批数据的offset被提交，但是可能这批数据的处理还没有结束，甚至这些数据可能还存在一些数据处理不了或者处理出错，甚至出现宕机的可能性，这时未处理的消息将会丢失，因为offset已经提交，下次读取会从新的offset处读取。所以要保证消息的可靠接收，需要将enable.auto.commit设置为false，防止程序自动提交，应该由应用程序处理完成后手动提交。
示例：
```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("group.id", "testsub");
props.put("enable.auto.commit", "false");
props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
KafkaConsumer<String, String> consumer = new KafkaConsumer<String,String>(props);
consumer.subscribe(Arrays.asList("test.testTopic","testsub"));

while (true) {
   ConsumerRecords<String, String> records = consumer.poll(100);
   Map<TopicPartition,OffsetAndMetadata> commitMap = new HashMap(10);
   for (ConsumerRecord<String, String> record : records) {
      logger.info(record.value()); //模拟消息
      commitMap.put(new TopicPartition(record.topic(),record.partition()),new OffsetAndMetadata(record.offset()+1));
   }
   if(commitMap.isEmpty()){
      continue;
    }
    consumer.commitSync(commitMap);
}
```

# 存储可靠性

## 刷盘时机
broker的刷盘时机主要是以下两个参数控制：
log.flush.interval.ms                  日志刷盘的时间间隔，每隔多少时间将消息刷到磁盘上
log.flush.interval.messages      日志刷盘的消息量，每积累多少条消息将消息刷到磁盘上

## 副本数
在创建消息Topic的时候需要指定消息的副本数  replicas
一般建议设置成3保证消息的可靠，再结合客户端发送方的ack参数，当ack参数设置为0表示不等待broker响应就发送下一条消息，当ack设置为1则表示需要等待leader响应，当ack设置为all则表示需要等待所有的replicas ISR都响应后才返回响应，其中all是最高可靠级别了，但是同时也降低了吞吐率。
