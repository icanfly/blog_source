title: nginx error_page配置
date: 2013-08-01

tags:
 - nginx
categories:
 - 服务器
---

今天偶然访问了一个线上应用不存在的url，应用报错，出现了乱码。

乱码是从nginx转发的tomcat报出来的。tomcat默认处理HTML是以ISO-8859-1处理的，所以就产生了乱码。

解决这个error_page的途径我尝试了两种方法：

1、让tomcat返回正常的非乱码的error_page
tomcat的错误页是在项目的web.xml中配置的，但是除了这个之外，别无其它编码配置。在网上搜索了有人提现将.html这种页面也交由jsp servlet处理就好，我认为这种方式不好，所以直接没尝试。
我配置的web.xml如下：

```xml
<error-page>
    <error-code>500</error-code>
    <location>/error.html</location>
</error-page>
```

那么首先想到的就是把error.html页的返回头改掉：
```
<meta http-equiv="content-type" content="text/html; charset=UTF-8"/>
```
但是改后，不幸的是还是不行！
tomcat还是把它处理成ISO-8859-1了。杯具！

2、第二种途径是不管tomcat返回的错误页，直接使用nginx的错误页
这里要注意一件事就是一定要配置nginx这个选项：proxy_intercept_errors on;

这个选项默认在nginx是off的。所以这时候你配置的所有error_page错误页都不会生效。为此我查了好久才知道是这个原因。

我的配置：
```xml
location   / {
    proxy_set_header Host $host;
    proxy_set_header  X-Real-IP  $remote_addr;
    proxy_set_header X-Forwarded-Proto  $scheme;
    proxy_set_header X-Forwarded-For $remote_addr;
    proxy_pass http://127.0.0.1:8080;
    proxy_intercept_errors on;
}
```
