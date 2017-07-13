title: 自动清理Solr Doc & Ranger Audits
date: 2017-07-12
tags:
 - ranger
 - 大数据
 - solr
categories:
 - 大数据

---

本文原文：[Solr TTL - Auto-Purging Solr Documents & Ranger Audits](https://community.hortonworks.com/articles/63853/solr-ttl-auto-purging-solr-documents-ranger-audits.html)

# 背景

最近公司的使用的大数据项目中频繁出现以下的报错:

`
ERROR [org.apache.ranger.audit.queue.AuditBatchQueue0] o.a.s.c.solrj.impl.CloudSolrClient:924- Request to collection ranger_audits failed due to (400) org.apache.solr.client.solrj.impl.HttpSolrClient$RemoteSolrException: Error from server at http://172.18.2.6:8886/solr/ranger_audits_shard1_replica1: Exception writing document id bc8d53c6-0878-450c-a9b0-1b8d34c55e1d to the index; possible analysis error: number of documents in the index cannot exceed 2147483519, retry? 0
`

搜索了一下原因，大致是说solr分片的索引量不能超过最大值（2的32次方）。由于线上solr是单机版本，所以数据量上已经超了，为了解决这个问题，我搜索到了下面这篇文章。

这篇文章将讲述如何在Solr文档上使用TTL(time to live)来自动清理过期文档。

这篇文章我们将关注如何自动地从Solr集合中删除文档，关于solr文档TTL的问题如下：

 - 节点磁盘满
 - 公司策略要求删除旧的审计日志(audit logs)
 - 自动清理等

SOLR-5795介绍了一个新的UpdateProcesser叫做`DocExpirationUpdateProcessorFactory`,它允许我们在solr文档上添加一个`过期时间`，并且确保过期的文档能够自动的被清除。

<!--more-->

# 如何做？

每个被索引的文档都有一个表示文档过期时间的字段（`_ttle_`一般情况下是这个字段，当然你也可以修改这个字段的名字），该字段标识了该文档将在什么时候过期。这个字段是由文档被索引的时间的相对时间确定的，`_ttle_`设置了文档的生命周期，例如：+10DAYS,+2WEEKS,+4HOURS等等。

例如：

当前时间为: 2016-10-26 20:14:00
_ttle_定义为：+2HOURS
那么过期时间将会是2016-10-26 22:14:00

`deleteByQuery`将会根据`autoDeletePeriodSeconds`的配置来触发，例如：86400，那么后台的一个线程将会将每天的方式执行删除操作(1天等于86400秒)。 这些`deleteByQuery`将会删除所有的过期时间小于当前时间的文档。

如果你想要自定义删除程序，你可以使用`autoDeleteChainName`来配置你自己的`updateRequestProcessorChain`,这个配置是对所有的删除生效的。

一旦删除程序完成，一个软提交将被触发，那么所有的过期的文档将不会出现在查询结果中了。

# 通常情况下的Solr

首先，来看看一个使用ttl的普通例子，在此我们使用一个叫做moveis的solr集合，并且我们希望在这个集合中的电影保存时间不超过10天。

找到solrCloud的一个节点（不用关心哪个节点，但是该节点上需要有zkcli客户端）

创建一个solr的初始化配置：

```
mkdir /opt/lucidworks-hdpsearch/solr_collections
mkdir /opt/lucidworks-hdpsearch/solr_collections/films 
chown -R solr:solr /opt/lucidworks-hdpsearch/solr_collections 
cp -R /opt/lucidworks-hdpsearch/solr/server/solr/configsets/basic_configs/conf /opt/lucidworks-hdpsearch/solr_collections/films
```

## 调整schema.xml

此例中文件在/opt/lucidworks-hdpsearch/solr_collections/films/conf下面

在schema.xml中添加以下字段的定义：

```xml
<field name="directed_by" type="string" indexed="true" stored="true" multiValued="true"/>
<field name="name" type="text_general" indexed="true" stored="true"/>
<field name="initial_release_date" type="string" indexed="true" stored="true"/>
<field name="genre" type="string" indexed="true" stored="true" multiValued="true"/>
```

同时添加以下字段用于自动清理

```xml
<field name="_ttl_" type="string" indexed="true" multiValued="false" stored="true" />
<field name="expire_at" type="date" multiValued="false" indexed="true" stored="true" />
```

`_ttl_`表示这个文档应该保存多长时间（例如：+10DAYS）
`expire_at`表示计算好的过期时间（索引时间+_ttl_）

## 调整solrconfig.xml

为了让过期时间能被自动计算，我们需要添加`DocExpirationUpdateProcessorFactory`

```xml
<updateRequestProcessorChain name="add-unknown-fields-to-the-schema">
    <processor class="solr.DefaultValueUpdateProcessorFactory">
      <str name="fieldName">_ttl_</str>
      <str name="value">+14DAYS</str>
    </processor>
    <processor class="solr.processor.DocExpirationUpdateProcessorFactory">
      <int name="autoDeletePeriodSeconds">300</int>
      <str name="ttlFieldName">_ttl_</str>
      <str name="expirationFieldName">expire_at</str>
    </processor>
    <processor class="solr.RemoveBlankFieldUpdateProcessorFactory"/>
    <processor class="solr.ParseBooleanFieldUpdateProcessorFactory"/>
    <processor class="solr.ParseLongFieldUpdateProcessorFactory"/>
    <processor class="solr.ParseDoubleFieldUpdateProcessorFactory"/>
    <processor class="solr.ParseDateFieldUpdateProcessorFactory">
      <arr name="format">
        <str>yyyy-MM-dd'T'HH:mm:ss.SSSZ</str>
        <str>yyyy-MM-dd'T'HH:mm:ss,SSSZ</str>
        <str>yyyy-MM-dd'T'HH:mm:ss.SSS</str>
        <str>yyyy-MM-dd'T'HH:mm:ss,SSS</str>
        <str>yyyy-MM-dd'T'HH:mm:ssZ</str>
        <str>yyyy-MM-dd'T'HH:mm:ss</str>
        <str>yyyy-MM-dd'T'HH:mmZ</str>
        <str>yyyy-MM-dd'T'HH:mm</str>
        <str>yyyy-MM-dd HH:mm:ss.SSSZ</str>
        <str>yyyy-MM-dd HH:mm:ss,SSSZ</str>
        <str>yyyy-MM-dd HH:mm:ss.SSS</str>
        <str>yyyy-MM-dd HH:mm:ss,SSS</str>
        <str>yyyy-MM-dd HH:mm:ssZ</str>
        <str>yyyy-MM-dd HH:mm:ss</str>
        <str>yyyy-MM-dd HH:mmZ</str>
        <str>yyyy-MM-dd HH:mm</str>
        <str>yyyy-MM-dd</str>
      </arr>
    </processor>
    <processor class="solr.AddSchemaFieldsUpdateProcessorFactory">
      <str name="defaultFieldType">text_general</str>
      <lst name="typeMapping">
        <str name="valueClass">java.lang.Boolean</str>
        <str name="fieldType">booleans</str>
      </lst>
      <lst name="typeMapping">
        <str name="valueClass">java.util.Date</str>
        <str name="fieldType">tdates</str>
      </lst>
      <lst name="typeMapping">
        <str name="valueClass">java.lang.Long</str>
        <str name="valueClass">java.lang.Integer</str>
        <str name="fieldType">tlongs</str>
      </lst>
      <lst name="typeMapping">
        <str name="valueClass">java.lang.Number</str>
        <str name="fieldType">tdoubles</str>
      </lst>
    </processor>
    <processor class="solr.LogUpdateProcessorFactory"/>
    <processor class="solr.RunUpdateProcessorFactory"/>
  </updateRequestProcessorChain>
```

同时要确保处理链在索引的每次更新都触发：

```xml
<initParamspath="/update/**,/query,/select,/tvrh,/elevate,/spell">
	<lst name="defaults">
		<str name="df">text</str>
		<str name="update.chain">add-unknown-fields-to-the-schema</str>
	</lst>
</initParams>
```

上传配置，创建集合，并索引示例数据。

```
/opt/lucidworks-hdpsearch/solr/server/scripts/cloud-scripts/zkcli.sh -zkhost horton0.example.com:2181/solr -cmd upconfig -confname films -confdir /opt/lucidworks-hdpsearch/solr_collections/films/conf
 
curl --negotiate -u : "http://horton0.example.com:8983/solr/admin/collections?action=CREATE&name=films&numShards=1"
 
curl --negotiate -u : ' http://horton0.example.com:8983/solr/films/update/json?commit=true' --data-binary @/opt/lucidworks-hdpsearch/solr/example/films/films.json -H 'Content-type:application/json'
```

查询单个文档：
```
curl --negotiate -u : http://horton0.example.com:8983/solr/films/select?q=*&start=0&rows=1&wt=json
```

结果如下：

```json
{
   "id":"/en/45_2006",
   "directed_by":[
      "Gary Lennon"
   ],
   "initial_release_date":"2006-11-30",
   "genre":[
      "Black comedy",
      "Thriller",
      "Psychological thriller",
      "Indie film",
      "Action Film",
      "Crime Thriller",
      "Crime Fiction",
      "Drama"
   ],
   "name":".45",
   "_ttl_":"+10DAYS",
   "expire_at":"2016-11-06T05:46:46.565Z",
   "_version_":1549320539674247200
}
```

# Ranger审计日志的管理

Ranger的审计日志可以保存在自定义的SolrCloud中，也可以保存在由Ambari Infra提供的SolrCloud中。

Ambari Infra是一项新服务，包括其自己的Solr实例，例如 存储Ranger审核或Atlas详细信息。 由于HDP 2.5审计日志已经正式由数据库转移到Solr。 在Ranger审核中，Solr（以及DB）只是一个短期存储，基本上它仅用于Ranger Admin UI中显示的审计信息。 审计长期存档应存储在HDFS或类似的内容中。

默认情况下，Ranger Solr Audit Collection附带预配置的TTL，因此Solr中的所有Ranger Audits将在90天后立即被删除。

如果您只想将审核日志存储30天或一周，会发生什么？ 看看下面的段落:)

## 全新安装-Solr审计日志处于关闭

如果您以前没有使用过Solr Audits，还没有通过Ambari启用Ranger Audits to Solr，那么很容易调整TTL配置。 转到您的Ranger Admin节点并执行以下命令：

```
sed -i 's/+90DAYS/+30DAYS/g' /usr/hdp/2.5.0.0-1245/ranger-admin/contrib/solr_for_audit_setup/conf/solrconfig.xml
```

之后，您可以访问Ambari并启用Ranger Solr Audits，将要创建的集合将使用新的设置。

审计样例：

```json
{
   "id":"5519e650-440b-4c14-ace5-c1b79ee9f3d5-47734",
   "access":"READ_EXECUTE",
   "enforcer":"hadoop-acl",
   "repo":"bigdata_hadoop",
   "reqUser":"mapred",
   "resource":"/mr-history/tmp",
   "cliIP":"127.0.0.1",
   "logType":"RangerAudit",
   "result":1,
   "policy":-1,
   "repoType":1,
   "resType":"path",
   "reason":"/mr-history/tmp",
   "action":"read",
   "evtTime":"2016-10-26T05:14:21.686Z",
   "seq_num":71556,
   "event_count":1,
   "event_dur_ms":0,
   "_ttl_":"+30DAYS",
   "_expire_at_":"2016-11-25T05:14:23.107Z",
   "_version_":1549227904852820000
}
```
过期时间已经自动变成了30天。

## 已经存在的一个Solr集群-Solr审计日志处于开启

如果您已经将Ranger Audit日志启用到Solr并且已经在Solr集合中收集了大量文档，则可以通过以下步骤调整TTL。 但是，重要的是要记住，这不影响旧文档，只会影响新文档。

转到托管Solr实例的Ambari Infra节点（再次，具有zkcli客户端的任何节点）

### 从zookeeper上下载solrconfig.xml

```
/usr/lib/ambari-infra-solr/server/scripts/cloud-scripts/zkcli.sh --zkhost horton0.example.com:2181 -cmd getfile /infra-solr/configs/ranger_audits/solrconfig.xml solrconfig.xml
```

### 编辑solrconfig.xml文件

```
sed -i 's/+90DAYS/+14DAYS/g' solrconfig.xml
```

### 上传solrconfig.xml文件到zookeeper上

```
/usr/lib/ambari-infra-solr/server/scripts/cloud-scripts/zkcli.sh --zkhost horton0.example.com:2181 -cmd putfile /infra-solr/configs/ranger_audits/solrconfig.xml solrconfig.xml
```

### 重新加载配置

```
curl -v --negotiate -u : "http://horton0.example.com:8983/solr/admin/collections?action=RELOAD&name=ranger_audits"
```

审计日志示例：

```json
{
   "id":"5519e650-440b-4c14-ace5-c1b79ee9f3d5-47742",
   "access":"READ_EXECUTE",
   "enforcer":"hadoop-acl",
   "repo":"bigdata_hadoop",
   "reqUser":"mapred",
   "resource":"/mr-history/tmp",
   "cliIP":"127.0.0.1",
   "logType":"RangerAudit",
   "result":1,
   "policy":-1,
   "repoType":1,
   "resType":"path",
   "reason":"/mr-history/tmp",
   "action":"read",
   "evtTime":"2016-10-26T05:16:21.674Z",
   "seq_num":71568,
   "event_count":1,
   "event_dur_ms":0,
   "_ttl_":"+14DAYS",
   "_expire_at_":"2016-11-09T05:16:23.118Z",
   "_version_":1549228030682988500
}
```

# 清空集合中所有文档

如果要从Solr集合中删除所有文档，以下命令可能会有所帮助：

```
curl -v --negotiate -u : "http://horton0.example.com:8983/solr/films/update?commit=true" -H "Content-Type: text/xml" --data-binary "<delete><query>*:*</query></delete>"
```
或者通过浏览器操作：
```
http://horton0.example.com:8983/solr/films/update?commit=true&stream.body=<delete><query>*:*</query></delete>
```

# 参考链接

- [https://lucene.apache.org/solr/5_3_0/solr-core/org/apache/solr/update/processor/DocExpirationUpdateProcessorFactory.html](https://lucene.apache.org/solr/5_3_0/solr-core/org/apache/solr/update/processor/DocExpirationUpdateProcessorFactory.html)
- [https://cwiki.apache.org/confluence/display/solr/Update+Request+Processors](https://cwiki.apache.org/confluence/display/solr/Update+Request+Processors)


