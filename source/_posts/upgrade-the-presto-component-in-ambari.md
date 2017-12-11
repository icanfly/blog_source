title: 升级Ambari中的Presto组件
date: 2017-08-10
tags:
 - ambari
 - presto
 - 大数据
categories:
 - 原创文章
thumbnail: /images/bigdata.png
---

# 前言

在几天前我的一篇文章中，提到了需要升级公司的Presto数据查询引擎，而我们的Presto引擎是由Ambari管理的，升级Presto并没有任何文档可以参考，都是自己不断摸索出来的路子。所以写下本文用于记录摸索过程，并留待以后方便追溯和查看。

对于Ambari管理的组件，对于HDP发行版来说，单个组件的升级可能导致HDP发行版其它组件间的冲突，所以Ambari并没有提供单个组件的升级。而Ambari提供的升级方式是Ambari主控台先进行升级，升级后的Ambari可以在WEB界面上完成各个组件的升级，这样所有组件的统一升级的结果是：拥有最新的比较稳定的版本，且各组件间可以很好的配合使用，不会出现冲突，这个是Hortonworks公司帮我已经解决的依赖及冲突问题，这也是使用类似CDH或者HDP这样的Hadoop发行版的原因，省事！

当然值得一提的是，升级大数据平台组件是一件需要非常谨慎又小心的事情，不到万不得已请不要贸然升级，升级前还要做好万全的准备，比如数据备份，升级计划梳理，灾难预案等等。

我这里升级的是Presto，是一个比较独立的组件，所以我打算单独升级它，并不使用整体升级Ambari来更新所有的组件，而且当前Presto并没有提供更新的Ambari插件，所以这里需要我自己进行一些Hack。

<!-- more -->

# 准备工作

虽然是在测试环境试验，但是仍然需要先对数据及配置进行备份，这是升级必备要做的事。而备份对于Presto引擎来说只限于备份配置数据，Presto引擎本身是基本纯内存计算的数据查询引擎，故并不存在持久化于磁盘之上的数据。

## 准备Presto升级用的RPM包

根据Presto Ambari插件源码，它是通过Maven仓库进行下载的：
```
[download]
presto_rpm_url = http://search.maven.org/remotecontent?filepath=com/facebook/presto/presto-server-rpm/0.161/presto-server-rpm-0.161.rpm
presto_cli_url = http://search.maven.org/remotecontent?filepath=com/facebook/presto/presto-cli/0.161/presto-cli-0.161-executable.jar

```

这里因为国内的一些因素从Maven中央仓库下载会非常缓慢，可以改成自己的私源Maven仓库或者将RPM包和Jar包从中央仓库手动下载后上传至私有静态文件服务器，然后把上面的文件路径地址改成你添加的新位置。

同时，因为目前最新的Presto RPM包版本为0.182，所以推荐将老版本的RPM包和Jar包更新成最新版，因为新版本还是解决了很多问题，其中有一个内存泄漏的问题，在前面的文章中提及过，也是为什么要升级Presto的原因。

具体链接如下：[Presto内存溢出(OutOfMemory)问题排查](/2017/08/08/troubleshooting-of-outofmemory-in-presto/)

## 备份原有的Presto配置

原有的Presto配置文件是通过Ambari Web管理控制台进行配置的，目前来看并没有针对单个组件的备份方式，Ambari[整体备份方式](https://ambari.apache.org/current/installing-hadoop-using-ambari/content/ambari-chap11-1.html)是通过备份ambari的数据库表来达到的，这不是我们想要的。

既然没有简单的备份方式，那么唯一可做的就是将每个配置项扣出来，自己通过某个方式进行备份。通过分析Ambari Web主控台的交互，发现有一个Web API可以获取到服务组件的配置，该API接口返回Json结构数据，这个Json数据可以用来作为备份数据：

> API接口形如：`/api/v1/clusters/test_cluster/configurations/service_config_versions?service_name.in(PRESTO)&is_current=true&fields=*&_=1502331979100`
  {test_cluster}为集群的名称，需要根据自己的实际Ambari集群进行修改
  {PRESTO}为安装的Presto服务名字，这个也需要根据自己安装定义的Presto服务组件名称指定

返回数据形式如下：

```json
{
  "href" : "http://172.17.31.251:8080/api/v1/clusters/test_cluster/configurations/service_config_versions?service_name.in(PRESTO)&is_current=true&fields=*&_=1502331979100",
  "items" : [
    {
      "href" : "http://172.17.31.251:8080/api/v1/clusters/test_cluster/configurations/service_config_versions?service_name=PRESTO&service_config_version=3",
      "cluster_name" : "test_cluster",
      "configurations" : [
        {
          "Config" : {
            "cluster_name" : "test_cluster",
            "stack_id" : "HDP-2.5"
          },
          "type" : "config.properties",
          "tag" : "version1502248133724",
          "version" : 9,
          "properties" : {
            "discovery.uri" : "http://master.hdp.test.cq:8285",
            "http-server.http.port" : "8285",
            "node-scheduler.include-coordinator" : "false",
            "query.max-memory" : "50",
            "query.max-memory-per-node" : "1"
          },
          "properties_attributes" : { }
        },
        {
          "Config" : {
            "cluster_name" : "test_cluster",
            "stack_id" : "HDP-2.5"
          },
          "type" : "connectors.properties",
          "tag" : "version1502269954279",
          "version" : 11,
          "properties" : {
            "connectors.to.add" : "{\n    'hive': ['connector.name=hive-hadoop2','hive.metastore.uri=thrift://master.hdp.test.cq:9083,thrift://slave3.hdp.test.cq:9083','hive.recursive-directories=true','hive.config.resources=/etc/hadoop/conf/core-site.xml,/etc/hadoop/conf/hdfs-site.xml']\n}",
            "connectors.to.delete" : "[]"
          },
          "properties_attributes" : { }
        },
        {
          "Config" : {
            "cluster_name" : "test_cluster",
            "stack_id" : "HDP-2.5"
          },
          "type" : "jvm.config",
          "tag" : "version1502330557517",
          "version" : 10,
          "properties" : {
            "jvm.config" : "-server\n-Xmx8G\n-XX:+UseG1GC\n-XX:+UseGCOverheadLimit\n-XX:+ExplicitGCInvokesConcurrent\n-XX:+HeapDumpOnOutOfMemoryError\n-XX:OnOutOfMemoryError=kill -9 %p"
          },
          "properties_attributes" : { }
        },
        {
          "Config" : {
            "cluster_name" : "test_cluster",
            "stack_id" : "HDP-2.5"
          },
          "type" : "node.properties",
          "tag" : "version1502269954279",
          "version" : 12,
          "properties" : {
            "node.environment" : "test",
            "plugin.config-dir" : "/etc/presto/catalog",
            "plugin.dir" : "/usr/lib/presto/lib/plugin"
          },
          "properties_attributes" : { }
        }
      ],
      "createtime" : 1502330556753,
      "group_id" : -1,
      "group_name" : "Default",
      "hosts" : [ ],
      "is_cluster_compatible" : true,
      "is_current" : true,
      "service_config_version" : 3,
      "service_config_version_note" : "",
      "service_name" : "PRESTO",
      "stack_id" : "HDP-2.5",
      "user" : "admin"
    }
  ]
}
```

# 升级步骤

## 第一步：更新Ambari主控机上的Presto插件目录

因为Ambari主控机上安装的Presto插件版本比较旧，且官方并没有怎么进行维护，有一些潜在的问题。这里我们需要对该Presto插件目录进行更新修复一些问题。

具体的修复过程，参见另一篇文章：[修复ambari presto插件在重启时的BUG](/2017/08/09/fix-the-bug-of-the-ambari-presto-plugin-at-restart/)

将修复过后的文件夹拷贝至Ambari的插件目录(通常位于/var/lib/ambari-server/resources/stacks/HDP/2.5/services下，2.5为当前HDP的版本信息)替换原有的PRESTO目录。

完成更新后，需要重启Ambari-server，这样才能让Ambari-server重新识别新的Presto插件目录。

## 第二步：停止Presto服务

在Ambari Web管控台上停止所有的Presto实例

{% asset_img BC3D34CA-5024-4F2A-8B78-4DEF99AF8A14.png %}

## 第三步：在Ambari上删除Presto服务

在第二步完成后（Ambari要求删除服务组件必须是在服务停止之后）进行Presto服务的删除操作：

{% asset_img 2FEB8D0C-48D8-4CA8-A67E-9B8133286069.png %}

## 第四步：在Presto机器上删除相关部署及部署缓存文件

1. Ambari的各个slave节点Agent在安装组件前会先将组件插件目录的packages下的一些脚本文件复制到本机并缓存，以便下次不再从主控机拷贝，由于我们这里进行了插件的升级更新，所以需要手工清理slave机器的缓存目录。

清理操作：

```shell
rm -rf /var/lib/ambari-agent/cache/stacks/HDP/2.5/services/PRESTO && ambari-agent restart

```
上述删除缓存目录并重启ambari-agent后，ambari-agent会重新从主控机master拉取服务插件的最新文件

2. Ambari不支持删除服务的同时对安装包进行卸载，也就是说删除的只是在Ambari管理的服务的可见性，服务其实仍然安装在Slave的机器上，如果我们需要安装新包，需要先卸载旧版本组件包。当然也可以对脚本进行进一步的修改，比如先检测组件包是否已经安装，如果安装先进行卸载等等操作，这里我们暂不这样操作。

卸载presto组件包：

```shell
rpm -e `rpm -qa | grep presto`

```

## 第五步：在Ambari主控机上重新添加服务并配置部署机器

上面第三步中删除了Presto，这里我们可以通过主控台的Admin->Stack And Versions重新添加服务，这里我们可以看到PRESTO服务已经是我们最新添加的版本了，点击Add Service根据提示一步步进行操作

## 第六步：还原配置项

在安装过程中会提示进行配置，这个时候就是需要我们还原之前备份的配置项，这点Ambari做并不好，没有什么类似配置导入的功能，所以这里需要我们手动的从备份的文件中找出各个配置项手工填入。

## 第七步：根据提示完成部署

在一切都准备就绪后，Ambari会自动进行组件的安装和配置，并会操作组件启动。

# 问题

在安装过程中可能会出现以下问题：

- sudo: sorry, you must have a tty to run sudo

使用不同账户，执行执行脚本时候sudo经常会碰到 sudo: sorry, you must have a tty to run sudo这个情况，其实修改一下sudo的配置就好了

vi /etc/sudoers (最好用visudo命令)

注释掉 Default requiretty 一行

#Default requiretty

意思就是sudo默认需要tty终端。注释掉就可以在后台执行了。

- 彻底删除某个组件的服务

这里有一个链接，里面主要讲怎么彻底删除组件，这个删除包括从Ambari-Server的主控台上删除，还包括从物理安装中移除，参考链接在这里：[Ambari里如何删除某指定的服务（图文详解）](http://www.cnblogs.com/zlslch/p/6653421.html)

# 总结

Ambari的插件机制对于更新来说并不是特别方便，它自身提供的更新机制并不适用于自定义的组件升级，需要根据上面所说的进行自定义升级操作。对于HDP发行版自带的组件升级推荐使用升级Ambari版本的方式进行，这样可以做到HDP发行版的完全兼容处理。Ambari在自定义组件升级前一定要做好数据和配置的备份工作，以应对升级失败的回滚方案。

最后，对于一个企业来说，数据就是它的生命，要对数据存敬畏之心，任何一次组件的操作都可能千万数据的损坏和丢失，对于企业来说都是致命性的打击。对任何一次数据组件的升级都要小心又谨慎，制定周密的升级计划和灾难预案。在实施线上操作之前，勿必先在测试环境进行验证可行性，切不可只盲目听从文档或者网上言论就贸然实施。
