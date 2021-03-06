title: 基于Apache Ranger的Presto数据权限控制
date: 2017-07-14
tag:
 - bug
 - druid
 - 大数据

categories:
 - 原创文章
thumbnail:
toc: true
---

我司大数据项目数据查询引擎使用到了presto，但是对于presto本身来说并没有人员及数据权限粒度的权限控制，我在上面做了一层封装。

# Apache Ranger权限控制

Apache Ranger是Apache开源的一款Hadoop生态的安全管理框架，它可以管理诸如：HDFS、HBASE、YARN等的权限访问和控制，提供了统一的权限管理后台，可以实施在线的权限变更和访问日志审计。
具体怎么使用Ranger进行权限控制及自定义服务的新增，可以参考以前的一篇文章：[Ranger自定义插件开发](http://www.lpnote.com/2017/01/23/how-to-add-a-custom-plugin-in-ranger/)

# sql解析引擎

## 使用Druid解析

SQL解析引擎最开始使用的是Druid开源SQL解析引擎，相对来说它对于SQL的解析也是比较简单的，在我们的场景下，我们需要控制人员能够访问到大数据仓库的catalog/schema/table/column级，所以我们需要将sql中的涉及到的表和列需要解析出来，而Druid可以满足我们，虽然Druid提供了Oracle/Mysql/Postgresql等传统关系数据库的sql解析，但是并没有一些开源nosql的类sql解析，但像presto这样的开源数据查询引擎本身支持标准的sql语句，所以对于我们来说大部分的sql它是可以进行标准sql解析的，所以我们就采用了它来进行sql解析的工作。

一切其实都进行得比较顺利，期间也出现过一些小问题最后都比较好的解决，其中有一个比较大的问题是发现了一个解析复杂sql语句的bug，该bug我已经提交给温少，详细的issue地址如下：[复杂sql解析不正确的问题](https://github.com/alibaba/druid/issues/1831)。该bug主要的问题是在存在union查询时，如果两边的子句中不同的表使用了相同的别名，在最后的解析结果时会出现混乱，表和列的对应关系会出现问题，终其原因是应该是在解析器的工作状态下只存在了一张映射表，而且该表没有命名空间的概念，导致有不同命名空间(即不同子句下)的表及别名产生了相互覆盖和影响。

<b>最新更新： 作者已经修复bug，最新版本中1.1.2中作者已经声称修复了该bug，但是我们已经换作自己的解析器工作，所以并没有及时去验证</b>

## 使用自研解析

Presto数据查询引擎实际使用的是Antlr进行自定义的SQL解析，它本身提供了一些AST的节点信息Java源码，所以我只需要将SQL传入它的解析器得到一个Statement，根据Statement的详细情况遍历其节点及子节点并解析其内容就要吧得到相关的数据表和列，解析过程中我同样根据AST的路径重新生成了一棵树，通过遍历树就可以知道表别名及真实表的对应关系，以及字段和数据表的真实对应关系。

这里值得一提的是，虽然SQL解析的问题解决了，但是我们真正要解决的问题是得到非常准确的SQL中涉及到的表和字段。这里我们需要解决的是诸如：select * from a,b这样的字段解析，因为*为表所有的字段，这里有两张表，所以解析的时候要注意到两张表的字段都解析出来，还有一个地方要注意的是：select id from a,b， 在该情况下id并不确定它是属于a表还是b表，所以这里对于纯的sql解析还是存在困难的，对于具体权限的解析还要涉及对应于数据源的元数据schema结合来进行判断id字段应该是属于哪张表还是两张表都有，这样才能将id归属到对应表。

# 权限控制

在应用程序中通过对SQL的语法解析，得到了数据表(形如: catalog.schema.table)和数据列的对应关系，而我们在ranger中新定义了一种服务类型，并通过新增ranger插件的形式与应用程序连通。应用程序通过Ranger提供的插件SDK，通过相关的接口将用户名，数据表和数据列传入，Ranger插件结合从Ranger服务器得到的权限及策略进行匹配，如果匹配成功则返回成功响应，应用程序收到权限允许的响应后就放行用户的查询，如果权限验证失败，则提示用户查询权限被禁止。

# 总结

Presto本身并未提供用户SQL查询权限的控制，我们实现的方式相对来说比较简单，我们自身提供了一层WEB层用于用户进行数据查询，所以所有的权限控制都做在了这WEB层，只要这WEB层控制住了用户的权限，那么在执行引擎之前就可以拦截非法的SQL执行请求。
