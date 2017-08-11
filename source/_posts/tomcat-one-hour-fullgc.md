title: 关于tomcat 开启gc日志后每隔1小时full gc的问题
date: 2014-08-20

tags:
 - tomcat
 - gc
categories:
 - 问题解析

---

关于tomcat 开启gc日志后每隔1小时full gc的问题

主要是因为rmi导致的，可以参见以下的博文：

http://www.iteye.com/topic/1121073

http://hllvm.group.iteye.com/group/topic/27945

http://docs.oracle.com/javase/6/docs/technotes/guides/rmi/sunrmiproperties.html