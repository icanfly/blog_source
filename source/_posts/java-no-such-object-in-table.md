title: no such object in table
date: 2015-04-07

tags: 
 - java

---

检查一下主机名配置,以及host文件或者DNS解析,可能Context.PROVIDER_URL需要域名而不能使用ip



把etc/hosts恢复成
```
127.0.0.1 localhost
```
试试