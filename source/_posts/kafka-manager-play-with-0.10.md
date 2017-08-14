title: kafka-manager 0.10 试玩
layout: post
date: 2017-01-12
tags:
 - kafka
 - 大数据
categories:
 - 原创文章
---


# 项目简介

kafka-manager项目提供了kafka集群的常用管理和监控功能，官方描述如下：

>A tool for managing Apache Kafka.
It supports the following:
- Manage multiple clusters
- Easy inspection of cluster state (topics, consumers, offsets, brokers, replica distribution, partition distribution)
- Run preferred replica election
- Generate partition assignments with option to select brokers to use
- Run reassignment of partition (based on generated assignments)
- Create a topic with optional topic configs (0.8.1.1 has different configs than 0.8.2+)
- Delete topic (only supported on 0.8.2+ and remember set delete.topic.enable=true in broker config)
- Topic list now indicates topics marked for deletion (only supported on 0.8.2+)
- Batch generate partition assignments for multiple topics with option to select brokers to use
- Batch run reassignment of partition for multiple topics
- Add partitions to existing topic
- Update config for existing topic
- Optionally enable JMX polling for broker level and topic level metrics.
- Optionally filter out consumers that do not have ids/ owners/ & offsets/ directories in zookeeper.


Kafka-manger项目：http://www.github.com/yahoo/kafka-manager

该项目目前尚未支持kafka 0.10.x 系列版本。不过在其的PR中有关于支持0.10版本的PR：https://github.com/yahoo/kafka-manager/pull/282

<!--more-->

# 本地合并步骤
git clone https://github.com/yahoo/kafka-manager.git
git fetch origin pull/282/head:0.10.0
git checkout 0.10.0

# 代码编译
cd kafka-manager
./sbt clean dist
如果是全新的编译环境，编译时间会非常久（主要原因是要初始化sbt编译系统的各种，比如下载ivy2依赖的jar包等）
编译完成后在目录 target/universal目录下会出现zip文件即为编译打包成功的部署包。

# 部署
解压到服务器某目录，只需要配置conf/application.conf文件内容即可：
kafka-manager.zkhosts="172.19.11.197:2181"
以上ZK地址根据实际情况修改成合适的地址。

# 启动
bin目录启动，./kafka-manger 

# kafka-manger特性研究
| 功能模块        | 功能点           | 是否具备  | 备注 |
| :-------------: |:-------------:| :-----:| :----:| 
| 集群管理       | 添加集群       | √ | | 
|       | 修改集群      |   √ | | 
|       | 启/禁用集群      |   √ | | 
|       | 删除集群      |   √ | | 
|       | 集群列表      |   √ | | 
| Broker管理 | Broker列表 |   √ | [指标](#metrics1)| 
|       | Broker详情      |   √ | 消息总量/Topic列表| 
| Topic管理 | Topic创建 |   √ | | 
|       | Topic列表      |   √ |  [指标](#metrics2)| 
|       | Topic详情      |   √ | | 
|       | Topic删除      |   √ | | 
|       | 分区调整      |   √ | | 
|       | 分区信息      |   √ | | 
| 订阅管理 | Consumer列表 |   √ | 需要JMX开启支持 | 
|       | Consumer详情      |   √ | | |
| 权限管理 | 登录 |   √ | 提供basic auth，后台可配置用户名、密码登录 | 


<span id = "metrics1">
## Broker 指标

 1. Message in /sec
 2. Bytes in /sec
 3. Bytes out /sec
 4. Bytes rejected /sec
 5. Failed fetch request /sec
 6. Failed produce request /sec	

 ** 需要Broker开启JMX **
</span>

<span id = "metrics2">
## Topic 指标

 1. Replication （副本数）
 2. Number of Partitions (分区数)
 3. Sum of partition offsets (offset大小，需要开启JMX支持）
 4. Total number of Brokers （Broker总数）
 5. Number of Brokers for Topic （Topic所占Broker数）
 6. Preferred Replicas %  （）
 7. Brokers Skewed % （Broker 均衡率）
 8. Brokers Spread % （Broker  扩散率）
 9. Under-replicated % （处于同步状态的比率）	

 ** 需要JMX开启支持 **
</span>

** 风险项	0.10版支持	　	master上尚未合并0.10版本的PR，需要自己合并测试后可用	**

附一些界面截图：
{% asset_img kafka-manager1.png [主页] %}
{% asset_img kafka-manager2.png [Broker列表及指标] %}
