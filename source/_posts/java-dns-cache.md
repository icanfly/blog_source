---
title: Java DNS缓存
author: LP's Notes
tags:
  - java
categories:
  - 转载文章
thumbnail: /images/java.png
blogexcerpt:
---

# jdk1.5和1.5之前版本

默认DNS缓存时间是永久缓存

# jdk 1.6以后

与security manager策略有关。

如果没有启用security manager，默认解析成功的DNS缓存时间为30秒，解析失败的DNS缓存时间为10秒。

策略配置文件：JAVA_HOME/jre/lib/security/java.policy

# 注意事项

对于多条A记录DNS，在缓存有效期内，取到的IP永远是缓存中全部A记录的第一条，并没有轮循之类的策略。
缓存失效之后重新进行DNS解析，如果每次域名解析返回的A记录顺序会发生变化，缓存中的数据顺序也会发生变化，取到的IP也变化。

# 缓存修改方法

1.	jvm启动参数里面配置-Dsun.net.inetaddr.ttl=value
2.	修改配置文件$JDK_HOME/lib/security/java.security相应的参数networkaddress.cache.ttl=value
3.	代码里直接设置：java.security.Security.setProperty(”networkaddress.cache.ttl” , “value”);
