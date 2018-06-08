---
layout: post
title: Eureka源码学习
categories: [springcloud]
description: Eureka
keywords: Eureka, Eureka源码学习
---
# Eureka源码学习
   为了价值观能多拿点分，正好自己也在学习eureka相关的内容，现将自己学习心得，结合网上的一些资料，
   整理成一下文字，与大家分享，如有不足之前欢迎大家拍砖。
## 本次主要分析一下几个点：
   - 服务注册(register)
   - 服务租约续期(renew)
   - 服务取消租约(shutdown)
   - 获取服务实例列表(getApplications)
   - Eureka Server的自我保护模式
### 1.1 如何入手分析
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

### 1.2 服务注册(register)
通过阅读DiscoveryClient的构造方法，我们会发现此类的主要逻辑入口是其构造方法，通过此方法找到注册的入口：
{% highlight java %}
    private void initScheduledTasks() {
         //...省去其他不相关代码
        if (clientConfig.shouldRegisterWithEureka()) {
            ...
            // InstanceInfo replicator
            instanceInfoReplicator = new InstanceInfoReplicator(
                    this,
                   instanceInfo,
                    clientConfig.getInstanceInfoReplicationIntervalSeconds(),
                    2); // burstSize
            //...省去其他不相关代码
            instanceInfoReplicator.start(clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());
        } else {
            logger.info("Not registering with Eureka server per configuration");
        }
    }
 {% endhighlight %}

在上面的函数中，我们可以看到关键的判断依据if (clientConfig.shouldRegisterWithEureka())。
在该分支内，创建了一个InstanceInfoReplicator类的实例，它会执行一个定时任务，
查看该类的run()函数了解该任务做了什么工作：
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
相信大家都发现了discoveryClient.register();这一行，真正触发调用注册的地方就在这里。
继续查看register()的实现内容如下：
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
通过属性命名，大家基本也能猜出来，注册操作也是通过REST请求的方式进行的。同时，这里我们也能看到发起注册请求的时候，
传入了一个com.netflix.appinfo.InstanceInfo对象，该对象就是注册时候客户端给服务端的服务的元数据。

### 1.3 服务租约续期(renew)
顺着上面的思路，我们继续来看DiscoveryClient服务续约，核心代码如下：
{% highlight java %}
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
		//……

{% endhighlight %}

从源码中，我们可以清楚看到了，对于服务续约相关的时间控制参数：
{% highlight java %}
    eureka.instance.lease-renewal-interval-in-seconds=30 //心跳间隔时间
    eureka.instance.lease-expiration-duration-in-seconds=90 //租约过期时间
{% endhighlight %}
上面两个参数默认分别是：30，90s，也就是说eureka server在连续三次没有收到客户端的心跳的话会认为此此客户端异常。
再进一步到com.netflix.discovery.DiscoveryClient.HeartbeatThread#run方法中，发现其调用的还是
com.netflix.discovery.DiscoveryClient#renew方法：
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
不难发现还是跟注册一样的rest 调用，具体eureka server 服务段代码大家有兴趣可以自己去看下。
### 1.4 服务取消租约(shutdown)
应用实例关闭时，Eureka-Client 向 Eureka-Server 发起下线应用实例。需要满足如下条件才可发起：
   - 配置 eureka.registration.enabled = true ，应用实例开启注册开关。默认为 false 。
   - 配置 eureka.shouldUnregisterOnShutdown = true ，应用实例开启关闭时下线开关。默认为 true 。
   继续看com.netflix.discovery.DiscoveryClient#shutdown方法：
   {% highlight java %}
    // DiscoveryClient.java
    public synchronized void shutdown() {

        // ... 省略无关代码

        // If APPINFO was registered
        if (applicationInfoManager != null
             && clientConfig.shouldRegisterWithEureka() // eureka.registration.enabled = true
             && clientConfig.shouldUnregisterOnShutdown()) { // eureka.shouldUnregisterOnShutdown = true
            applicationInfoManager.setInstanceStatus(InstanceStatus.DOWN);
            unregister();
        }
    }
   {% endhighlight %}
    - 调用 ApplicationInfoManager#setInstanceStatus(...) 方法，设置应用实例为关闭( DOWN )。
    - 调用 #unregister() 方法，实现代码如下：
   {% highlight java %}
   // DiscoveryClient.java
   void unregister() {
      // It can be null if shouldRegisterWithEureka == false
      if(eurekaTransport != null && eurekaTransport.registrationClient != null) {
          try {
              logger.info("Unregistering ...");
              EurekaHttpResponse<Void> httpResponse = eurekaTransport.registrationClient.cancel(instanceInfo.getAppName(), instanceInfo.getId());
              logger.info(PREFIX + appPathIdentifier + " - deregister  status: " + httpResponse.getStatusCode());
          } catch (Exception e) {
              logger.error(PREFIX + appPathIdentifier + " - de-registration failed" + e.getMessage(), e);
          }
      }
   }

   // AbstractJerseyEurekaHttpClient.java
   @Override
   public EurekaHttpResponse<Void> cancel(String appName, String id) {
       String urlPath = "apps/" + appName + '/' + id;
       ClientResponse response = null;
       try {
           Builder resourceBuilder = jerseyClient.resource(serviceUrl).path(urlPath).getRequestBuilder();
           addExtraHeaders(resourceBuilder);
           response = resourceBuilder.delete(ClientResponse.class);
           return anEurekaHttpResponse(response.getStatus()).headers(headersOf(response)).build();
       } finally {
           if (logger.isDebugEnabled()) {
               logger.debug("Jersey HTTP DELETE {}/{}; statusCode={}", serviceUrl, urlPath, response == null ? "N/A" : response.getStatus());
           }
           if (response != null) {
               response.close();
           }
       }
   }
   {% endhighlight %}
    调用 AbstractJerseyEurekaHttpClient#cancel(...) 方法，
    DELETE 请求 Eureka-Server 的 apps/${APP_NAME}/${INSTANCE_INFO_ID} 接口，
    实现应用实例信息的下线。

### 1.5 获取服务实例列表(getApplications)
Eureka-Client 启动时，首先执行一次全量获取进行本地缓存注册信息，首先代码如下：
{% highlight java %}
// DiscoveryClient.java

private final AtomicReference<Applications> localRegionApps = new AtomicReference<Applications>();

DiscoveryClient(ApplicationInfoManager applicationInfoManager, EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs args,
                    Provider<BackupRegistry> backupRegistryProvider) {

     // ... 省略无关代码

    // 【3.2.5】初始化应用集合在本地的缓存
    localRegionApps.set(new Applications());

    // ... 省略无关代码

    // 【3.2.12】从 Eureka-Server 拉取注册信息
    if (clientConfig.shouldFetchRegistry() && !fetchRegistry(false)) {
        fetchRegistryFromBackup();
    }

     // ... 省略无关代码
}
{% endhighlight %}

由于具体代码较多，分为全量获取，增量获取具体代码大家可以沿着这条线看下去。

### 1.6 Eureka Server的自我保护模式
#### 1.6.1 定义
当Eureka Server节点在短时间内丢失过多客户端时（可能发生了网络分区故障），那么这个节点就会进入自我保护模式。
一旦进入该模式，Eureka Server就会保护服务注册表中的信息，不再删除服务注册表中的数据（也就是不会注销任何微服务）。
当网络故障恢复后，该Eureka Server节点会自动退出自我保护模式

首先，我们来看下在自动保护机制里扮演重要角色的两个变量：
{% highlight java %}
// AbstractInstanceRegistry.java

protected volatile int numberOfRenewsPerMinThreshold;//期望最小每分钟续租次数

protected volatile int expectedNumberOfRenewsPerMin;//期望最大每分钟续租次数
{% endhighlight %}
#### 1.6.1 触发条件
当每分钟心跳次数( renewsLastMin ) 小于 numberOfRenewsPerMinThreshold 时，
并且开启自动保护模式开关( eureka.enableSelfPreservation = true ) 时，
触发自动保护机制，不再自动过期租约，实现代码如下:
{% highlight java %}
// AbstractInstanceRegistry.java
public void evict(long additionalLeaseMs) {

   if (!isLeaseExpirationEnabled()) {
       logger.debug("DS: lease expiration is currently disabled.");
       return;
   }

   // ... 省略过期租约逻辑
}

// PeerAwareInstanceRegistryImpl.java
@Override
public boolean isLeaseExpirationEnabled() {
   if (!isSelfPreservationModeEnabled()) {
       // The self preservation mode is disabled, hence allowing the instances to expire.
       return true;
   }
   return numberOfRenewsPerMinThreshold > 0 && getNumOfRenewsInLastMin() > numberOfRenewsPerMinThreshold;
}
{% endhighlight %}

#### 1.6.2 计算公式
计算公式如下：

 expectedNumberOfRenewsPerMin = 当前注册的应用实例数 x 2

 numberOfRenewsPerMinThreshold = expectedNumberOfRenewsPerMin * 续租百分比( eureka.renewalPercentThreshold )

为什么乘以 2

默认情况下，注册的应用实例每半分钟续租一次，那么一分钟心跳两次，因此 x 2 。

这块会有一些硬编码的情况，因此不太建议修改应用实例的续租频率。

为什么乘以续租百分比

低于这个百分比，意味着开启自我保护机制。

默认情况下，eureka.renewalPercentThreshold = 0.85 。
如果你真的调整了续租频率，可以等比去续租百分比，以保证合适的触发自我保护机制的阀值。
另外，你需要注意，续租频率是 Client 级别，续租百分比是 Server 级别。
#### 1.6.3 计算时机
目前有四个地方会计算 numberOfRenewsPerMinThreshold 、 expectedNumberOfRenewsPerMin，分别为：
Eureka-Server 初始化定时重置,应用实例注册,应用实例下线,有兴趣大家自己可以具体看下。


## A&Q
