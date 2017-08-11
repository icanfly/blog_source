title: Apache Ranger在HDFS中的最佳实践
date: 2017-04-11

tags:
 - ranger
 - hdfs
 - 大数据
 - 最佳实践
 - hortonworks
categories:
 - 大数据
---

HDFS对于任何Hadoop大数据平台来说都是核心组成部分，为了加强对Hadoop平台的数据保护，将安全控制深入到HDFS层是非常有必要的。HDFS本身提供了Kerberos认证，并且提供了基于POSIX风格的权限和HDFS——ACL控制，当然它也可以使用基于Apache Ranger的权限控制体系。

Apache Ranger (http://hortonworks.com/hadoop/ranger/) 是一个集中式的Hadoop体系的安全管理解决方案，它提供给管理者在HDFS或者其它Hadoop体系组件上创建和应用安全策略的功能。

# Ranger是怎么在HDFS上工作的？

为了在HDP发行版中加强安全性，建议安装和配置Kerberos, Apache Knox和Apache Ranger。

<b>
Apache Ranger提供了一个和HDFS原生权限相匹配适应的授权模型。 HDFS Ranger插件会首先检测是否存在对应的授权策略对应用户授权，如果存在那么用户权限检测通过。如果没有这样的策略，那么Ranger插件会启用HDFS原生的权限体系进行权限检查（POSIX or HDFS ACL）。这种模型在Ranger中适用于HDFS和YARN服务。
</b>

{% asset_img hdfs-ranger-authorization-model.png [HDFS Ranger授权模型] %}

<!--more-->

对于其它服务，比如Hive或者HBase，Ranger是作为唯一的有效授权依据，在HDP发行版中，使用Ambari配置HDFS使用Ranger的路径如下：Ambari → Ranger → HDFS config → Advanced ranger-hdfs-security

{% asset_img ambari-hdfs-ranger-config.png [HDFS Ranger配置] %}

基于Ranger的这种授权模型，用户可以非常安全的将Ranger添加到已经存在的大数据集群中，而该集群可能之前一直使用的是基于POSIX的权限体系。建议对于所有的部署，这个选项都设置成true。

<b>
Ranger的用户界面可以让管理者非常容易地找到用户的授权关系（Ranger policy or native HDFS）</b> 用户可以方便的查看审计内容（路径为：Ranger→ Audit），如果在界面上“Access Enforcer”列的内容为“Ranger-acl”，那说明Ranger的策略被应用到了用户身上。如果“Access Enforcer”列的内容为“Hadoop-acl”,表示该访问是由HDFS原生的POSIX权限和HDFS ACL提供的。

{% asset_img ranger-audit.png [Ranger审计日志] %}

*Having a federated authorization model may create a challenge for security administrators looking to plan a security model for HDFS.*

# 如何确保安全

当Ranger和Hadoop都安装完后，建议管理员按下面的步骤进行配置：

- Change HDFS umask to 077
- Identify directory which can be managed by Ranger policies
- Identify directories which need to be managed by HDFS native permissions
- Enable Ranger policy to audit all records

改变HDFS掩码为077，确定哪些目录由Ranger授权管理，哪些目录由HDFS原生权限控制。启用Ranger的审计功能



下面是详细步骤：

- Change HDFS umask to 077 from 022. This will prevent any new files or folders to be accessed by anyone other than the owner
管理员可以在Ambari中这样操作：
{% asset_img hdfs-umask.png [HDFS 掩码设置] %}

在HDFS中默认的掩码为022，在这种情况下，所有的用户都具有所有HDFS文件系统文件和文件夹的读取权限。 你可以通过以下命令进行检查：

> 
$ hdfs dfs -ls /apps
Found 3 items
drwxrwxrwx   – falcon hdfs       0 2015-11-30 08:02 /apps/falcon
drwxr-xr-x   – hdfs   hdfs           0 2015-11-30 07:56 /apps/hbase
drwxr-xr-x   – hdfs   hdfs           0 2015-11-30 08:01 /apps/hive

- 指定哪些目录由Ranger授权

<b>建议这些目录由Ranger来进行管理和授权（/app/hive,/apps/Hbase以及一些自定义的数据目录）</b> HDFS本身的授权模型对于这些需求来说显得捉襟见肘。 可以使用chmod修改默认权限，例如：
> 
$ hdfs dfs -chmod -R 000 /apps/hive
$ hdfs dfs -chown -R hdfs:hdfs /apps/hive
$ hdfs dfs -ls /apps/hive
Found 1 items
d———   – hdfs hdfs          0 2015-11-30 08:01 /apps/hive/warehouse

Ranger可以给用户进行显式的授权，例如：
{% asset_img ranger-authorize.png [Ranger授权] %}

管理员可以照着这个图对其它目录进行用户授权，你可以通过以下方式进行授权验证：
 - Connect to HiveServer2 using beeline
 - Create a table 
  - create table employee( id int, name String, ssn String);
 - Go to ranger, and check the HDFS access audit. The enforcer should be ‘ranger-acl’

 {% asset_img ranger-audit2.png [审计日志] %}

- 指定哪些目录由HDFS原生权限控制

建议让HDFS原生权限管理/tmp和/user目录。这些目录通常被各种应用使用于创建用户级的目录。这里你也需要设置/user目录的权限为“700”:
> $ hdfs dfs -ls /user
Found 4 items
drwxrwx—   – ambari-qa hdfs          0 2015-11-30 07:56 /user/ambari-qa
drwxr-xr-x   – hcat      hdfs          0 2015-11-30 08:01 /user/hcat
drwxr-xr-x   – hive      hdfs          0 2015-11-30 08:01 /user/hive
drwxrwxr-x   – oozie     hdfs          0 2015-11-30 08:02 /user/oozie
 

> $ hdfs dfs -chmod -R 700 /user/*
$ hdfs dfs -ls /user
Found 4 items
drwx——   – ambari-qa hdfs          0 2015-11-30 07:56 /user/ambari-qa
drwx——   – hcat      hdfs          0 2015-11-30 08:01 /user/hcat
drwx——   – hive      hdfs          0 2015-11-30 08:01 /user/hive
drwx——   – oozie     hdfs          0 2015-11-30 08:02 /user/oozie

- 确保所有的HDFS数据操作都是被审计的
当Ranger是通过Ambari安装时，它会创建一个默认的策略，该策略允许所有的目录和文件访问并开启审计功能。这个策略同时也应用于Ambari冒烟测试用户“ambari-qa”用来验证HDFS服务的可用性。如果管理员禁用或者删除了该策略，那么需要创建一个类似的策略来允许审计所有的文件和目录。

{% asset_img ranger-policy.png [审计日志] %}

# 总结

保证HDFS的安全性是保证Hadoop安全性的起点。 Ranger为HDFS提供了一个集中统一管理安全策略的接口。建议管理员合理使用Ranger以及HDFS本身的权限机制来全程覆盖HDFS的授权管理。

本文英文原文：https://hortonworks.com/blog/best-practices-in-hdfs-authorization-with-apache-ranger/

  
  