title: Fastjson关于autoType is not support问题解析
date: 2017-05-12
tags:
 - fastjson
 - 问题解析
categories:
 - 原创文章

---

在做大数据查询系统的过程中，需要对Presto对正在进行的查询进行管理，允许用户进行kill。今天有用户反馈线上的任务无法kill，通过查看线上日志得知在解析json的时候报了一个错：
```
com.alibaba.fastjson.JSONException: autoType is not support. output
   at com.alibaba.fastjson.parser.ParserConfig.checkAutoType(ParserConfig.java:888)
   at com.alibaba.fastjson.parser.DefaultJSONParser.parseObject(DefaultJSONParser.java:325)
   at com.alibaba.fastjson.parser.DefaultJSONParser.parseObject(DefaultJSONParser.java:520)
   at com.alibaba.fastjson.parser.DefaultJSONParser.parse(DefaultJSONParser.java:1335)
   ...
```

从上图可以看出错误代码出在了：
```
QueryVO query = JSON.parseObject(result, QueryVO.class);
```
根据错误堆栈，错误发生在323行：

{% asset_img DefaultJSONParser.png [DefaultJSONParser] %}

<!--more-->

即在这里的判断如果json的key的内容等于‘@type’，并且并没有禁用DisableSpecialKeyDetect这个关键字检测，则会进行特殊类型反序列化处理，也就是325行，在这里fastjson自己定义了一个特殊的关键字‘@type’用于保留反序列化时类型信息，而我们返回的json内容中恰好就含有这样的关键字，fastjson当成了自己的预定义解析类型进行解析，故会报出刚才的错误。



根据以上代码的提示，可以在进行JSON解析的时候，将Feature.DisableSpecialKeyDetect传入解析器中，设置禁用关键字解析。

```java
QueryVO query = JSON.parseObject(result, QueryVO.class, Feature.DisableSpecialKeyDetect);
```

如此便解决了问题。
