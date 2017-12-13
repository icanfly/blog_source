---
title: 定制springcloud服务注册到consul中的instanceId
author: LP's Notes
tags:
  - SpringCloud
  - 微服务
categories:
  - 原创文章
thumbnail:
blogexcerpt:
---

# 背景

在使用SpringCloud构建微服务过程中，我们使用Consul作为服务的注册中心，中间过程也踩了不少的坑，今天又踩了一个：我们根据官方的建议，在注册springcloud服务的时候，instanceId使用的是以下的配置：
```yaml
spring:
  cloud:
    consul:
      host: 127.0.0.1
      port: 8500
      discovery:
        health-check-path: /management/health
        service-name: mq-gateway
        health-check-interval: 10s
        prefer-ip-address: true
        instance-id: ${spring.cloud.consul.discovery.service-name}:${server.port}:${random.value}
```

重点就在于这个instance-id的配置，它由服务名+服务端口+随机值组成。这种看起来唯一且没有什么问题的配置，却是接下来坑的开始。

<!-- more -->

# 问题

在微服务的开发过程中，不断有开发人员抱怨在开发过程中一些不正常的停止微服务会导致consul上的服务注册实例越来越多，而且IP和端口都一模一样，究其原因是因为不正常的停止导致consul无法正常反注册服务，导致服务注册驻留在consul上，并变为critical状态，而当程序重启时，重新注册的instance-id又会随着${random.value}的配置而与之前的配置不同，这就导致了不断有不同instance-id的实例注册到consul上，而且他们的健康检测url都一样，这个时候当新服务启动后，所有的原有的critical状态的服务的健康检测都能通过，这时候看到的现象就是consul上这个服务挂了很多个实例（其实这些实例都是同一个服务实例）。

而且出于安全的原因，有个非常蛋疼的地方在于consul的服务实例反注册还只能由服务注册所在的机器发起才能反注册。

关于实例重复被注册，在SpringCloud的Github上也有讨论，[链接在这里](https://github.com/spring-cloud/spring-cloud-consul/issues/318)。不过从维护者的回答看出来，好像官方并没有打算做这方面的改进措施。

求人不如求己，自己也试着来看看有没有解决方案吧。

# 解决方案

## 一、通过注册修改微服务健康检测的url来规避

因为多个实例中健康检测的url相同，所以没法区分哪个是正常的实例，所以我们只需要将健康检测的url变成不相同即可，简单的实现如下：

```yaml
spring:
  cloud:
    consul:
      host: 127.0.0.1
      port: 8500
      discovery:
        health-check-path: /${spring.cloud.consul.discovery.instance-id}/management/health
        service-name: mq-gateway
        health-check-interval: 10s
        prefer-ip-address: true
        instance-id: ${spring.cloud.consul.discovery.service-name}:${server.port}:${random.value}
```

但是这种方案有一个很大的弊端在于：健康检测的url对于每个服务来说变得不可得，都是一些随机的url，会导致外部的一些监控程序无法通过某种规则构造服务的健康检测url，从而掌握服务的健康状况，这是一种对于监控系统来说非常不友好的方式。

## 二、通过IP和端口确定instance-id的唯一性

同一个程序，多次启动导致instance-id不相同的原因在于${random.value}，我们尝试去掉它，而${spring.cloud.consul.discovery.service-name}:${server.port}并不能保证唯一性，我们需要加上一个特征使它变得唯一，很好想到的就是用IP来限制：服务名+IP+PORT，这样基本就限制住了唯一性。

但是有一个问题是SpringBoot或者SpringCloud并没有提供一个获取本地IP的配置项。这里我们需要仿造${random.value}的配置原理，构造一个我们自己的IP配置获取方式。


```java
public class RandomValuePropertySource extends PropertySource<Random> {

	/**
	 * Name of the random {@link PropertySource}.
	 */
	public static final String RANDOM_PROPERTY_SOURCE_NAME = "random";

	private static final String PREFIX = "random.";

	private static final Log logger = LogFactory.getLog(RandomValuePropertySource.class);

	public RandomValuePropertySource(String name) {
		super(name, new Random());
	}

	public RandomValuePropertySource() {
		this(RANDOM_PROPERTY_SOURCE_NAME);
	}

	@Override
	public Object getProperty(String name) {
		if (!name.startsWith(PREFIX)) {
			return null;
		}
		if (logger.isTraceEnabled()) {
			logger.trace("Generating random property for '" + name + "'");
		}
		return getRandomValue(name.substring(PREFIX.length()));
	}

	private Object getRandomValue(String type) {
		if (type.equals("int")) {
			return getSource().nextInt();
		}
		if (type.equals("long")) {
			return getSource().nextLong();
		}
		String range = getRange(type, "int");
		if (range != null) {
			return getNextIntInRange(range);
		}
		range = getRange(type, "long");
		if (range != null) {
			return getNextLongInRange(range);
		}
		if (type.equals("uuid")) {
			return UUID.randomUUID().toString();
		}
		return getRandomBytes();
	}

	private String getRange(String type, String prefix) {
		if (type.startsWith(prefix)) {
			int startIndex = prefix.length() + 1;
			if (type.length() > startIndex) {
				return type.substring(startIndex, type.length() - 1);
			}
		}
		return null;
	}

	private int getNextIntInRange(String range) {
		String[] tokens = StringUtils.commaDelimitedListToStringArray(range);
		int start = Integer.parseInt(tokens[0]);
		if (tokens.length == 1) {
			return getSource().nextInt(start);
		}
		return start + getSource().nextInt(Integer.parseInt(tokens[1]) - start);
	}

	private long getNextLongInRange(String range) {
		String[] tokens = StringUtils.commaDelimitedListToStringArray(range);
		if (tokens.length == 1) {
			return Math.abs(getSource().nextLong() % Long.parseLong(tokens[0]));
		}
		long lowerBound = Long.parseLong(tokens[0]);
		long upperBound = Long.parseLong(tokens[1]) - lowerBound;
		return lowerBound + Math.abs(getSource().nextLong() % upperBound);
	}

	private Object getRandomBytes() {
		byte[] bytes = new byte[32];
		getSource().nextBytes(bytes);
		return DigestUtils.md5DigestAsHex(bytes);
	}

	public static void addToEnvironment(ConfigurableEnvironment environment) {
		environment.getPropertySources().addAfter(
				StandardEnvironment.SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME,
				new RandomValuePropertySource(RANDOM_PROPERTY_SOURCE_NAME));
		logger.trace("RandomValuePropertySource add to Environment");
	}
```
这个给了我们很大的提示，我们自己也可以在SpringBoot程序启动的时候注入一个我们自己的ProperySource将机器的IP作为配置项作为其它其它配置项的引用。并结合我们自己的配置中心客户端，可以在开发人员不感知的情况下就把这个事情给做掉。

```
public class CustomizeApplication extends SpringApplication {


    public static ConfigurableApplicationContext run(Object source, String... args) {
        return run(new Object[] { source }, args);
    }

    public static ConfigurableApplicationContext run(Object[] sources, String[] args) {

        advanceFetchApolloConfig(sources);

        return new CustomizeApplication(sources).run(args);
    }

    ...//此处省略

    @Override
    protected void configurePropertySources(ConfigurableEnvironment environment, String[] args) {
        super.configurePropertySources(environment, args);

        //add local overwrite config file
        if(localOverwriteConfig != null){
            environment.getPropertySources().addFirst(localOverwriteConfig);
        }

        environment.getPropertySources().addAfter(ConfigConsts.PREDECESSOR_OF_APOLLO, bootstrapConfig);

        //注入Server相关属性及配置
        environment.getPropertySources().addAfter(bootstrapConfig.getName(), new ServerPropertiesSource(environment));
    }

    ...//此处省略
}
```

自定义的PropertySource:
```java
public class ServerPropertiesSource extends PropertySource<Object> {

    private Logger logger = LoggerFactory.getLogger(ServerPropertiesSource.class);

    public static final String SERVER_PROPERTIES_NAME = "xxx.server";

    public static final String SERVER_ADDR_PATTERN = SERVER_PROPERTIES_NAME + ".addr.pattern";

    private LoadingCache<String, Object> loadingCache = CacheBuilder.newBuilder()
            .expireAfterAccess(60, TimeUnit.SECONDS)
            .maximumSize(1000).build(new CacheLoader<String, Object>() {
                @Override
                public Object load(String key) throws Exception {
                    return _getProperty(key);
                }
            });
    private Environment environment;

    public ServerPropertiesSource(Environment environment) {
        this(SERVER_PROPERTIES_NAME, environment);
    }

    public ServerPropertiesSource(String name, Environment environment) {
        super(name, new Object());
        this.environment = environment;
    }

    @Override
    public Object getProperty(String name) {
        try {
            return loadingCache.get(name);
        } catch (Exception e) {
            return null;
        }
    }

    public Object _getProperty(String name) {
        if (!StringUtils.startsWithIgnoreCase(name, SERVER_PROPERTIES_NAME)) {
            return null;
        }

        if (logger.isTraceEnabled()) {
            logger.trace("get server property for '" + name + "'");
        }

        return getServerProperty(name.substring(SERVER_ADDR_PATTERN.length()));
    }

    private Object getServerProperty(String subName) {
        if (StringUtils.startsWithIgnoreCase(subName, ".addr")) {
            return getServerIp();
        }
        return null;
    }

    private Object getServerIp() {
        try {
            String serverAddrPattern = this.environment.getProperty(SERVER_ADDR_PATTERN);
            if (serverAddrPattern != null) {
                Pattern pattern = Pattern.compile(serverAddrPattern);
                return InetAddressUtils.getLocalAddress(pattern);
            }
            return InetAddressUtils.getLocalAddress();
        } catch (Exception e) {
            logger.error(e.getMessage(),e);
            throw e;
        }
    }

应用配置：
```yaml
spring:
  cloud:
    consul:
      host: 127.0.0.1
      port: 8500
      discovery:
        health-check-path: /management/${instance-id}/health
        service-name: mq-gateway
        health-check-interval: 10s
        prefer-ip-address: true
        instance-id: ${spring.cloud.consul.discovery.service-name}:${xxx.server.addr}:${server.port}
```

```
SpringBoot启动：

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableApolloConfig({ "application", "common.consul"})
public class DemoApplication {

	public static void main(String[] args) {
		CustomizeApplication.run(DemoApplication.class, args);
	}
}
```
注册到Consul中的服务：
{% asset_img consul.png %}

至此，大功告成！
