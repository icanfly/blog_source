title: Linux自动同步时间
date: 2013-12-10

tags:
 - linux
categories:
 - 转载文章
thumbnail: /images/linux.png
---

 1、 一般的Linux发行版都带有ntpdate这个命令，如没有，可以从其它发行版中拷贝一个/usr/local/bin
 2、#crontab -e
 添加
 ```
 */10 * * * *    /usr/local/bin/ntpdate  time.nist.gov> /dev/null 2&1
 ```
 每10秒执行一次ntpdate，当然ntpdate也可以用脚本代替，这样可以更加灵活 time.nist.gov 是一个时间同步服务器

 3、
```
 /etc/rc.d/init.d/crond restart
```
