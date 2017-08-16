title: 修复ambari presto插件在重启时的BUG
date: 2017-08-09
tags:
 - ambari
 - presto
 - 大数据
 - bug
 - 参与开源
categories:
 - 原创文章

---

# 前言

随着商业数据的不断累积与爆发式增长，传统的数据存储已经不能很好的满足日益增长的数据系统需要，传统的通过关系型数据进行数据获取的方式正在随着数据量的快速增长而出现了瓶颈，随着以Hadoop为代表的大数据平台的出现，有效解决了数据高速增长带来的数据存储和管理问题。现在大数据平台基本上每家公司都发展中长期必备的基础设施，公司基础数据平台使用的Hortonworks提供的大数据平台发行版，使用ambari进行数据平台的管理工作。前段时间大数据平台引入的Presto数据查询引擎作为即席查询工具，并使用ambari对presto的插件支持进行管理。

Ambari是由Hortonworks公司出品一款大数据平台管理软件，它将大数据平台所需要的各种组件进行统一管理，并提供大数据平台主机管理及各组件的安装制定、配置、启停、监控等一系列非常方便的功能。同时它也支持用户自定义组件插件，提供了良好的扩展能力。

<!-- more -->

{% asset_img 432E0D4D-7C6B-4374-8FCB-AE10016546D1.png %}

# 问题

Presto官方为Ambari提供了一个presto的插件，插件地址：[Ambari-Presto-Service](https://github.com/prestodb/ambari-presto-service)，该插件遵循Ambari的插件体系规范，可以集成到Ambari提供Presto的管理功能。

官方提供了一个Presto的Ambari插件集成[文档地址](https://prestodb.io/ambari-presto-service/getting-started.html#)，指明了如何将插件集成到Ambari中，该文档有点旧，只能是对照着大致看下，帮助理解。

官方的Presto存在在Ambari界面上对Presto进行重启会失败的问题,该问题由几个小问题组成，我都对其进行了修复。

> 多提一句的是：官方对于这个Ambari的插件的维护似乎并不上心，该插件最新的更新时间已经是7个月以前，而且最新的改动只是一个微小改动。所以，在官方网站上并不能找到相关的解决办法及修复，所有的所有只能是靠自己去理解Ambari插件的运行原理并根据思考去尝试修复。

## 问题1：Presto状态检测与Ambari状态检测的冲突问题

在Ambari上进行Restart操作时，Ambari主控台实际上是依次调用了插件的stop/status/start函数，stop调用是为了让组件停止运行，而status调用是检测组件是否正确意义上停止运行，确保后续的start调用不会存在问题。对于Presto插件来说，节点为了coordinator和worker两种类型，但是获取他们的状态都通过Presto `launcher.py`脚本文件的status函数，该函数返回节点的状态：0表示正在运行；3表示已停止运行。而Presto插件脚本文件[presto_coordinator.py](https://github.com/prestodb/ambari-presto-service/blob/master/package/scripts/presto_coordinator.py)在调用status时使用的Execute类，该类为Ambari内部工具类，使用就是执行你传入的脚本内容，并在你的脚本内容执行返回码不为0时抛出错误异常。

presto_coordinator.py脚本函数

```python
def status(self, env):
        from params import daemon_control_script
        Execute('{0} status'.format(daemon_control_script))
```

现在问题就出现在这里，Ambari主控台调用presto_coordinator.py脚本的status函数时，Execute内部实现规定脚本正常执行完成时应该返回0，返回其它值将导致Execute抛出ExecutioinFailed异常。Execute实际是调用了Presto安装目录bin/launcher的status函数（该函数返回了状态码3），导致Execute收到了一个不被期望的返回值3，然后它抛出了一个ExecutionFailed异常，最终导致Ambari重启Presto异常中断。

  `/usr/lib/python2.6/site-packages/resource_management/libraries/script/script.py`文件内容：

```python
def restart(self, env):
    """
    Default implementation of restart command is to call stop and start methods
    Feel free to override restart() method with your implementation.
    For client components we call install
    """
     
    //此处省略N行

    service_name = config['serviceName'] if config is not None and 'serviceName' in config else None
      try:
        #TODO Once the logic for pid is available from Ranger and Ranger KMS code, will remove the below if block.
        services_to_skip = ['RANGER', 'RANGER_KMS']
        if service_name in services_to_skip:
          Logger.info('Temporarily skipping status check for {0} service only.'.format(service_name))
        elif is_stack_upgrade:
          Logger.info('Skipping status check for {0} service during upgrade'.format(service_name))
        else:
          self.status(env)
          raise Fail("Stop command finished but process keep running.")
      except ComponentIsNotRunning as e:
        pass  # expected
      except ClientComponentHasNoStatus as e:
        pass  # expected


   //此处省略N行


```
从上面的代码可以看到调用`self.status(env)`后如果不抛出异常，则后面的`raise Fail("Stop command finished but process keep running.")`就会被执行，导致Ambari流程中断：

{% asset_img BD998F8D-234F-4D82-890B-910A6F38E36D.png %}

## 问题1解决方案

对于这种不兼容情况，需要捕获异常并进行处理，在节点未运行的情况下（返回码3）给予Ambari主控台正确的信息：

修改后的presto_coordinator.py脚本函数
```python
def status(self, env):
        from params import daemon_control_script
        try:
            Execute('{0} status'.format(daemon_control_script))
        except ExecutionFailed as ef:
            if ef.code == 3: #等于3表示Presto节点未运行
                #这里只能抛出这个异常，这个异常在Ambari的框架中会被捕获并被正确理解和处理
                raise ComponentIsNotRunning("ComponentIsNotRunning") 
            else:
                raise ef
```

## 问题2：脚本调用Python Http时传入参数类型不匹配

{% asset_img 7143981D-4C64-41CD-9288-11283C297726.png %}

原因在于presto_coordinator.py脚本中start的时候传入构建PrestoClient对象的port属性并非string类型或者int类型

```python
def start(self, env):
        from params import daemon_control_script, config_properties, \
            host_info
        self.configure(env)
        Execute('{0} start'.format(daemon_control_script))
        if 'presto_worker_hosts' in host_info.keys():
            all_hosts = host_info['presto_worker_hosts'] + \
                host_info['presto_coordinator_hosts']
        else:
            all_hosts = host_info['presto_coordinator_hosts']
        smoketest_presto(PrestoClient('localhost','root',config_properties['http-server.http.port']),all_hosts)

```
## 问题2方案：进行类型转换

修改下该函数： 
```python
def start(self, env):
        from params import daemon_control_script, config_properties, \
            host_info
        self.configure(env)
        Execute('{0} start'.format(daemon_control_script))
        if 'presto_worker_hosts' in host_info.keys():
            all_hosts = host_info['presto_worker_hosts'] + \
                host_info['presto_coordinator_hosts']
        else:
            all_hosts = host_info['presto_coordinator_hosts']
        smoketest_presto(PrestoClient('localhost','root',int(config_properties['http-server.http.port']),all_hosts)
```
解决该问题！

## 问题3：重启冒烟测试时Presto Coodinator拒绝连接

{% asset_img 7D4B838D-E9C6-4264-B335-BB24721ED642.png %}

原因在于：

{% asset_img 136AD971-81E0-41E8-8EBD-9CA9FD244BA5.png %}

在Presto Coordinator节点刚刚开始启动的情况下就进行了冒烟测试，这个启动并未保证Coordinator节点启动完成

## 问题3方案：增加延时时间，等待节点启动完毕

{% asset_img 9E32B640-4897-4BA3-A557-1D4D77F7914D.png %}

该方案只是延时的等待时间，在这个延时时间内一般情况下节点都会启动完成并能成功冒烟测试成功。

# 总结

通过这次的问题排查，对于Ambari的插件体系也有一个大致的认识，也增强了自己问题排查、分析、解决问题的能力。对于Ambari Presto插件的修改我也会积极反馈至开源社区，提交相关ISSUE和PR，希望能得到社区的接纳。^_^

提交的社区ISSUE: [ISSUE-28](https://github.com/prestodb/ambari-presto-service/issues/28)

我的修复PR: [PULL-29](https://github.com/prestodb/ambari-presto-service/pull/29)


