---
title: Ranger自定义插件开发
date: 2017-01-23 14:58:20
tags:
- 大数据
- ranger
---

>英文链接：https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=53741207

# Ranger简介

Apache Ranger为Hadoop体系提供了统一的安全体系，包括细致的访问控制和统一的审计。 Apache Ranger 0.4版本为很多服务提供了授权和审核，这些服务包括： HDFS, HBase, Hive, Knox和Storm。如果要添加更多的服务支持就需要多个模块的修改，包括UI，API，数据库schema等等。

Apache Ranger 0.5版本支持了一个统一的模型来更好地支持新组件的接入，而这些接入并不需要改变Ranger的代码。这篇文章主要阐述了自定义Ranger组件的编程模型以及步骤。

# 创建自定义授权插件

这节将提供的是一个创建授权插件的高阶视图。 更多的每一步细节都会在后面的步骤中说明。

## 定义服务类型(Service-type)

1. 创建一个JSON格式的文件，包含以下内容：

  - 资源,比如：database,table,column等
  - 访问类型,比如：select, update, create, drop等
  - 连接服务的配置,比如：JDBC URL,JDBC driver,credentials等

2. 加载JSON文件到Ranger中

<!--more-->

## Ranger授权插件开发

### 服务初始化

1. 创建一个静态的（或者全局的）`RangerBasePlugin`实例（或者继承至它的一个实例），并提供一个方便后面授权使用的引用
2. 调用该实例的`init()`方法。这个步骤将会初始化policy-engine，它会从Ranger Admin拉取安全策略，并启动一个后台线程用于周期性的从Ranger Admin更新安全策略。
3. 在插件实例中注册一个审计处理器，比如：`RangerDefaultAuditHandler`。 插件将会使用这个审计处理器生成资源访问的审计日志。

### 资源授权访问

1. 创建一个`RangerAccessRequest`实现的实例，该实例拥有资源、访问类型、用户等需要被授权的细节逻辑，这个实现类可以参考`RangerAccessRequestImpl`。
2. 调用前面创建的插件实例方法`isAccessAllowed()`。
3. 根据返回的结果决定允许还是拒绝操作。

### 资源查找

1. 继承`RangerBaseService`类，实现`lookupResource()`和`validateConfig()`方法。
2. 为这个类提供一个在服务定义中的名字。
3. 将包含该实现的类的类库（jar包文件）以及该类库依赖的其它类库拷贝到Ranger Admin的`ranger-plugins/<service-type>`目录

## 安装插件到服务中

要想访问授权生效，Ranger插件必须先进行安装和配置。 请参看查看相关文档，如何注册一个授权（authorizer）。

# 服务类型

## 服务类型定义

一个服务的资源，如：资源访问类型（read/write/create/delete/submit/...）, 连接服务的配置信息（url/username/password/...）,在策略计算时的自定义的一些条件（IP range等）。这些都需要定义在JSON文件中，最新的服务类型定义格式请查看最新的文档[服务定义格式](https://github.com/apache/incubator-ranger/blob/master/agents-common/src/main/resources/service-defs/)

## 例子-YARN服务类型定义

```json
{
 "name": "yarn",
 "implClass": "org.apache.ranger.services.yarn.RangerServiceYarn",
 "label": "YARN",
 "description": "YARN",
 "guid": "5b710438-edcf-4e20-834c-a9a267b5b963",
  "resources":
 [
  {
   "name": "queue",
   "type": "string",
   "level": 10,
   "mandatory": true,
   "lookupSupported": true,
   "recursiveSupported": true,
   "matcher": "org.apache.ranger.plugin.resourcematcher.RangerPathResourceMatcher",
   "matcherOptions": {"wildCard":true, "ignoreCase":true, "pathSeparatorChar":"."},
   "label": "Queue",
   "description": "Queue"
  }
 ],
 "accessTypes":
 [
  {
   "name": "submit-app",
   "label": "submit-app"
  },
  {
   "name": "admin-queue",
   "label": "admin-queue"
  }
 ],

 "configs":
 [
  {
   "name": "username",
   "type": "string",
   "mandatory": true,
   "label": "Username"
  },
  {
   "name": "password",
   "type": "password",
   "mandatory": true,
   "label": "Password"
  },
  {
   "name": "yarn.url",
   "type": "string",
   "mandatory": true,
   "label": "YARN REST URL"
  },

  {
   "name": "commonNameForCertificate",
   "type": "string",
   "mandatory": false,
   "label": "Common Name for Certificate"
  }
 ],

 "policyConditions":
 [
  {
   "name": "ip-range",
   "evaluator": "org.apache.ranger.plugin.conditionevaluator.RangerIpMatcher",
   "label": "IP Address Range",
   "description": "IP Address Range"
  }
 ]
}
```

## 注册服务类型

服务类型注册必须使用Ranger Admin提供的RESTFUL API来进行。 服务类型一旦注册成功，Ranger Admin就会提供一个创建服务的UI页面(在以前的发行版中叫做repositories)以及该服务的策略。 Ranger插件使用服务定义和策略来确定一个访问请求应该被允许还是被拒绝。 Ranger Admin提供的REST API可以通过`curl`这个小命令行工具调用:
```
curl -u admin:admin -X POST -H "Accept: application/json" -H "Content-Type: application/json" –d @ranger-servicedef-yarn.json http://ranger-admin-host:port/service/plugins/definitions
```

# Ranger插件开发

## Ranger Authorizer

Ranger服务授权体系主要是通过以下方式实现的：提供一个lib，该lib实现了服务钩子(hook)拦截资源访问，并调用Ranger API获得授权以及记录审计日志。 当在某个服务里面安装ranger插件时，这些钩子会被自动注册到服务中。在这节中，我们会深入YARN ranger插件的实现细节（YARN服务类型定义已经在前面一节中做过了）。

在服务的初始化过程中，一个静态（或者全局的）`RangerBasePlugin`实例应该被创建出来， 该实例的引用也应该被提供出来，以方便后续访问授权请求的使用。

在初始化过程中，插件将会从本地缓存中加载策略（如果本地存在的话），并启动一个策略更新器用于从Ranger Admin中拉取更新后的策略。

YARN服务需要授权服务实现`YarnAuthorizationProvider`接口。 Ranger YARN插件实现了`init()`和`checkPermission()`方法用于提供授权和YARN队列的访问审计。
```java
public class RangerYarnAuthorizer extends YarnAuthorizationProvider {
   	private static RangerBasePlugin plugin = null;
   
    @Override
	public void init(Configuration conf) {
		  plugin = new RangerBasePlugin("yarn", "yarn");
		  plugin.init(); // this will initialize policy engine and policy refresher
		  plugin.setDefaultAuditHandler(new RangerDefaultAuditHandler());
	}

	@Override
	public boolean checkPermission(AccessType accessType, PrivilegedEntity entity, UserGroupInformation ugi) {
		  RangerAccessRequestImpl request  = new RangerAccessRequestImpl();
		  RangerResourceImpl      resource = new RangerResourceImpl();
		 
		  resource.setValue("queue", entity.getName());
		   request.setResource(resource);
		   request.setAccessType(getRangerAccessType(accessType));
		   request.setUser(ugi.getShortUserName());
		   request.setUserGroups(Sets.newHashSet(ugi.getGroupNames()));
		   request.setAccessTime(new Date());
		   request.setClientIPAddress(getRemoteIp());
		  RangerAccessResult result = plugin.isAccessAllowed(request);
		  return result == null ? false : result.getIsAllowed();
 	}
}

```

## 资源查找

在Ranger Admin中构建策略的时候，用户会输入需要保护的资源的名字。为了让用户使用更方便，Ranager Admin提供了一自动完成特性，该特性会根据输入的内容查询服务中可用的匹配资源。

lookup的实现是针对被访问的资源中的服务。它涉及服务提供的API以及检索可用的资源。为了完成`autocomplete`特性，Ranger Admin要求插件提供`RangerBaseService`的一个具体实现。这个实现类必须在Ranger的服务类型中注册，并且要保证该类库已经放置到了Ranger Admin的类路径下。

```java
public class RangerServiceYarn extends RangerBaseService {
	 public HashMap<String, Object> validateConfig() throws Exception {
	  	// TODO: connect to YARN resource manager; throw Exception on failure
	 }
	 
	 public List<String> lookupResource(ResourceLookupContext context) throws Exception {
	  	// TODO: retrieve the resource list from YARN resource manager using REST API
	 }
}
```

# 安装和配置插件

以下的实现了Ranger插件的jar文件必须保证已经放置在了服务的类路径下（如YARN）：

- ranger-plugins-audit-&lt;version&gt;.jar
- ranger-plugins-common-&lt;version&gt;.jar
- ranger-plugins-cred-&lt;version&gt;.jar

Ranger插件在初始化的时候会读取以下文件，也请确保以下文件已经放置在了服务的类路径下：

- ranger-&lt;serviceType&gt;-audit.xml
- ranger-&lt;serviceType&gt;-security.xml
- ranger-policymgr-ssl.xml

Ranger插件需要以下配置才能正常运行，这些配置属性通常在ranger-&lt;serviceType&gt;-security.xml中。

| 配置 | 默认值  |  备注  |
| :------: |:----:| :--------:| 
| ranger.plugin.&lt;serviceType&gt;.service.name | No default value. This configuration must be provided. | Name of the service containing policies for the plugin |
| ranger.plugin.&lt;serviceType&gt;.policy.source.impl | org.apache.ranger.admin.client.RangerAdminRESTClient | Name of the class used to retrieve policies. |
| ranger.plugin.&lt;serviceType&gt;.policy.rest.url | No default value. | URL to Ranger Admin |
| ranger.plugin.&lt;serviceType&gt;.policy.rest.ssl.config.file | No default value. This configuration must be provided if SSL is enabled between plugin and Ranger Admin. | Path to the file containing SSL details to contact Ranger Admin |
| ranger.plugin.&lt;serviceType&gt;.policy.cache.dir | No default value. If no valid value is specified, local caching of policies will not be done. |Directory where Ranger policies are cached after successful retrieval from the source |
| ranger.plugin.&lt;serviceType&gt;.policy.pollIntervalMs | 30000 | How often to poll for changes in policies? |

