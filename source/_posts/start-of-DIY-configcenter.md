title: DIY配置中心系列（一）：起步
date: 2017-08-14
tags:
 - 配置中心
 - DIY
categories:
 - 原创文章

---

> 配置中心从开发到线上接入运行已经过去快半年时间了，目前配置中心整体运行非常平稳，达到当初的设计目的，这里才敢有勇气拿出来分享设计，毕竟一样东西拿出来与人分享是需要勇气和底气的。同时对自己的实践过程进行一些总结，希望自己有时间回头瞭望时会有新的认识和发现。

# 前言

对于一个可运行的程序来说，配置可以说是驱动它运行的灵魂。良好的配置能指导程序正确的运行，并产出符合期望的结果。对于现有的软件来说，配置可以是多种多样的，配置可以写在配置文件中由程序运行时读取，也可以在启动程序时通过命令行方式传入，配置也可以是各种环境参数由程序在运行时进行读取，甚至有些参数直接写死在程序代码中。

对于程序开发来说，程序中某个参数的值在各种部署环境下或者在某种情况下需要更改，那这种值就可以抽象成配置项。在没有引入配置文件之前，要对各种部署进行适应的方式就是修改源代码中相应值然后重新打包并部署，而如果将该值抽象成配置后独立于配置文件中，这样就能够独立于程序之外单独进行修改而不用重新编译整体程序。

<!-- more -->

# 常见传统配置方式

对于传统程序开发来说，往往存在以下几种常见的配置方式：

 ## 方式一：从应用中加载配置

 在传统的应用开发中，开发人员习惯于将配置放入到项目的相对路径下，如类路径resources下面等等，配置文件的内容随着线下各种环境的值进行修改，等待发布时再修改项目中的配置文件中的配置项为线上配置值。对于这种配置的方式，我们可以数数从软件的生命周期开始需要经过多少次修改，至少应该是包括以下几次修改：

 - 开发阶段

 开发阶段时，开发人员需要根据自己本机的开发环境的相应配置修改配置文件并打包编译，以确保正确的配置值能够在本地环境驱动程序运行。

 - 测试阶段

 测试阶段开始，测试人员或者开发人员需要修改测试环境对应的配置文件并打包编译，以确保正确的配置值能够在本地环境驱动程序运行。

 - 上线阶段

 在上线前，仍然需要手工修改线上的配置值然后再打包并部署。


 从上面可以看出此种方式缺点很明显：很容易出错！很容易在各种环境配置的切换中配置错误，修改越多，出错的概率就越大。

 ## 方式二：Maven多profile配置管理

 该配置方式充分利用的是maven工具的profile过滤替换功能。

 - 将应用的运行环境划分成不同的环境profile：开发/测试/预发/灰度/线上等等profile
 - 项目结构下分别建立各环境的配置文件，如：config_dev.properties/config_test.properties等
 - 通过maven打包时的profile过滤机制替换项目中的占位符为特定profile的配置值

 关于使用Maven profile机制打包的方法这里不作叙述，大家可以自行google。

 这种方式改进了方式一的缺点，让配置的修改各自有了归属，对开发环境的profile配置修改不会影响到测试环境的profile配置以及线上的配置。但是这样的配置是有缺点的：
 - 对于应用的开发有一定的阻碍，要想配置修改生效，必须每次都得重新指定profile并通过maven编译后部署，十分繁琐。
 - 对于线上的配置值，尤其是一些比较重要的、机密的配置值裸露在项目中，存在一定的安全问题。

 ## 方式三：Spring多profile配置管理

 该机制与上面的Maven的profile很像，但是机制确有所有不同：

 - 将应用的运行环境划分成不同的环境profile：开发/测试/预发/灰度/线上等等profile
 - 项目结构下分别建立各环境的配置文件，如：config_dev.properties/config_test.properties等
 - Spring配置文件新建profiles节点，并将不同profile的配置bean进行配置加载
 - 应用在启动时传入启用的profile名称，由Spring启用对应的profile

 这种方式相对于Maven Profile的方式来说增强的地方在于：修改配置后可以直接重启而不用经过编译的繁琐过程。
 但是这种方式仍然存在Maven Profile的其它缺点外还存在的一个问题是：它必须和Spring深度捆绑！

 ## 方式四：服务器配置覆盖替换

 项目中保留本地开发的默认配置文件，各环境（开发、测试、预发、灰度、线上）在应用服务器特定目录下放置配置替换文件，修改应用服务器启动脚本加载特定目录下面的配置文件进行替换覆盖。

 以Jetty部署为例：

 ```shell
 #应用环境配置
PROJECT_DIR='/data/www/java/${APP_DEPLOY_DIR}'
LOGGER_ROOT=${PROJECT_DIR}/logs
 
if [ ! -d $LOGGER_ROOT ];then
    mkdir -p $LOGGER_ROOT
fi
 
#Jetty
JETTY_HOME='/opt/jetty'
JETTY_LOGS=${LOGGER_ROOT}
JETTY_RUN=${JETTY_HOME}/run
#配置文件
JAVA_OPTIONS="-Dconf.file=file://${PROJECT_DIR}/conf/conf.properties -Dlogger.root=${LOGGER_ROOT}"
 
##JVM参数设置
JAVA_OPTIONS="$JAVA_OPTIONS -server -Xms4096m -Xmx4096m -XX:+UseConcMarkSweepGC
                 -XX:+UseCMSCompactAtFullCollection -XX:CMSMaxAbortablePrecleanTime=5000 -XX:+CMSClassUnloadingEnabled
                 -XX:+UseCMSInitiatingOccupancyOnly -XX:CMSInitiatingOccupancyFraction=80"
JAVA_OPTIONS="$JAVA_OPTIONS -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=${LOGGER_ROOT}/java.hprof"
JAVA_OPTIONS="$JAVA_OPTIONS -verbose:gc -Xloggc:${LOGGER_ROOT}/gc.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps"
JAVA_OPTIONS="$JAVA_OPTIONS -Djava.awt.headless=true"
JAVA_OPTIONS="$JAVA_OPTIONS -Dsun.net.client.defaultConnectTimeout=10000"
JAVA_OPTIONS="$JAVA_OPTIONS -Dsun.net.client.defaultReadTimeout=30000"
JAVA_OPTIONS="$JAVA_OPTIONS -XX:+DisableExplicitGC"
 
usage()
{
    echo "Usage: ${0##*/} [-d] {start|stop|run|restart|check|supervise} [ CONFIGS ... ] "
    exit 1
}
 ```

其中的-Dconf.properties=file://$PROJECT_DIR/conf/conf.properties定义了加载配置替换文件的路径。
当然要实现特定环境的配置文件替换还需要在应用中配置时加入类似如下的配置：
```xml
<bean id="propertyConfigurer"
     class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
   <property name="fileEncoding" value="UTF-8"/>
   <property name="systemPropertiesModeName" value="SYSTEM_PROPERTIES_MODE_OVERRIDE"/>
   <property name="ignoreResourceNotFound" value="true"/>
   <property name="locations">
      <list>
         <!-- 本地开发配置文件 -->
         <value>classpath:props/config.properties</value>
         <!-- 线上覆盖替换文件 -->
         <value>${conf.properties}</value>
      </list>
   </property>
</bean>
```

这样实现了配置文件的加载替换覆盖。

这种方式解决了上面几种配置方式的安全问题，同时也满足一部分开发的便利方式。
但是该方式引出了新的问题：配置管理的繁琐性，不同的环境需要不同的替换配置文件，环境越多，配置的替换文件越多，对线下线上的服务器应用配置维护带来了很大的难度。开发的生命周期中，需要对很多地方做修改工作，容易疏忽和遗漏，开发人员在开发、测试等线下环境需要自行登录到线下服务器进行配置的修改，繁琐且不易管理。

 ## 配置中心

# 什么是配置中心

基于传统配置管理上的问题，统一的配置管理平台呼之欲出，主要目的就是要彻底解决以上遇到的所有问题，使应用与配置相对独立分开，应用打包交付后放入一个指定环境服务器，即可自动感知并拉取远程配置中心中对应的配置，这种方式也为现在非常流行的基于docker的容器化交付体系提供了实现基础。

那什么是配置中心？为什么要实施配置中心？

配置中心提供了配置的统一规范化管理平台，提供统一的操作平台供开发，测试，运维各种人员使用，提供各种应用部署环境的配置集中修改及配置的分发操作，减少QA测试、运维的机械性地配置文件修改劳动。并维护一个所有人员的权限操作列表，以区分不同人员的权限范围。对任何配置的修改操作进行日志审计记录，随时可查询配置的变更历史信息。可以做到应用上线安装好应用服务器后不做任何修改放入应用包直接运行的效果，这也是应用容器化的最根本的前置条件。同时也做到了应用迁移的便利性，以前的应用扩容，运维人员需要COPY整个应用服务器及相关目录的结构和内容到新扩容机器，接入配置中心后，应用的扩容将会是直接分配新机器，扔入应用包，直接启动就可以，无需其它繁杂的应用配置工作。


# 配置中心体系实现的功能

它实现了：

- 支持对接入应用的定义
- 支持对各运行环境/数据中心的定义
- 支持对配置版本的定义
- 支持对灰度配置的定义
- 支持配置修改的权限管理及历史记录
- 支持全兼容的原有应用无痛接入（支持直接导入项目properties文件）
- 支持提供Restful API接口供其他语言开发SDK或者Agent
- 支持对Spring框架的全兼容


# 开源界的配置中心

在我们的配置开始我们DIY的配置中心开发之前，业界已经存在的以下几个开源的配置中心方案：

- 淘宝开源的Diamond 
- 360开源的QConf
- 百度开源的Disconf
- 携程开源的Apollo（在我们开始自研配置中心之前它并没有开源，我们也不知道有这个东西）

QConf是基于zookeeper实现的一款多语言的配置中心，整体架构采用配置中心+Agent的方式，它需要在各部署机器安装Agent以代理配置的获取，这给我们的实施带来了困难并没有得到采用。
Disconf支持的配置方式很多，看得让人眼花，这也许是放弃它的原因。
Apollo这个配置中心不得不说设计的方向和我们的完全一样，很多概念上和我们的配置中心设计思想都是一致的，当然实现手法上可能各有千秋。也许它早点开源，我们就会直接采用它的实现进行二次开发也说不定。
Diamond结构简单，设计精巧，整体完全的分布式高可用设计非常清晰，但是介于它的层次划分仍然存在一些小的问题，但是瑕不掩瑜，我们打算依照Diamond的架构设计依葫芦画瓢开干。

# DIY配置中心术语

配置中心一些术语的定义：

- 模块/应用（Module）
定义了接入的最小单位，从抽象意义上来说，它可以是一个应用，也可以是一个庞大应用的一个子模块，对于我们一般的应用来说，接入是以应用为单位申请新建，而对于像基础中间件这样的中间件业务来说，它就是配置中心中某个基础组件下接入一个新应用。

- 运行环境（Profile）
对于配置中心来说，定义了接入应用的运行环境：开发（dev）/ 测试（test）/ 预发布（stage）/线上（product），不同的运行环境肯定会有配置值的差别。
而对于服务器来说，会定义一个基于/etc/config.env的文件，文件内容形式大致如下：
```
#配置中心本地缓存目录
CONFIG_CACHE_DIR=/etc/config_center/cache
#配置中心本地容灾目录
CONFIG_LOCAL_DIR=/etc/config_center/local
#服务器运行环境设定，应用程序根据该值从配置中心拉取指定的环境配置信息初始化配置
RUN_ENV=test
#配置中心API服务器地址
CONFIG_SERVER=http://api.config.domain
```

- 配置版本（Version）
定义了运行环境的版本信息，允许多版本的并行运行，多版本的概念同一些分布式的组件如dubbo类似，提供了配置版本的平滑过渡。配置版本只是方便用于在线配置与应用不兼容升级变更的平滑过渡，不建议运行环境长期存在多个版本的并存，在配置过渡完成后可以删除失效的版本。

- 配置灰度（Gray）
这是一个很有用的功能，可以防止“手滑”党的很大一部分错误发生。通常一个应用下面会部署多台机器，而在配置中心对于配置的修改会准实时同步到应用机器。如果一个非常重要的可实时生效的配置项被“手滑”的修改成了一个错误的值，那么该应用下的所有机器将会应用该配置，那么产生的后果将会相当严重，轻则线上应用不能正常运行，重则整个应用集群宕机不可用。配置灰度的提出，主要是解决这种问题，配置灰度可以在现有的配置版本基础上创建一个灰度配置，灰度配置中需要指定应用该灰度的服务器信息（如：IP），只有符合该IP的服务器才会应用该灰度的配置，达到配置修改的小范围验证目的，当验证通过后可以通过“完成灰度”来将灰度配置应用到正式版本上完成配置的正式发布。

# DIY配置中心整体架构

{% asset_img image2016-8-24_14-37-34.png %}

# 总结

配置中心提供了配置的统一规范化管理平台，提供统一的操作平台供开发，测试，运维各种人员使用，提供各种应用部署环境的配置集中修改及配置的分发操作，减少QA测试、运维的机械性地配置文件修改劳动，大大提升了开发、测试、运维等人员的工作效率。上面大致描述了配置中心的作用和基本构成要素，在实现配置中心过程需要考虑的地方其实是很多的，在接下来的几篇文章中我将详细介绍实现整个配置中心过程中的一些细节。
