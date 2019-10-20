---
layout: post
title: SpringCloud zuul 中的超时时间
categories: [Springcloud]
description: zuul 超时时间
keywords: zuul timeout, SpringCloud
---
### SpringCloud zuul中的几个超时时间,优先级顺序如下：

1.hystrixTimeout

2.ribbonTimeout

3.clientTimeout


#### 源代码解析
 关于springCloud zuul 网关的超时时间配置网上的说法很多，但是都不如源码来的清晰
 SpringCloud zuul 网关中集成了hystrix，ribbon组件，对于转发http的请求的客户端
 自己可以选择Okhhttp,apache http-client,或者Spring 的Rest templet,
 但是不管怎么样最终都换转换成对应的hystrixCommond,比如OkHttpRibbonCommand，HttpClientRibbonCommand，HttpClientRibbonCommand
 而超时时间获取规则都在其父类AbstractRibbonCommand实现，具体源码如下：
 ````
 protected static int getHystrixTimeout(IClientConfig config, String commandKey) {
 		int ribbonTimeout = getRibbonTimeout(config, commandKey);
 		DynamicPropertyFactory dynamicPropertyFactory = DynamicPropertyFactory.getInstance();
 		int defaultHystrixTimeout = dynamicPropertyFactory.getIntProperty("hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds",
 			0).get();
 		int commandHystrixTimeout = dynamicPropertyFactory.getIntProperty("hystrix.command." + commandKey + ".execution.isolation.thread.timeoutInMilliseconds",
 			0).get();
 		int hystrixTimeout;
 		if(commandHystrixTimeout > 0) {
 			hystrixTimeout = commandHystrixTimeout;
 		}
 		else if(defaultHystrixTimeout > 0) {
 			hystrixTimeout = defaultHystrixTimeout;
 		} else {
 			hystrixTimeout = ribbonTimeout;
 		}
 		if(hystrixTimeout < ribbonTimeout) {
 			LOGGER.warn("The Hystrix timeout of " + hystrixTimeout + "ms for the command " + commandKey +
 				" is set lower than the combination of the Ribbon read and connect timeout, " + ribbonTimeout + "ms.");
 		}
 		return hystrixTimeout;
 	}
 ````
 通过上面源码可以看出如果，配置Hystrix超时时间就以Hystrix超时时间为准，如果没配置则以ribbon
 配置的超时时间为准，记得这个就一切ok,不需要去配啥ribbonReadTimeout，ribbonConnectTimeout。
