title: 关于apache http转发后后端应用获取前端port问题
date: 2015-05-21

tags:
 - apache
 - nginx
 - http

---

apache+jetty转发配置下，jetty下应用获取request.getServerPort()获取到的是jetty的端口，而非apache入口的端口，情形如下：


apache通过配置虚拟主机：

```xml
<VirtualHost *:80>
     ServerName "admin.test.com"
     ProxyRequests Off
     ProxyPass / http://localhost:6808/
     ProxyPassReverse / http://localhost:6808/     
</VirtualHost>
```

在80端口接受外界访问，然后转发到端口6808上。

<!--more-->

但是在6808端口上的应用在获取request.getServerPort()时获取到的是6808，而非80,对于这种情况在构造自引用地址时会出现一些问题：

```java
String basePath = request.getScheme()+"://"+request.getServerName()+":"+request.getServerPort();
```

这块代码一般会引用在前端的JSP页面或者程序需要进行URL地址引用的时候。这个时候应用端口就不正确了。

一般的解决办法：

1、修改basePath的获取方式，改成配置文件方式，直接读取配置文件，但是这种方式不是特别方便，不推荐使用

2、使用反向代理服务器配置

**nginx配置方式**
```xml
proxy_set_header host $host:$port
```

**apache配置方式**
```xml
<VirtualHost *:80>
     ServerName "admin.test.com"
     ProxyRequests Off
     ProxyPass / http://localhost:6808/
     ProxyPassReverse / http://localhost:6808/
     ProxyPreserveHost On
</VirtualHost>
```