﻿# Spring源码阅读

标签（空格分隔）： spring 源码

---

本文主要针对springBoot、springCloud的相关组件的源码阅读及个人理解。

参考链接
----

 1. https://blog.csdn.net/woshilijiuyi/article/details/82219585
 2. https://blog.csdn.net/qq_35119422/article/details/81559410

目录
--

 1. springboot启动过程
 2. springboot的starter
 3. 自定义starter
 4. eureka源码
 5. feign源码
 6. hystrix源码

eureka源码分析
----------

 - 各个服务组件是如何将自己注册到eureka上的呢？

简单来说就是想eureka发送REST请求将自己注册到Eureka Server上的。

 - 那么，这些注册信息又是如何在Eureka上保存的呢？

Eureka内部通过，使用双层的Map结构来保存服务的注册信息。第一层key是服务名，通过spring.application.name配置，第二层key是具体的实例名称（比如，一个服务可以启动多个实例），通过spring.instance.hostname设置。

 - 服务注册到eureka上以后，如何告诉eureka自己还”活着“呢？避免eureka将自己剔除呢？

服务于eureka会通过心跳机制，来告诉eureka，自己还”活着“。这也叫”服务续约“，关于续约，有下面两个重要的参数：
```java
spring.instance.lease-renewal-interval-in-seconds=30
spring.instance.lease-expiration-duration-in-seconds=90
```
前者定义了服务多久eureka通信一次，后者告诉eureka多久将无心跳的服务踢下线。上面的30和90秒都是默认时间，也就是说，不手动配置，这两个参数也是生效的，只不过取默认值。

 - 服务消费者如何获取eureka上注册的服务提供者呢？

 消费者会发送REST请求给eureka，以获取eureka上注册的服务清单（可能不仅仅只包含服务提供者）。处于性能考虑，eureka会维护（缓存）一份只读的服务清单来返回给客户端。该缓存清单没隔30s会更新一次。如果希望修改这个时间，可以使用如下的配置：
 ```java
 eureka.client.registry-fetch-interval-seconds=15
 ```
 消费者获取到服务清单以后（起止包含服务提供者的元数据信息），可以根据自己的实际需要进行调用，Ribbon默认采用轮训的方式进行调用，从而实现负载均衡。
 
 

 - 服务下线

正常的服务下线时，会给eureka发送下线的Rest请求，告诉eureka将自己从服务列表剔除。

但是，如果服务异常关闭了，是不会往eureka发送下线请求的，eureka会每60秒检测一次是否有超时时间超过90秒的无心跳服务，如果有就将其从服务列表清除。

 - 源码分析

一般配置一个eureka服务，我们会做两件事情：

 1. 在Application主类中配置@EnableDiscoveryClient/@EnableEurekaClient注解
 2. 在application.properties中配置defaultZone

注意：@EnableDiscoveryClient/@EnableEurekaClient功能类似，区别在于@EnableDiscoveryClient适用于包含eureka在内的多个注册中心，但是@EnableEurekaClient只适用于eureka。（@EnableEurekaServer注解也可以达到上面两个注解的效果，但是区别嘛。。。嗯  暂时不清楚，官方也没说。）

@EnableDiscoveryClient主要用来开启DiscoveryClient实例，通过梳理，可以得到如下所示的依赖关系图：

![此处输入图片的描述][1]

真正实现发现服务的是com.netflix.discovery.DiscoveryClient类，该类声明及注释如下：
```java
/**
 * The class that is instrumental for interactions with <tt>Eureka Server</tt>.
 *
 * <p>
 * <tt>Eureka Client</tt> is responsible for a) <em>Registering</em> the
 * instance with <tt>Eureka Server</tt> b) <em>Renewal</em>of the lease with
 * <tt>Eureka Server</tt> c) <em>Cancellation</em> of the lease from
 * <tt>Eureka Server</tt> during shutdown
 * <p>
 * d) <em>Querying</em> the list of services/instances registered with
 * <tt>Eureka Server</tt>
 * <p>
 *
 * <p>
 * <tt>Eureka Client</tt> needs a configured list of <tt>Eureka Server</tt>
 * {@link java.net.URL}s to talk to.These {@link java.net.URL}s are typically amazon elastic eips
 * which do not change. All of the functions defined above fail-over to other
 * {@link java.net.URL}s specified in the list in the case of failure.
 * </p>
 *
 * @author Karthik Ranganathan, Greg Kim
 * @author Spencer Gibb
 *
 */
@Singleton
public class DiscoveryClient implements EurekaClient {
```
哈哈哈哈

**服务注册源码分析**

在com.netflix.discovery.DiscoveryClient类中可以找到一个方法initScheduledTasks()，这个方法在DiscoveryClient的都早函数中，会被调用，他主要是负责启动服务列表获取、服务注册的定时任务，源码如下：
```java
/**
     * Initializes all scheduled tasks.
     */
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
            logger.info("Starting heartbeat executor: " + "renew interval is: {}", renewalIntervalInSecs);

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
            instanceInfoReplicator = new InstanceInfoReplicator(
                    this,
                    instanceInfo,
                    clientConfig.getInstanceInfoReplicationIntervalSeconds(),
                    2); // burstSize

            statusChangeListener = new ApplicationInfoManager.StatusChangeListener() {
                @Override
                public String getId() {
                    return "statusChangeListener";
                }

                @Override
                public void notify(StatusChangeEvent statusChangeEvent) {
                    if (InstanceStatus.DOWN == statusChangeEvent.getStatus() ||
                            InstanceStatus.DOWN == statusChangeEvent.getPreviousStatus()) {
                        // log at warn level if DOWN was involved
                        logger.warn("Saw local status change event {}", statusChangeEvent);
                    } else {
                        logger.info("Saw local status change event {}", statusChangeEvent);
                    }
                    instanceInfoReplicator.onDemandUpdate();
                }
            };

            if (clientConfig.shouldOnDemandUpdateStatusChange()) {
                applicationInfoManager.registerStatusChangeListener(statusChangeListener);
            }

            instanceInfoReplicator.start(clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());
        } else {
            logger.info("Not registering with Eureka server per configuration");
        }
    }
```
上面的if (clientConfig.shouldFetchRegistry())就是判断是否需要启动服务列表获取的定时任务，if (clientConfig.shouldRegisterWithEureka())就是判断是否需要将服务注册到eureka上去。定时任务的执行周期都可以配置文件进行配置，否则取默认值。

 
 在if (clientConfig.shouldRegisterWithEureka())分支内创建了一个InstanceInfoReplicator对象，这个对象实现了Runnable接口，通过查看其run方法：
 ```java
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
 ```
 它内部的register方法就是做注册使用的，同时com.netflix.appinfo.InstanceInfo保存了注册服务的元信息。
 ```java
 /**
     * Register with the eureka service by making the appropriate REST call.
     */
    boolean register() throws Throwable {
        logger.info(PREFIX + "{}: registering service...", appPathIdentifier);
        EurekaHttpResponse<Void> httpResponse;
        try {
            httpResponse = eurekaTransport.registrationClient.register(instanceInfo);
        } catch (Exception e) {
            logger.warn(PREFIX + "{} - registration failed {}", appPathIdentifier, e.getMessage(), e);
            throw e;
        }
        if (logger.isInfoEnabled()) {
            logger.info(PREFIX + "{} - registration status: {}", appPathIdentifier, httpResponse.getStatusCode());
        }
        return httpResponse.getStatusCode() == Status.NO_CONTENT.getStatusCode();
    }
 ```
 
 **服务获取源码分析**
 
 在com.netflix.discovery.DiscoveryClient类内部的initScheduledTasks方法内部还有一个if判断主要是启动服务获取的定时任务的，源码如下：
 ```java
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
 ```


  [1]: https://github.com/Audi-A7/learn/blob/master/image/spring/eurekaClient.png?raw=true