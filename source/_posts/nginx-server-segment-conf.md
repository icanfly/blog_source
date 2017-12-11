title: nginx server节配置
date: 2014-08-06

tags:
 - nginx
cateogries:
 - 转载文章
thumbnail: /images/nginx.png
---

```xml
server {
    listen          80;
    server_name     *.test.com default;
    root                    /home/a/share/htdocs;
    index                   index.html index.htm;
    location   / {
        proxy_set_header Host $host;
        proxy_set_header  X-Real-IP  $remote_addr;
        proxy_set_header X-Forwarded-Proto  $scheme;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_pass http://127.0.0.1:8080/test/;
    }

#    rewrite ^/(.*)$     http://www.test.com/abc.htm$1;
}
```
server_name节点表示从哪个域名过来，nginx里可以配置多个server节点以支持不同域名的转发需求。

default的意思是如果所有的server节点都没有匹配，那么就使用这个default节点匹配了。

index节点表示如果域名后没有带任何的地址信息，则默认访问的页面，一般应用会以index.html展现。

location节点可以根据正则表达式进行配置，以满足不同路径的转发规则。

proxy_set_header 节点主要是将请求的原始信息附加到nginx的转发上，从而能让后端服务（如tomcat)能够获取到最原始的客户端信息，而不是nginx转发端的信息

rewrite 节点是重定向，当访问匹配的正则表达式路径时会被重定向到相应的地址上。
