title: 基于YARN-Based集群的presto自动安装(Presto On YARN)
date: 2017-05-24
tags:
 - presto
 - 大数据
 - yarn
categories:
 - 大数据

---

如果你正计划使用HDP的发行版，那么你可以使用Ambari和Apache Slider来执行基于YARN的Presto的自动安装和集成，在安装过程中，Apache Slider和Presto的包都会被安装。

# 部署Presto到基于YARN的集群

安装部署的前提是假设你有一些关于Presto的基础知识以及了解它的一些配置文件。所有的例子引用都来至于：https://github.com/prestodb/presto-yarn/

## 前提条件

- 基于HDP 2.2+ 或者 CDH 5.4+ 的集群
- Apache Slider 0.80.0（可以从这个地址[下载](https://slider.incubator.apache.org/)）
- JDK 1.8
- Zookeeper
- openssl >= 1.0.1e-16
- Ambari 2.1

<!--more-->

## Presto安装目录结构

当你使用Ambari Slider View在一个YARN集群上安装Presto的时候，Presto的安装目录不同于标准的目录，这里会有一些区别。

如果你使用Slider Scripts或者使用Ambari slider view安装Presto到YARN集群，Presto将会通过使用Presto tarball包安装（并不是rpm包）。安装发生在YARN应用被启动时并且你可以在你的YARN nodemanager节点上找到Presto server的安装目录，该目录是由`yarn.nodemanager.local-dirs`该参数指定的。 比如你的`yarn.nodemanager.local-dirs`参数指定为/mnt/hadoop/nm-local-dirs，并且`app_user`为yarn，那么你就会发现Presto被安装在了/mnt/hadoop-hdfs/nm-local-dir/usercache/yarn/appcache/application_<id>/container_<id>/app/install/presto-server-<version>，这个路径的第一部分（直到container_id）在Slider中被称作AGENT_WORK_ROOT，那么这么来说，Presto就是被安装在`AGENT_WORK_ROOT/app/install/presto-server-<version>` 这里。

对于正常情况下的Presto安装，会将presto的catalog、plugin以及lib目录安装在presto安装的主目录下。
同样的在这里，catalog目录会是在`AGENT_WORK_ROOT/app/install/presto-server-<version>/etc/catalog`，plugin和lib目录会是分别在`AGENT_WORK_ROOT/app/install/presto-server-<version>/plugin`和`AGENT_WORK_ROOT/app/install/presto-server-<version>/lib`，用于启动Presto服务的脚本会是在`AGENT_WORK_ROOT/app/install/presto-server-<version>/bin`.

Presto的日志目录是基于你配置的`data directory`目录下的， 如果你在`appConfig.json`配置成`/var/lib/presto/data`，那么你就会得到presto的日志目录`/var/lib/presto/data/var/log/`

## Presto安装配置选项

在安装过程中，Ambari Slider View允许你设置Presto运行的必要参数。

## 使用Ambari Slider View来安装Presto到YARN集群上

Ambari支持通过Slider View部署Slider应用包并提供Slider的集成。 Slider View for Ambari允许你通过Ambari WEB控制台部署和管理Slider应用。

使用Ambari Slider View安装Presto到YARN集群的步骤如下：

1、安装Ambari，如果没有安装，这里有一些[教程](http://docs.hortonworks.com/HDPDocuments/Ambari-2.1.0.0/bk_Installing_HDP_AMB/content/ch_Installing_Ambari.html)

2、下载[Apache Slider](https://slider.incubator.apache.org/)
3、复制Presto应用包`presto-yarn-package-<version>-<presto-version>.zip`到`/var/lib/ambari-server/resources/apps/`(你的ambari服务节点上)
4、重启Ambari服务器
5、重新登录Ambari服务器
6、配置Ambari的中间过程略过...
7、配置和自定义服务，并安装它们，Slider的最小服务集是：HDFS,YARN,Zookeeper。当然你也必须选择安装Slider。
8、对于Slider客户端安装，你需要更新它的配置如果你不是使用默认配置安装的Hadoop和Zookeeper。因此`slider-env.sh`应该需要指出你的`JAVA_HOME`和`HADOOP_CONF_DIR`

`
export JAVA_HOME=/usr/lib/jvm/java
export HADOOP_CONF_DIR=/etc/hadoop/conf
`
9、对于Zookeeper，如果你使用了一个不同的区别于默认的`/usr/lib/zookeeper`的目录：
- 那么请在`slider-client`节添加一个自定义属性`zk.home`,值是你的zookeeper路径。
- 如果zookeeper不是使用的默认端口2181，那么你还需要指定`slider.zookeeper.quorum`,形式为`node:port`。

10、当所的有服务都安装完毕并且运行起来后，你就可以在Ambari中配置Slider来创建和管理你的应用了。
- 点击`Admin`（左上角）-> Manage Ambari
- 从左侧面板中选择`Views`
- 创建Slider View，填入必要的字段。 `ambari.server.url`格式为：`http://<ambari-server-url>:8080/api/v1/clusters/<clustername>`，其中clustername是你Ambari集群的名字。
- 选择右上角的`Views`控制按钮
- 选择你在上一步创建的实例（例如：‘Slider’）
- 点击`Create App`来创建一个新的Presto YARN应用

11、提供Presto服务详细配置，默认情况下，UI会从`*-default.json`文件中计算读取，这些文件都存在于你的`presto-yarn-package-*.zip`文件中。
12、应用名必须是小写的，例如：presto1
13、你可以设置一些配置值，比如你想给presto设置一个connector，那么只需要更新`global.catalog`的属性值就可以了，下面这个链接是对各个配置值的解释。
[Presto Configuration Options for YARN-Based Clusters](https://prestodb.io/presto-yarn/installation-yarn-configuration-options.html)
14、为Slider准备HDFS。HDFS的用户目录应该和你的配置文件中设置的`global.app_user`字段内容一致。假如app_user被设置成yarn，那么操作就会像下面这样：
su hdfs hdfs dfs -mkdir -p /user/yarn
su hdfs hdfs dfs -chown yarn:yarn /user/yarn
15、将配置中的`global.presto_server_port`从8080改成其它值，比如8089，因为8080可能已经被Ambari或者其它服务占用。总之是需要找到一个可用的端口。
16、提前在每个节点上创建好presto的本地目录（目录配置在`appConfig-default.json`中，例如：/var/lib/presto/），并且该目录的拥有者必须是`global.app_user`的配置值，不然Slider会因为权限问题不能成功启动Presto服务
mkdir -p /var/lib/presto/data
chown -R yarn:hadoop /var/lib/presto/data
17、如果你想添加一些额外的配置属性，可以使用自定义属性那节，额外的属性目前支持如下：
- site.global.plugin
- site.global.additional_config_properties
- site.global.additional_node_properties
以上的参数格式请参考(Presto Configuration Options for YARN-Based Clusters)[https://prestodb.io/presto-yarn/installation-yarn-configuration-options.html]

18、点击完成。这步等效于Slider的bin/slider脚本中'package install'以及'create'。 如果操作成功，你就会看到YARN应用被成功拉起，你可以操作如下：
 - 点击`app launched`，查看Slider view的状态
 - 点击`Quick Links`，这会带你到YARN的WEBUI中

19、如果部署失败了，就需要检查一下任务的历史日志了，或者看下节点的本地日志，可以参考(Debugging and Logging for YARN-Based Clusters)[https://prestodb.io/presto-yarn/installation-yarn-debugging-logging.html]
20、你可以在UI界面上管理应用的生命周期（如：`start`,`stop`,`flex`,`destroy`等）

# 额外的配置选项

当你安装完Presto和Slider后，你可以重新配置Presto的配置项或者添加更多的配置

## 在Slider View中重新配置Presto

在你启动Presto后，你也可以更新它的配置，例如，你想添加一个新的connector。
1、在Slider View的实例界面上点击`Actions`
2、停止正在运行的Presto应用
3、点击`Destory`来删除掉在Slider中已经存在的实例
4、点击`Create App`按钮重新创建一个新实例，并使用新的配置值

## 高级配置选项

- 配置内存、CPU、以及YARN的CGroups
- 失败策略
- YARN Lable
更多信息，参见[Advanced Configuration Options for YARN-Based Clusters](https://prestodb.io/presto-yarn/installation-yarn-configuration-options-advanced.html)

# 调试和日志
更多信息，参见：[Debugging and Logging for YARN-Based Clusters](https://prestodb.io/presto-yarn/installation-yarn-debugging-logging.html)

# 参考链接

- [http://slider.incubator.apache.org/docs/getting_started.html](http://slider.incubator.apache.org/docs/getting_started.html)
- [http://docs.hortonworks.com/HDPDocuments/Ambari-2.0.1.0/bk_Installing_HDP_AMB/content/ch_Installing_Ambari.html](http://docs.hortonworks.com/HDPDocuments/Ambari-2.0.1.0/bk_Installing_HDP_AMB/content/ch_Installing_Ambari.html)