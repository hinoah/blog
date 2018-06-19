---
title: Spring Cloud Zuul 实现服务提供者自定义状态检测
date: 2018-06-19 23:22:47
tags: [Spring Cloud, Zuul]
---

Zuul 用作于 Spring Cloud 的微服务网关，是微服务架构中所有流量的入口。可以配合服务注册中心 Eureka，Zookeeper 用于服务发现。
有时候，我们由于某个服务提供者的问题需要手动下线掉该节点，但又不想停用该节点上的进程，有没有什么比较好的办法呢？

### zuul 如何判断服务的状态
注册于注册中心的服务，zuul 如何判断其当前的健康状态呢。
翻看源码，发现 zuul 为我们提供了这样一个接口(本质上是 netflix.loadbalancer 提供的)
```java
package com.netflix.loadbalancer;

/**
 * Interface that defines how we "ping" a server to check if its alive
 * @author stonse
 *
 */
public interface IPing {
    
    /**
     * Checks whether the given <code>Server</code> is "alive" i.e. should be
     * considered a candidate while loadbalancing
     * 
     */
    public boolean isAlive(Server server);
}
```
根据接口定义，isAlive 方法接收参数 Server，Server 类中主要定义了服务提供者的 IP、端口等一些元数据，根据这些数据可以发挥当前服务实例是否可用。Zuul 会周期性调用这个接口，来判断服务提供者是否“活着”，如果发现某个服务提供者不再存活，则将其剔除出服务提供列表。

下面来看一下 zuul 的默认实现:
```java
package com.netflix.loadbalancer;

/**
 * No Op Ping
 * @author stonse
 *
 */
public class NoOpPing implements IPing {

    @Override
    public boolean isAlive(Server server) {
        return true;
    }

}
```
这里实际上就表示只要是从注册中心获得到的服务实例，默认都认为是可用的。

### 自定义检测逻辑
那假设这样一种场景，某个服务实例本身没有挂掉，但因为其环境或网络的原因导致其接口实际上不可用，我们如何优雅得暂时让其“隐身”呢？
我们可不可以自己定义一种规则，实现 zuul 对其当前服务提供者状态的监测？答案是肯定的。
首先我们实现这个 IPing 接口，然后编写自己的监测逻辑，我这边以 Zookeeper 作为服务发现的情况贴一下代码实现。
```java
import com.netflix.loadbalancer.IPing;
import com.netflix.loadbalancer.Server;
import org.springframework.cloud.zookeeper.discovery.ZookeeperServer;
import org.springframework.cloud.zookeeper.support.StatusConstants;
import org.springframework.util.CollectionUtils;

import java.util.Map;

/**
 * 根据实例 metadata 检测
 *
 * @author hinoah
 */
public class InstanceStatusPing implements IPing {

    private static final String INSTANCE_STATUS_KEY = StatusConstants.INSTANCE_STATUS_KEY;

    @Override
    public boolean isAlive(Server server) {
        if (server instanceof ZookeeperServer) {
            ZookeeperServer instance = (ZookeeperServer) server;
            Map<String, String> metadata = instance.getInstance().getPayload().getMetadata();
            if (!CollectionUtils.isEmpty(metadata)) {
                if (metadata.containsKey(INSTANCE_STATUS_KEY)) {
                    return metadata.get(INSTANCE_STATUS_KEY).equalsIgnoreCase(StatusConstants.STATUS_UP);
                }
            }
        }
        return true;
    }
}
```
这里根据服务实例上的元信息的 INSTANCE_STATUS_KEY 的值来判断当前服务实例是否可用，当我们需要下线某个服务又不希望杀死进程时，只需要在 zookeeper 上将这个值改为 STATUS_OUT_OF_SERVICE 就可以了。
最后，还需要将其注册到 Ribbon 的配置中去。
```java
import com.netflix.loadbalancer.IPing;
import com.vpgame.microservice.gateway.loadbalancer.ping.InstanceStatusPing;
import org.springframework.cloud.netflix.ribbon.RibbonClients;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
@RibbonClients(defaultConfiguration = RibbonConfig.class)
public class RibbonConfig {

    @Bean
    public IPing instanceStatusPing() {
        return new InstanceStatusPing();
    }

}
```

下一篇将讲解下如何实现服务实例分配不同权重的功能。
