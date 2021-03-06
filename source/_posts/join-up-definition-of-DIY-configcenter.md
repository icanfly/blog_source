title: DIY配置中心系列（二）：接入定义
date: 2017-08-16
tags:
 - 配置中心
 - DIY
categories:
 - 原创文章
toc: true
---

# 前言

对于配置中心主控台的设计，其实我设计的原则是设计成一个统一管理元数据（应用接入、环境定义、版本定义、人员权限划分、配置管控等）功能的一个综合平台。对于一个应用接入配置中心来说，很自然想到需要一些实体来承担配置的管理载体。主控台是用户使用操作最为频繁的地方，良好的组织结构与交互设计能够给人带来办事效率上的提高，所以这块我也是经过了很多次版本的设计与修改，最终产出的是一个类似树型的结构体系。

<!-- more -->

# Module定义

对于一个接入来说，我们配置中心平台需要知道该接入的一些元数据信息，比如该应用的名称是什么，它是由谁负责，联系方式是什么，是归属于哪个部门，是什么类型的应用等等，这些都是属于接入定义的范畴。这些属性当中有的我们可以通过其它外围系统获取，而有些则需要在新建接入的时候录入。

我们的配置中心除了希望应用能够很好的接入以外，还希望能够接入一些自研的基础组件的配置。所以我们在定义这个载体的时候并没有直接采用application这个名称，而是使用了module这个名称。这个名称涵盖了应用接入和基础组件的接入，具体这两种接入的区别以及不同点，稍后会有介绍。


Module定义的主要元数据如下：

 - 模块名称
   这里是一个接入的唯一标识，由英文字母以及一些数字、下划线组成
 - 描述
   描述该接入的一些其它方面的描述
 - 接入语言
   多语言接入选择接入语言，如Java/NodeJS/PHP等
 - 管理员设定
   设定该模块的管理员，模块的管理员对该模块的配置授权有决定作用
 - 归属部门
   设定该模块归属于哪个部门



# Profile定义

在配置中心中，环境的术语名称叫Profile，这其实是一个非常标准化的称呼。在国外的很多网站上，个人设置其实就是用的这个Profile词汇；而在Spring开发中，对于不同类型的配置的区分也是通过叫做Profile的选项进行区分。所以这里我们也用Profile这个词来指代环境的定义。

传统的开发过程中，系统在各个不同的环境中是部署了多套系统，比如对应开发、测试、预发布和线上的环境，软件系统在各个环境都部署一套。这样做的好处是可以做到系统的环境的完全隔离，但是代价就是系统部署的量级随着环境套数的变化成指数增长。新定义一套环境是比较容易的，但是新建一套环境对应的系统部署则是非常麻烦的事情。

## 环境划分

配置中心在这里采用的是另一种环境划分方式： 将环境划分成线上环境和线下环境。

 - 线上环境: 所有的部署于线上的系统连接线上的配置中心部署，并与线下完全隔离，做到安全隔离。线上的环境可以自己进行定义，目前定义了两套环境：灰度环境(stage)、正式环境（product），当然可以增加更多的环境定义。
 - 线下环境: 所有的部署于线下的系统连接线下的配置中心部署，线下的配置中心定义的环境有：开发（dev）、测试（test）、预发布（pre）等。

这样的环境定义后，为我们后面部署配置中心打下了基础，我们部署的配置中心将只需要部署两套系统（因为考虑安全的因素，线上与线下完全隔离）。不必像传统软件系统那样，新增一套环境则需要新增加一套软件部署。

Profile定义的主要元数据如下：

 - 环境名称
   定义了环境的英文名称，该环境名称也包括了系统内置的一些预定义的环境名称，系统标准内置了dev/test/pre/stage/product等一系列的标准环境定义，也可以自己定义新的环境名称
 - 环境描述
   描述环境名称以外更多的描述
 - 环境类型
   环境类型用以区分是内置类型还是自定义类型
 - 所属模块
   定义了该环境是属于哪个模块的，


# Version定义

对于Version这个版本来说，如果大家使用过Dubbo，一定会不对它里面的Version的概念陌生，而我们这里的Version概念与之差不太多。Version主要用于多版本配置的并行运行或者新旧滚动发布。对于Dubbo中Version的主要使用方式就是在涉及到不兼容升级接口时，在滚动升级过程中同时并存新旧版本的服务，让接口消费方有一定的时间窗口逐步迁移至新的接口服务中来。当所有的旧接口调用都迁移到新接口调用中后，老版本的接口或者服务器就可以完全下线了。

在配置中心中，Version同样用于多版本并存、滚动更新这样的场景。当一个应用在某次的功能修改中发生了非常大的变更，而该变更对于当前正在线上运行的应用来说是不兼容的，也就是我们在上线操作过程中不能直接对线上的配置进行修改，因为这样直接的不兼容修改会直接反馈到线上运行应用，导致线上应用出现故障。最稳妥的做法就是新建一个版本，在应用中依赖该版本的配置进行发布，上线后线上的运行程序就会依赖新版本的配置，旧版本的配置将会失效，这时我们就可以安全的删除旧版本的配置了。

同时Version因为是树型结构的最底层，所以也充当了配置数据存储的角色。所有的Module-Profile-Version形成的坐标都对应了一组的配置，该配置将以JSON的形式存储在Version的元数据中

Version的主要元数据如下：

- 版本名称
  在Module-Profile的路径下唯一确定一个版本信息
- 所属Profile
  归属的环境信息
- 所属Module
  归属的接入定义
- 配置内容（JSON）
  配置内容，内部以JSON格式组织，配置存储的结构将在后面文章中再详细介绍
- 数据版本
  用户编辑配置的数据版本演进，该属性对于配置的顺序应用有非常重要的作用
- 配置指纹
  对配置内容进行的一个指纹签名，该属性对于后续SDK接入后配置的对比更新有非常重要的作用
- 描述
  对该版本的产生原因进行一些说明

# 小结

通过上面的描述，Module/Profile/Version共同组成了一个三维坐标标记了一组配置。我们可以很快的画出一个带树状的组织结构：

- 应用接入型



                      Module                                                  ------> Module
                        |
        -----------------------------------------------------------
               |                  |               |        |
              Dev                Test             Pre      ...                ------> Profile
               |                  |               |
            -----------------     -----------     ------------------
            |         |     |     |         |      |         |    |
           default    v2   ...    default   v0.1   default   v10  ...         ------> Version




# 问题

以上Module/Profile/Version的划分并非没有问题，这种划分很好的解决了以应用为维度的接入，但是对于另一个以基础组件为维度的接入却显得比较困难。

## 基础组件如何接入

对于一层的Module定义来说，可以很好的满足应用接入的需求，但是对于基础中间件的接入来说，它涉及到的维度就有两层了，一层是中间件本身的接入定义，另一层是使用该中间件的应用接入定义。怎么理解这两层含义呢？我们来举个例子：

在现有的基础之上，假如我们要接入一个消费中间件MQ的组件SDK，该SDK在初始化以及运行过程中需要一些动态可调的配置，现在我们把它接入配置中心。那我们就在主控台上新建一个Module，取名我们就定为MiddleWare-MQ，好了现在定义好了一个Module，该Module下分各种环境及配置信息。那么问题来了：

 - 不同应用接入的中间件配置有可能不同，并不是一个大一统的配置
 - 不同应用接入的中间件配置需要该应用的负责人有权限修改配置，即接入配置中心的中间件各接入方要有权限能够修改自己的接入配置

以上两点来说，刚才我们以应用接入的设计并不能很好的满足。

### SubModule定义

基于此现状，对基于应用接入的结构进行了扩展，将Module层进行扩展，支持多层Module父子关系（目前两层已经足够）：

 - 第一层定义接入中间件属性，该层属于公共层，由接入方（中间件团队）维护，这层下面不直接挂接Profile，而是挂接SubModule。
 - 第二层定义接入方属性，该层属于个性化层，由接入方（中间件团队）新建和维护，同时将配置编辑权限授权接入业务方，让业务方有权参与中间件配置编辑。该层下面才开始挂接Profile。

这里需要说明一下，第二层的新建建议是由中间件基础组件团队来执行，而执行前是需要业务方通过申请基础组件接入，由基础组件团队新建完第二层后将该层编辑权限授予业务方。

这样经过扩展之后整体的树型结构变成这样：

- 基础组件接入型




                      Module                                                  ------> Module
                        |
            --------------------------------------------
            |             |            |             |
           subModule1   subModule2    subModule3    ...                      ------>  SubModule
                          |
                          |
                          |
        -----------------------------------------------------------
               |                  |               |        |
              Dev                Test             Pre      ...                ------> Profile
               |                  |               |
            -----------------     -----------     ------------------
            |         |     |     |         |      |         |    |
           default    v2   ...    default   v0.1   default   v10  ...         ------> Version


那么这两种树型融合后的结果就是：Module分两种类型(type)，一种是下面直接挂载Profile，而另一种是下面还需要挂载SubModule。那么这时Module的主要元数据就需要增加至少两个属性：

 - type        用以区分该Module是哪种类型
 - parent      用以标识该Module的父Module是谁，当type为应用接入类型时，该module的parent自然就为空了


最新Module定义的主要元数据如下：

 - 模块名称
   这里是一个接入的唯一标识，由英文字母以及一些数字、下划线组成
 - 类型
   用以区分该Module是哪种类型
 - 父Module
   用以标识该Module的父Module是谁
 - 描述
   描述该接入的一些其它方面的描述
 - 接入语言
   多语言接入选择接入语言，如Java/NodeJS/PHP等
 - 管理员设定
   设定该模块的管理员，模块的管理员对该模块的配置授权有决定作用
 - 归属部门
   设定该模块归属于哪个部门

# 总结

Module/Profile/Version共同组成了一个三维坐标标记了一组配置，它的组织结构形如以下结构：

- 应用接入型



                      Module                                                  ------> Module
                        |
        -----------------------------------------------------------
               |                  |               |        |
              Dev                Test             Pre      ...                ------> Profile
               |                  |               |
            -----------------     -----------     ------------------
            |         |     |     |         |      |         |    |
           default    v2   ...    default   v0.1   default   v10  ...         ------> Version


- 基础组件接入型




                      Module                                                  ------> Module
                        |
            --------------------------------------------
            |             |            |             |
           subModule1   subModule2    subModule3    ...                      ------>  SubModule
                          |
                          |
                          |
        -----------------------------------------------------------
               |                  |               |        |
              Dev                Test             Pre      ...                ------> Profile
               |                  |               |
            -----------------     -----------     ------------------
            |         |     |     |         |      |         |    |
           default    v2   ...    default   v0.1   default   v10  ...         ------> Version


除了这种树型结构，在后面我们还引用了`灰度控制`的概念，以便于在变更配置项时能先验证后发布的过程，极大的降低了因错误地修改配置而导致的故障出现概率。

这种结构可以很好的适应各种部署问题。比如也可以把Profile抽象成机房IDC的概念，那么假如现在线上有两个数据中心，北京(bj)和上海(sh)，那我们完全可以在所有数据中心中只部署一套配置中心，仅仅使用profile就可以实现多数据中心的配置管理，如profile可以定义成product-sh，product-bj等等。

当然所有数据中心只部署一套配置中心就需要解决如何高效部署的问题，这个点我会在后面的文章中描述。
