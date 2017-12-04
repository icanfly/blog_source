title: Nginx_Session_Sticky踩坑记录
date: 2017-09-12
tags:
 - nginx
 - 踩坑
 - session
categories:
 - 原创文章

---

# 常见Session方案

一个多用户的WEB系统一定离不开多用户的登录和会话保持的问题，用户登录可以通过SSO单点登录解决，但是用户的SESSION会话保持是需要一个基础设施来支撑的。对于传统的单机部署的WEB应用，SESSION会话由本机的应用服务器（tomcat/jetty/jboss）负责应用SESSION会话的保持。但是对于分布式部署的WEB应用来说，单机的会话保持显然并不能适用在这种场景下面，下面是分布式WEB应用场景时一般采取的策略：

- 方案一：通过前端负载均衡进行SESSION_STICKY
- 方案二：应用服务器层SESSION同步
- 方案三：应用SESSION层统一管理

<!-- more -->

1、对于方案一，操作简单，应用不需要进行任何设置，同一用户首次访问应用提供服务的某台服务器后，后续的访问请求会一直发往该台服务器进行处理，这里解决的就是SESSION本机存储的问题，如果采用的是SESSION本地存储，然后请求又是在多台机器之间分发，那么会造成用户不断的登录和退出，无法正常使用应用。这种方案一般在负载均衡服务器(Nginx/Apache等）上进行配置即可。

2、对于方案二，操作方式稍微比方案一要复杂，但是仍然向使用者屏蔽了使用细节。对于该方案要求后端的真实应用服务器之间要建立同步通道进行SESSION会话数据的同步，这种方案在后端真实服务器相对较少时可以采用没有太多问题，但是一旦服务器数量以及用户数量并发超过一定数量会造成网络风暴的问题。试想，一个用户的SESSION在某台服务器进行修改后要同步到所有其它服务器的SESSION管理存储上，这是一个1+N的过程，性能和网络都会无法承担这样的开销。

3、对于方案三，这是目前大型互联网公司采用的方案,如淘宝，同样对使用用户屏蔽实现细节，用户使用过程中就好像使用原生的SESSION一样。方案三采用集中式SESSION存放，应用服务器并不负责用户SESSION的管理，SESSION管理交由统一封装的SESSION框架层负责处理，SESSION框架层拦截应用服务器的SESSION存取并与SESSION存储交互获取和设置数据，这里的SESESSION的存储又分为多种方式，常见的有：缓存服务，Cookie等。一些不重要的，非关键性的用户数据可以通过SESSION框架存入Cookie中，而重要的用户数据存入远程的缓存服务中。

# 问题现象

我们这里有一个应用，线上会部署多台服务器，当时为了方便快速上线就采用了上面方案一的方式，在线上Nginx上配置了session_sticky，然而这正是问题的始源：

通过我们自己的APM监控系统发现在最初后端的两台服务器正常的各自分担了50%的网络流量，但是在后面的一段时间里流量会慢慢的向基中一台机器聚集，而另一台机器流量几乎降低到微乎其微。这个问题困扰了好几天，前面几天一直发现了该问题，但是一直忙于处理其它事务，今天终于有时间慢慢来分析这方面的问题。

# 问题排查

1、首先查看的Nginx的配置是否正确

{% asset_img FD684B9B-728E-45AA-9566-598BE2487350.png %}

采用了SESSION_STICKY，并且两台服务器间采用一样的权重比率。没问题。

2、排查用户访问IP的问题

最开始一直认为Nginx的SESSION_STICKY是通过用户的IP进行的分流（其实后面证明我的想法是错的），所以想到的是查找用户访问的IP，通过询问运维，得到的结论是用户都是通过内网统一一个IP访问，这里有一个误导，导致我认为这就是导致该问题的原因。如果真是按IP对用户进行分流切分的话，那如何解释之前可以平均分配流量的问题呢？我一直在不停的反问我自己。

3、在多个不同用户的机器上重现问题

我使用了多个同事的电脑进行操作以及查看资料，发现Nginx的SESSION_STICKY是通过Nginx反写cookie实现的，通过查看多个同事的浏览器cookie，我发现了这个cookie：route=739d4e2d09f01c606bc43936e6e743e3; 基本上是所有的同事浏览器cookie都是一样的，这也应验了为什么基本上的流量都会往一台机器上发送了。

# 问题分析

既然有了上面的问题排查，那么最重要的一个问题就落在了为什么不同的用户会产生同样的cookie呢，我试着将我自己的电脑上的cookie清空，然后再重新登录，再查看该cookie值。重复这样几次后，我发现均衡正常了，可以按一定的比率会话分别粘滞在两台机器上。同时我也仔细翻看了Nginx的SESSION_STICKY说明，其中有一条也让我恍然大悟：Nginx SESSION_STICKY产生的cookie是根据配置按后端可用的upstream服务器中的一台的IP通过MD5加密（或明文，可配置）后得到的一串数字，而并非是由前端的IP决定。

NGINX SESSION_STICKY 原理：

{% asset_img sticky0.jpg %}

{% asset_img sticky1.png %}


这里导致上面的问题的原因慢慢的开始浮出水面：

1、为什么Cookie是一直不变的？

原因是大部分的同事都是使用笔记本，特别是大部分人都是MacPro控，所以对于他们来说，工作或者下班时是重来不需要关闭电脑的，电脑一合就走人，所以浏览器是一直打开的未关闭过，这种情况也在部分使用台式机的同事存在，也是下班电脑睡眠就走人，并未关闭浏览器。对于route这个cookie是浏览器关闭才会失效，所以一直开启的浏览器时该cookie会一直有效。

2、为什么基本上的同事的Cookie都是一样的？

这个问题就要从应用的发布说起了。我们的发布流程是灰度发布过程，在应用发布时是一台一台的发布的，总共两台机器，其中第一台的发布的过程中大量的请求被定向到另一台机器，而第一台发布完成时流量并不会切换回来，因为新产生的cookie已经是第二台机器的cookie，并且该cookie是一直有效的，除非有人为的手动关闭整个浏览器。这就解释了为什么应用在第一次上线时是流量均衡的，但是一旦后面上线过后流量变成只向其中一台聚集的情况，这也是使用Nginx Session_Sticky的一个问题。

# 问题解决

问题解决方案其实已经在上面第一段内容提及了，实现会话保持的三个方案中，选择其中一个，对于我们之前使用的第一种方案，其实有相应的缓解方式，就是增开几台机器，让流量在发布的时候也分布到其它机器，只是在我们的场景下，只有两台机器，当发布进行时，所有的流量都汇聚到其中一台上，以后也固化到这台访问了。如果将机器扩充多一些，那么在一台机器发布时，流量会分担到其它机器，整个集群相对来说还是比较均衡的（其中只有一台没啥流量，相当于是浪费一台机器），能保证整体流量在N-1台机器上均衡（N为应用机器总量）。

我们解决的方式是使用方案三，使用外置的Session会话存储的方案。这里我们可以自己写一套Session管理的框架，但是介于开源世界已经有实现方案了，比如Spring的session方案。于是我们的方案就是基于此来做改造。我们使用spring session框架，基于redis集群的会话保持方案改造了自己的应用，非常简单快速的实现了session的外置存储支持，以及应用的无状态化。基于此，nginx的session_sticky配置也可以去掉了，应用也可以很好的扩容了。