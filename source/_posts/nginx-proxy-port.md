title: Nginx负载均衡重定向问题
date: 2015-04-10

tags:
 - nginx
 - 负载均衡
categories:
 - 原创文章
thumbnail:
---

当负载端口不是80时，发现所有 response.sendRedirect() 重定向的页面都返回80端口，后来发现是代理设置Header时没有指定Ngnix监听的负载端口

#设置被代理服务器的端口或套接字，以及URL

```xml
proxy_set_header Host $host:6112;
```
