title: Fastjson的一个BUG
date: 2016-01-17 20:20:00

tags:
 - fastjson
 - bug
categories:
 - 问题解析

---

项目中使用的fastjson版本为：1.1.41，今天突然在重启线上服务器后莫名出现异常，而这个异常以前重来没有出现过，这个异常类似这样：
```java
com.alibaba.fastjson.JSONException: write javaBean error
    at com.alibaba.fastjson.serializer.JavaBeanSerializer.write(JavaBeanSerializer.java:212) ~[fastjson-1.1.41.jar:na]
    at Serializer_6.write1(Unknown Source) ~[na:na]
    at Serializer_6.write(Unknown Source) ~[na:na]
    at com.alibaba.fastjson.serializer.JSONSerializer.write(JSONSerializer.java:369) ~[fastjson-1.1.41.jar:na]
    at com.alibaba.fastjson.JSON.toJSONString(JSON.java:418) ~[fastjson-1.1.41.jar:na]
    at com.alibaba.fastjson.JSON.toJSONString(JSON.java:568) ~[fastjson-1.1.41.jar:na]
 ...
 ...
 ...
at java.lang.Thread.run(Thread.java:745) [na:1.7.0_79]
Caused by: java.lang.ClassCastException: com.google.common.collect.Lists$TransformingSequentialList cannot be cast to com.xxx.common.dto.pager.PagerData
    at Serializer_9.write1(Unknown Source) ~[na:na]
    at Serializer_9.write(Unknown Source) ~[na:na]
    at com.alibaba.fastjson.serializer.ObjectFieldSerializer.writeValue(ObjectFieldSerializer.java:115) ~[fastjson-1.1.41.jar:na]
    at com.alibaba.fastjson.serializer.ObjectFieldSerializer.writeProperty(ObjectFieldSerializer.java:68) ~[fastjson-1.1.41.jar:na]
    at com.alibaba.fastjson.serializer.JavaBeanSerializer.write(JavaBeanSerializer.java:194) ~[fastjson-1.1.41.jar:na]
    ... 66 common frames omitted
```

百思不得其解，因为我返回的对象中根本就没有com.xxx.common.dto.pager.PagerData 这个对象信息，为什么在序列化的时候会出现这个错误呢，非常怪异。让人十分摸不着头脑的是这个错误是重启服务器后发生。毕竟线上一直在报错，当时情急之下的解决办法试了两个方法：
1. 再次重启服务器
   重新启动服务器几次，错误依然，仅仅一次代码的小调整（根本和报错的问题风马牛不相及），但是启动服务器就报这个错，给跪了！
2. 赶紧找其它json类库暂时替代
   json框架毕竟我还是熟悉几个的，情急下只能仓促使用Gson（google出品的json框架）临时替代了fastjson的json序列化输出，问题解决！！！

<!--more-->

然后就是走上了寻找问题之路，找到fastjson的github网站，在[issue列表](https://github.com/alibaba/fastjson/issues?utf8=%E2%9C%93&q=ClassCastException)中经过一些查找搜索终于找到一个issue和我遇到问题非常相像：
1. [issue 60](https://github.com/alibaba/fastjson/issues/60)
2. [issue 107](https://github.com/alibaba/fastjson/issues/107)
情形和我的基本一致，而且我使用的fastjson版本也刚好落在他们描述的bug版本区间。
fastjson的开发者wenshao回复在1.1.42修复

我于是赶紧到Maven库查看fastjson版本，不看不知道，一看版本已经演进了好多，最新版本已经是1.2.7，更新代码maven依赖至1.2.7，然后测试发布到线上，多次重启确定没有再出现强制类转换异常。

关于这个错误的造成原因，还没有深入去了解是什么原因造成的，因为这个异常有一定的偶然性，可验证性比较差，线上服务器也是偶发出现。代码提交也没有明确是在哪次提交的时候修复了这个bug，等后面有时间再看看这个问题。
