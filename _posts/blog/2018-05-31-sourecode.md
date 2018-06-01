---
layout: post
title: Eureka源码学习
categories: [springcloud]
description: Eureka
keywords: Eureka, Eureka源码学习
---
# Eureka源码学习



为了完成分享的任务，找了些eureka源码学习的资料，整理如下：

## 1. What is Eureka?
其官方文档中对自己的定义是：

> Eureka is a REST (Representational State Transfer) based service that is primarily used in the AWS cloud for locating services for the purpose of load balancing and failover of middle-tier servers. We call this service, the Eureka Server. Eureka also comes with a Java-based client component,the Eureka Client, which makes interactions with the service much easier. The client also has a built-in load balancer that does basic round-robin load balancing.

简单来说Eureka就是Netflix开源的一款提供服务注册和发现的产品，并且提供了相应的Java客户端。

## 2. Why Eureka?
那么为什么我们在项目中使用了Eureka呢？我大致总结了一下，有以下几方面的原因：

* **它提供了完整的Service Registry和Service Discovery实现**
	* 首先是提供了完整的实现，并且也经受住了Netflix自己的生产环境考验，相对使用起来会比较省心。

* **和Spring Cloud无缝集成**
	* 我们的项目本身就使用了Spring Cloud和Spring Boot，同时Spring Cloud还有一套非常完善的开源代码来整合Eureka，所以使用起来非常方便。
	* 另外，Eureka还支持在我们应用自身的容器中启动，也就是说我们的应用启动完之后，既充当了Eureka的角色，同时也是服务的提供者。这样就极大的提高了服务的可用性。

* **Open Source**
	* 最后一点是开源，由于代码是开源的，所以非常便于我们了解它的实现原理和排查问题。

## 3. Dive into Eureka
相信大家看到这里，已经对Eureka有了一个初步的认识，接下来我们就来深入了解下它吧~

### 3.1 Overview

#### 3.1.1 Basic Architecture

![architecture-overview](/images/posts/2018-06-01/architecture-overview.png)

上图简要描述了Eureka的基本架构，由3个角色组成：

1. **Eureka Server**
	* 提供服务注册和发现

2. **Service Provider**
	* 服务提供方
	* 将自身服务注册到Eureka，从而使服务消费方能够找到

3. **Service Consumer**
	* 服务消费方
	* 从Eureka获取注册服务列表，从而能够消费服务

需要注意的是，上图中的3个角色都是逻辑角色。在实际运行中，这几个角色甚至可以是同一个实例，比如在我们项目中，Eureka Server和Service Provider就是同一个JVM进程。

#### 3.1.2 More in depth

![architecture-detail](/images/posts/2018-06-01/architecture-detail.png)

上图更进一步的展示了3个角色之间的交互。

1. Service Provider会向Eureka Server做Register（服务注册）、Renew（服务续约）、Cancel（服务下线）等操作。
2. Eureka Server之间会做注册服务的同步，从而保证状态一致
3. Service Consumer会向Eureka Server获取注册服务列表，并消费服务

### 3.2 Eureka 主要要了解的功能点
#### 3.2.1 服务注册
 在看具体源码前，我们先回顾一下之前我们所实现的内容，从而找一个合适的切入口去分析。首先，服务注册中心、服务提供者、服务消费者这三个主要元素来说，后两者（也就是Eureka客户端）在整个运行机制中是大部分通信行为的主动发起者，而注册中心主要是处理请求的接收者。所以，我们可以从Eureka的客户端作为入口看看它是如何完成这些主动通信行为的。

 我们在将一个普通的Spring Boot应用注册到Eureka Server中，或是从Eureka Server中获取服务列表时，主要就做了两件事：
   1. 在应用主类中配置了@EnableDiscoveryClient注解
   2. 在application.properties中用eureka.client.serviceUrl.defaultZone参数指定了服务注册中心的位置

  顺着上面的线索，我们先查看@EnableDiscoveryClient的源码如下：
  {% highlight java %}
       /**
        * Annotation to enable a DiscoveryClient implementation.
        * @author Spencer Gibb
        */
       @Target(ElementType.TYPE)
       @Retention(RetentionPolicy.RUNTIME)
       @Documented
       @Inherited
       @Import(EnableDiscoveryClientImportSelector.class)
       public @interface EnableDiscoveryClient {

       }
  {% endhighlight %}

   从该注解的注释我们可以知道：该注解用来开启DiscoveryClient的实例。通过搜索DiscoveryClient，我们可以发现有一个类和一个接口。通过梳理可以得到如下图的关系：
   ![eureka-code-1](/images/posts/2018-06-01/eureka-code-1.png)

   其中，左边的org.springframework.cloud.client.discovery.DiscoveryClient是Spring Cloud的接口，它定义了用来发现服务的常用抽象方法，而org.springframework.cloud.netflix.eureka.EurekaDiscoveryClient是对该接口的实现，从命名来就可以判断，它实现的是对Eureka发现服务的封装。所以EurekaDiscoveryClient依赖了Eureka的com.netflix.discovery.EurekaClient接口，EurekaClient继承了LookupService接口，他们都是Netflix开源包中的内容，它主要定义了针对Eureka的发现服务的抽象方法，而真正实现发现服务的则是Netflix包中的com.netflix.discovery.DiscoveryClient类。

   那么，我们就看看来详细看看DiscoveryClient类。先解读一下该类头部的注释有个总体的了解，注释的大致内容如下：

   这个类用于帮助与Eureka Server互相协作。

   Eureka Client负责了下面的任务：
   - 向Eureka Server注册服务实例
   - 向Eureka Server为租约续期
   - 当服务关闭期间，向Eureka Server取消租约
   - 查询Eureka Server中的服务实例列表

   Eureka Client还需要配置一个Eureka Server的URL列表。
   在具体研究Eureka Client具体负责的任务之前，我们先看看对Eureka Server的URL列表配置在哪里。根据我们配置的属性名：eureka.client.serviceUrl.defaultZone，通过serviceUrl我们找到该属性相关的加载属性，但是在SR5版本中它们都被@Deprecated标注了，并在注视中可以看到@link到了替代类com.netflix.discovery.endpoint.EndpointUtils，我们可以在该类中找到下面这个函数：
 {% highlight java %}
   public static Map<String, List<String>> getServiceUrlsMapFromConfig(
   			EurekaClientConfig clientConfig, String instanceZone, boolean preferSameZone) {
       Map<String, List<String>> orderedUrls = new LinkedHashMap<>();
       String region = getRegion(clientConfig);
       String[] availZones = clientConfig.getAvailabilityZones(clientConfig.getRegion());
       if (availZones == null || availZones.length == 0) {
           availZones = new String[1];
           availZones[0] = DEFAULT_ZONE;
       }
   	……
       int myZoneOffset = getZoneOffset(instanceZone, preferSameZone, availZones);

       String zone = availZones[myZoneOffset];
       List<String> serviceUrls = clientConfig.getEurekaServerServiceUrls(zone);
       if (serviceUrls != null) {
           orderedUrls.put(zone, serviceUrls);
       }
   	……
       return orderedUrls;
   }
   {% endhighlight %}
   Region、Zone
   在上面的函数中，我们可以发现客户端依次加载了两个内容，第一个是Region，第二个是Zone，从其加载逻上我们可以判断他们之间的关系：

   通过getRegion函数，我们可以看到它从配置中读取了一个Region返回，所以一个微服务应用只可以属于一个Region，如果不特别配置，就默认为default。若我们要自己设置，可以通过eureka.client.region属性来定义。
   {% highlight java %}
   public static String getRegion(EurekaClientConfig clientConfig) {
       String region = clientConfig.getRegion();
       if (region == null) {
           region = DEFAULT_REGION;
       }
       region = region.trim().toLowerCase();
       return region;
   }
   {% endhighlight %}
   通过getAvailabilityZones函数，我们可以知道当我们没有特别为Region配置Zone的时候，将默认采用defaultZone，这也是我们之前配置参数eureka.client.serviceUrl.defaultZone的由来。若要为应用指定Zone，我们可以通过eureka.client.availability-zones属性来进行设置。从该函数的return内容，我们可以Zone是可以有多个的，并且通过逗号分隔来配置。由此，我们可以判断Region与Zone是一对多的关系。
   {% highlight java %}
   public String[] getAvailabilityZones(String region) {
   	String value = this.availabilityZones.get(region);
   	if (value == null) {
   		value = DEFAULT_ZONE;
   	}
   	return value.split(",");
   }
   {% endhighlight %}
   ServiceUrls
   在获取了Region和Zone信息之后，才开始真正加载Eureka Server的具体地址。它根据传入的参数按一定算法确定加载位于哪一个Zone配置的serviceUrls。
   {% highlight java %}
   int myZoneOffset = getZoneOffset(instanceZone, preferSameZone, availZones);
   String zone = availZones[myZoneOffset];
   List<String> serviceUrls = clientConfig.getEurekaServerServiceUrls(zone);
   {% endhighlight %}
   具体获取serviceUrls的实现，我们可以详细查看getEurekaServerServiceUrls函数的具体实现类EurekaClientConfigBean，该类是EurekaClientConfig和EurekaConstants接口的实现，用来加载配置文件中的内容，这里有非常多有用的信息，这里我们先说一下此处我们关心的，关于defaultZone的信息。通过搜索defaultZone，我们可以很容易的找到下面这个函数，它具体实现了，如何解析该参数的过程，通过此内容，我们就可以知道，eureka.client.serviceUrl.defaultZone属性可以配置多个，并且需要通过逗号分隔。
   {% highlight java %}
   public List<String> getEurekaServerServiceUrls(String myZone) {
   	String serviceUrls = this.serviceUrl.get(myZone);
   	if (serviceUrls == null || serviceUrls.isEmpty()) {
   		serviceUrls = this.serviceUrl.get(DEFAULT_ZONE);
   	}
   	if (!StringUtils.isEmpty(serviceUrls)) {
   		final String[] serviceUrlsSplit = StringUtils.commaDelimitedListToStringArray(serviceUrls);
   		List<String> eurekaServiceUrls = new ArrayList<>(serviceUrlsSplit.length);
   		for (String eurekaServiceUrl : serviceUrlsSplit) {
   			if (!endsWithSlash(eurekaServiceUrl)) {
   				eurekaServiceUrl += "/";
   			}
   			eurekaServiceUrls.add(eurekaServiceUrl);
   		}
   		return eurekaServiceUrls;
   	}
   	return new ArrayList<>();
   }
   {% endhighlight %}
   当客户端在服务列表中选择实例进行访问时，对于Zone和Region遵循这样的规则：优先访问同自己一个Zone中的实例，其次才访问其他Zone中的实例。通过Region和Zone的两层级别定义，配合实际部署的物理结构，我们就可以有效的设计出区域性故障的容错集群。

   服务注册
   在理解了多个服务注册中心信息的加载后，我们再回头看看DiscoveryClient类是如何实现“服务注册”行为的，通过查看它的构造类，可以找到它调用了下面这个函数：
   {% highlight java %}
   private void initScheduledTasks() {
       ...
       if (clientConfig.shouldRegisterWithEureka()) {
           ...
           // InstanceInfo replicator
           instanceInfoReplicator = new InstanceInfoReplicator(
                   this,
                  instanceInfo,
                   clientConfig.getInstanceInfoReplicationIntervalSeconds(),
                   2); // burstSize
           ...
           instanceInfoReplicator.start(clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());
       } else {
           logger.info("Not registering with Eureka server per configuration");
       }
   }
   {% endhighlight %}
   在上面的函数中，我们可以看到关键的判断依据if (clientConfig.shouldRegisterWithEureka())。在该分支内，创建了一个InstanceInfoReplicator类的实例，它会执行一个定时任务，查看该类的run()函数了解该任务做了什么工作：
   {% highlight java %}
   public void run() {
       try {
           discoveryClient.refreshInstanceInfo();
           Long dirtyTimestamp = instanceInfo.isDirtyWithTime();
           if (dirtyTimestamp != null) {
               discoveryClient.register();
               instanceInfo.unsetIsDirty(dirtyTimestamp);
           }
       } catch (Throwable t) {
           logger.warn("There was a problem with the instance info replicator", t);
       } finally {
           Future next = scheduler.schedule(this, replicationIntervalSeconds, TimeUnit.SECONDS);
           scheduledPeriodicRef.set(next);
       }
   }
   {% endhighlight %}
   相信大家都发现了discoveryClient.register();这一行，真正触发调用注册的地方就在这里。继续查看register()的实现内容如下：
   {% highlight java %}
   boolean register() throws Throwable {
       logger.info(PREFIX + appPathIdentifier + ": registering service...");
       EurekaHttpResponse<Void> httpResponse;
       try {
           httpResponse = eurekaTransport.registrationClient.register(instanceInfo);
       } catch (Exception e) {
           logger.warn("{} - registration failed {}", PREFIX + appPathIdentifier, e.getMessage(), e);
           throw e;
       }
       if (logger.isInfoEnabled()) {
           logger.info("{} - registration status: {}", PREFIX + appPathIdentifier, httpResponse.getStatusCode());
       }
       return httpResponse.getStatusCode() == 204;
   }
    {% endhighlight %}
   通过属性命名，大家基本也能猜出来，注册操作也是通过REST请求的方式进行的。同时，这里我们也能看到发起注册请求的时候，传入了一个com.netflix.appinfo.InstanceInfo对象，该对象就是注册时候客户端给服务端的服务的元数据。

   服务获取与服务续约
   顺着上面的思路，我们继续来看DiscoveryClient的initScheduledTasks函数，不难发现在其中还有两个定时任务，分别是“服务获取”和“服务续约”：
    {% highlight java %}
   private void initScheduledTasks() {
       if (clientConfig.shouldFetchRegistry()) {
           // registry cache refresh timer
           int registryFetchIntervalSeconds = clientConfig.getRegistryFetchIntervalSeconds();
           int expBackOffBound = clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
           scheduler.schedule(
                   new TimedSupervisorTask(
                           "cacheRefresh",
                           scheduler,
                           cacheRefreshExecutor,
                           registryFetchIntervalSeconds,
                           TimeUnit.SECONDS,
                           expBackOffBound,
                           new CacheRefreshThread()
                   ),
                   registryFetchIntervalSeconds, TimeUnit.SECONDS);
   	}
   	if (clientConfig.shouldRegisterWithEureka()) {
   		int renewalIntervalInSecs = instanceInfo.getLeaseInfo().getRenewalIntervalInSecs();
           int expBackOffBound = clientConfig.getHeartbeatExecutorExponentialBackOffBound();
           logger.info("Starting heartbeat executor: " + "renew interval is: " + renewalIntervalInSecs);

           // Heartbeat timer
           scheduler.schedule(
                   new TimedSupervisorTask(
                           "heartbeat",
                           scheduler,
                           heartbeatExecutor,
                           renewalIntervalInSecs,
                           TimeUnit.SECONDS,
                           expBackOffBound,
                           new HeartbeatThread()
                   ),
                   renewalIntervalInSecs, TimeUnit.SECONDS);
   		// InstanceInfo replicator
   		……
   	}
   }
   {% endhighlight %}
   从源码中，我们就可以发现，“服务获取”相对于“服务续约”更为独立，“服务续约”与“服务注册”在同一个if逻辑中，这个不难理解，服务注册到Eureka Server后，自然需要一个心跳去续约，防止被剔除，所以他们肯定是成对出现的。从源码中，我们可以清楚看到了，对于服务续约相关的时间控制参数：

   eureka.instance.lease-renewal-interval-in-seconds=30
   eureka.instance.lease-expiration-duration-in-seconds=90
   而“服务获取”的逻辑在独立的一个if判断中，其判断依据就是我们之前所提到的eureka.client.fetch-registry=true参数，它默认是为true的，大部分情况下我们不需要关心。为了定期的更新客户端的服务清单，以保证服务访问的正确性，“服务获取”的请求不会只限于服务启动，而是一个定时执行的任务，从源码中我们可以看到任务运行中的registryFetchIntervalSeconds参数对应eureka.client.registry-fetch-interval-seconds=30配置参数，它默认为30秒。

   继续循序渐进的向下深入，我们就能分别发现实现“服务获取”和“服务续约”的具体方法，其中“服务续约”的实现较为简单，直接以REST请求的方式进行续约：
   {% highlight java %}
   boolean renew() {
       EurekaHttpResponse<InstanceInfo> httpResponse;
       try {
           httpResponse = eurekaTransport.registrationClient.sendHeartBeat(instanceInfo.getAppName(), instanceInfo.getId(), instanceInfo, null);
           logger.debug("{} - Heartbeat status: {}", PREFIX + appPathIdentifier, httpResponse.getStatusCode());
           if (httpResponse.getStatusCode() == 404) {
               REREGISTER_COUNTER.increment();
               logger.info("{} - Re-registering apps/{}", PREFIX + appPathIdentifier, instanceInfo.getAppName());
               return register();
           }
           return httpResponse.getStatusCode() == 200;
       } catch (Throwable e) {
           logger.error("{} - was unable to send heartbeat!", PREFIX + appPathIdentifier, e);
           return false;
       }
   }
   {% endhighlight %}
   而“服务获取”则相对复杂一些，会根据是否第一次获取发起不同的REST请求和相应的处理，具体的实现逻辑还是跟之前类似，有兴趣的读者可以继续查看服务客户端的其他具体内容，了解更多细节。

   服务注册中心处理
   通过上面的源码分析，可以看到所有的交互都是通过REST的请求来发起的。下面我们来看看服务注册中心对这些请求的处理。Eureka Server对于各类REST请求的定义都位于：com.netflix.eureka.resources包下。

   以“服务注册”请求为例：
   {% highlight java %}
   @POST
   @Consumes({"application/json", "application/xml"})
   public Response addInstance(InstanceInfo info,
                     @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication) {
       logger.debug("Registering instance {} (replication={})", info.getId(), isReplication);
       // validate that the instanceinfo contains all the necessary required fields
       ...
       // handle cases where clients may be registering with bad DataCenterInfo with missing data
       DataCenterInfo dataCenterInfo = info.getDataCenterInfo();
       if (dataCenterInfo instanceof UniqueIdentifier) {
           String dataCenterInfoId = ((UniqueIdentifier) dataCenterInfo).getId();
           if (isBlank(dataCenterInfoId)) {
               boolean experimental = "true".equalsIgnoreCase(
   					serverConfig.getExperimental("registration.validation.dataCenterInfoId"));
               if (experimental) {
                   String entity = "DataCenterInfo of type " + dataCenterInfo.getClass()
   										+ " must contain a valid id";
                   return Response.status(400).entity(entity).build();
               } else if (dataCenterInfo instanceof AmazonInfo) {
                   AmazonInfo amazonInfo = (AmazonInfo) dataCenterInfo;
                   String effectiveId = amazonInfo.get(AmazonInfo.MetaDataKey.instanceId);
                   if (effectiveId == null) {
                       amazonInfo.getMetadata().put(
   							AmazonInfo.MetaDataKey.instanceId.getName(), info.getId());
                   }
               } else {
                   logger.warn("Registering DataCenterInfo of type {} without an appropriate id",
   						dataCenterInfo.getClass());
               }
           }
       }

       registry.register(info, "true".equals(isReplication));
       return Response.status(204).build();  // 204 to be backwards compatible
   }
   {% endhighlight %}
   在对注册信息进行了一大堆校验之后，会调用org.springframework.cloud.netflix.eureka.server.InstanceRegistry对象中的register(InstanceInfo info, int leaseDuration, boolean isReplication)函数来进行服务注册：
   {% highlight java %}
   public void register(InstanceInfo info, int leaseDuration, boolean isReplication) {
   	if (log.isDebugEnabled()) {
   		log.debug("register " + info.getAppName() + ", vip " + info.getVIPAddress()
   				+ ", leaseDuration " + leaseDuration + ", isReplication "
   				+ isReplication);
   	}
   	this.ctxt.publishEvent(new EurekaInstanceRegisteredEvent(this, info,
   			leaseDuration, isReplication));

   	super.register(info, leaseDuration, isReplication);
   }
   {% endhighlight %}
   在注册函数中，先调用publishEvent函数，将该新服务注册的事件传播出去，然后调用com.netflix.eureka.registry.AbstractInstanceRegistry父类中的注册实现，将InstanceInfo中的元数据信息存储在一个ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>>对象中，它是一个两层Map结构，第一层的key存储服务名：InstanceInfo中的appName属性，第二层的key存储实例名：InstanceInfo中的instanceId属性



### 3.3 Eureka Server实现细节

看来上面部分代码，再看下具体的时序图：

#### 3.3.1 Register

首先来看Register（服务注册），这个接口会在Service Provider启动时被调用来实现服务注册。同时，当Service Provider的服务状态发生变化时（如自身检测认为Down的时候），也会调用来更新服务状态。

接口实现比较简单，如下图所示。

1. `ApplicationResource`类接收Http服务请求，调用`PeerAwareInstanceRegistryImpl`的`register`方法
2. `PeerAwareInstanceRegistryImpl`完成服务注册后，调用`replicateToPeers`向其它Eureka Server节点（Peer）做状态同步

![eureka-server-register](/images/posts/2018-06-01/eureka-server-register.png)

注册的服务列表保存在一个嵌套的hash map中：

* 第一层hash map的key是app name，也就是应用名字
* 第二层hash map的key是instance name，也就是实例名字

以[3.2.4.2](#service-provider)中的截图为例，`RESERVATION-SERVICE`就是app name，`jason-mbp.lan:reservation-service:8000`就是instance name。

Hash map定义如下：

{% highlight java %}
private final ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>> registry =
new ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>>();
{% endhighlight %}

#### 3.3.2 Renew

Renew（服务续约）操作由Service Provider定期调用，类似于heartbeat。主要是用来告诉Eureka Server Service Provider还活着，避免服务被剔除掉。接口实现如下图所示。

可以看到，接口实现方式和`register`基本一致：首先更新自身状态，再同步到其它Peer。

![eureka-server-renew](/images/posts/2018-06-01/eureka-server-renew.png)

#### 3.3.3 Cancel

Cancel（服务下线）一般在Service Provider shut down的时候调用，用来把自身的服务从Eureka Server中删除，以防客户端调用不存在的服务。接口实现如下图所示。

![eureka-server-cancel](/images/posts/2018-06-01/eureka-server-cancel.png)

#### 3.3.4 Fetch Registries

Fetch Registries由Service Consumer调用，用来获取Eureka Server上注册的服务。

为了提高性能，服务列表在Eureka Server会缓存一份，同时每30秒更新一次。

![eureka-server-fetch](/images/posts/2018-06-01/eureka-server-fetch.png)

#### 3.3.5 Eviction

Eviction（失效服务剔除）用来定期在Eureka Server检测失效的服务，检测标准就是超过一定时间没有Renew的服务。

默认失效时间为90秒，也就是如果有服务超过90秒没有向Eureka Server发起Renew请求的话，就会被当做失效服务剔除掉。

失效时间可以通过`eureka.instance.leaseExpirationDurationInSeconds`进行配置，定期扫描时间可以通过`eureka.server.evictionIntervalTimerInMs`进行配置。

接口实现逻辑见下图：

![eureka-server-evict](/images/posts/2018-06-01/eureka-server-evict.png)

#### 3.3.6 How Peer Replicates

在前面的Register、Renew、Cancel接口实现中，我们看到了都会有replicateToPeers操作，这个就是用来做Peer之间的状态同步。

通过这种方式，Service Provider只需要通知到任意一个Eureka Server后就能保证状态会在所有的Eureka Server中得到更新。

具体实现方式其实很简单，就是接收到Service Provider请求的Eureka Server，把请求再次转发到其它的Eureka Server，调用同样的接口，传入同样的参数，除了会在header中标记isReplication=true，从而避免重复的replicate。

#### 3.3.7 How Peer Nodes are Discovered

那大家可能会有疑问，Eureka Server是怎么知道有多少Peer的呢？

Eureka Server在启动后会调用`EurekaClientConfig.getEurekaServerServiceUrls`来获取所有的Peer节点，并且会定期更新。定期更新频率可以通过`eureka.server.peerEurekaNodesUpdateIntervalMs`配置。

这个方法的默认实现是从配置文件读取，所以如果Eureka Server节点相对固定的话，可以通过在配置文件中配置来实现。

如果希望能更灵活的控制Eureka Server节点，比如动态扩容/缩容，那么可以override `getEurekaServerServiceUrls`方法，提供自己的实现，比如我们的项目中会通过数据库读取Eureka Server列表。

具体实现如下图所示：

![eureka-server-peer-discovery](/images/posts/2018-06-01/eureka-server-peer-discovery.png)

#### 3.3.8 How New Peer Initializes

最后再来看一下一个新的Eureka Server节点加进来，或者Eureka Server重启后，如何来做初始化，从而能够正常提供服务。

具体实现如下图所示，简而言之就是启动时把自己当做是Service Consumer从其它Peer Eureka获取所有服务的注册信息。然后对每个服务，在自己这里执行Register，isReplication=true，从而完成初始化。

![eureka-server-peer-init](/images/posts/2018-06-01/eureka-server-peer-init.png)

### 3.4 Service Provider实现细节

现在来看下Service Provider的实现细节，主要就是Register、Renew、Cancel这3个操作。

#### 3.4.1 Register

Service Provider要对外提供服务，一个很重要的步骤就是把自己注册到Eureka Server上。

这部分的实现比较简单，只需要在启动时和实例状态变化时调用Eureka Server的接口注册即可。需要注意的是，需要确保配置`eureka.client.registerWithEureka`=true。

![service-provider-register](/images/posts/2018-06-01/service-provider-register.png)

#### 3.4.2 Renew

Renew操作会在Service Provider端定期发起，用来通知Eureka Server自己还活着。
这里有两个比较重要的配置需要注意一下：

1. `eureka.instance.leaseRenewalIntervalInSeconds`

	Renew频率。默认是30秒，也就是每30秒会向Eureka Server发起Renew操作。

2. `eureka.instance.leaseExpirationDurationInSeconds`

	服务失效时间。默认是90秒，也就是如果Eureka Server在90秒内没有接收到来自Service Provider的Renew操作，就会把Service Provider剔除。

具体实现如下：

![service-provider-renew](/images/posts/2018-06-01/service-provider-renew.png)

#### 3.4.3 Cancel

在Service Provider服务shut down的时候，需要及时通知Eureka Server把自己剔除，从而避免客户端调用已经下线的服务。

逻辑本身比较简单，通过对方法标记`@PreDestroy`，从而在服务shut down的时候会被触发。

![service-provider-cancel](/images/posts/2018-06-01/service-provider-cancel.png)

#### 3.4.4 How Eureka Servers are Discovered

这里大家疑问又来了，Service Provider是怎么知道Eureka Server的地址呢？

其实这部分的主体逻辑和[3.3.7 How Peer Nodes are Discovered](#how-peer-nodes-are-discovered)几乎是一样的。

也是默认从配置文件读取，如果需要更灵活的控制，可以通过override `getEurekaServerServiceUrls`方法来提供自己的实现。定期更新频率可以通过`eureka.client.eurekaServiceUrlPollIntervalSeconds`配置。

![client-discover-eureka-server](/images/posts/2018-06-01/client-discover-eureka-server.png)

### 3.5 Service Consumer实现细节

Service Consumer这块的实现相对就简单一些，因为它只涉及到从Eureka Server获取服务列表和更新服务列表。

#### 3.5.1 Fetch Service Registries

Service Consumer在启动时会从Eureka Server获取所有服务列表，并在本地缓存。需要注意的是，需要确保配置`eureka.client.shouldFetchRegistry`=true。

![service-consumer-fetch-registries](/images/posts/2018-06-01/service-consumer-fetch-registries.png)

#### 3.5.2 Update Service Registries

由于在本地有一份缓存，所以需要定期更新，定期更新频率可以通过`eureka.client.registryFetchIntervalSeconds`配置。

![service-consumer-update-registries](/images/posts/2018-06-01/service-consumer-update-registries.png)

#### 3.5.3 How Eureka Servers are Discovered

Service Consumer和Service Provider一样，也有一个如何知道Eureka Server地址的问题。

其实由于Service Consumer和Service Provider本质上使用的是同一个Eureka客户端，所以这部分逻辑是一样的，这里就不再赘述了。详细信息见[3.4.4节](#how-eureka-servers-are-discovered)。

## 4. Summary

本文主要介绍了Eureka的实现思路，通过深入了解Eureka Server、Service Provider、Service Consumer的实现，我们清晰地看到了服务注册、发现的一系列过程和实现方式。

相信对正在使用Eureka的同学会有一些帮助，同时希望对暂不使用Eureka的同学也能有一定的启发，毕竟服务注册、发现还是比较基础和通用的，了解了实现方式后，在使用上应该能更得心应手一些吧~