---
layout:     post
title:      Spring Cloud Ribbon 负载均衡源码分析
subtitle:   Spring Cloud Ribbon 负载均衡源码分析
date:       2019-08-08
author:     金志宏
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - Java
    - Netflix
    - Spring Cloud
    - 微服务
    - Ribbon
---

# Spring Cloud Ribbon 负载均衡源码分析

> Ribbon 可以实现微服务客户端的软负载均衡，说白了就是通过一系列算法，在服务调用时动态获取服务列表并且选择一个节点调用。Ribbon 有多种负载均衡策略，程序员可以自己进行配置。



- 以下列出几种常用的负载平衡策略，后面源码主要介绍前4种负载均衡策略。

| 实现类                    | 默认                                                  | 策略                                                         |
| ------------------------- | ----------------------------------------------------- | ------------------------------------------------------------ |
| RoundRobinRule            | DEFAULT                                               | 最基本的负载平衡策略，即循环规则                             |
| RandomRule                |                                                       | 一种在现有网络中随机分配流量的负载均衡策略                   |
| BestAvailableRule         |                                                       | 跳过带有“跳闸”断路器的服务器并选择具有最低并发请求的服务器的规则。 |
| WeightedResponseTimeRule  |                                                       | 使用平均/百分位响应时间为每个服务器分配动态“权重”的规则，然后以“加权循环”方式使用该规则。刚启动时如果统计信息不足，则上有RoundRobinRule策略，等统计信息足够，会切换到WeightedResponseTimeRule |
| RetryRule                 |                                                       | 重试 先按照轮询策略获取服务，如果获取失败则在指定时间内重试，获取可用服务 |
| AvailabilityFilteringRule |                                                       | 会先过滤掉由于多次访问故障而处于断路器跳闸状态的服务，还有并发的连接数超过阈值的服务，然后对剩余的服务列表进行轮询 |
| ZoneAvoidanceRule         | RestTemplate默认使用，底层还是通过 RoundRobinRule实现 | 符合判断server所在区域的性能和server的可用性选择服务         |

- 实现 `IRule` 接口定义`choose`方法来选择服务，`AbstractLoadBalancerRule` 抽象类实现`IRule`，负载均衡策略类继承自`AbstractLoadBalancerRule` 见下图。

[![nGuhNT.png](https://s2.ax1x.com/2019/09/08/nGuhNT.png)](https://imgchr.com/i/nGuhNT)



- 为了测试，先自己写一个微服务框架，用`Eureka Server`注册中心单节点，端口 10001

```properties
# Eureka 服务端口号
server.port=10001
# 应用名
spring.application.name=spring-cloud-eureka-server
# Eureka 相关配置
# 1.主机名
eureka.instance.hostname=localhost
# 2.注册地址
eureka.client.service-url.defaultZone=http://${eureka.instance.hostname}:${server.port}/eureka/
# 3.单节点模式下关闭客户端行为 因为 Eureka Server 也可以互相注册作为集群
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
```

- `product-service` 产品相关 service ，这里做 2 个节点的集群，端口分别为20001,20002

```properties
# 另一个配置一样，20002
server.port=20001
# 服务的service id
spring.application.name=product-service
# 注册到10001
eureka.client.service-url.defaultZone=http://localhost:10001/eureka/
ribbon.eureka.enabled=true
```

暴露接口

```java
@RestController
@RequestMapping("/api/v1/product")
public class ProductController {
  
  	@Value("${server.port}")
    private String port;
  
    @GetMapping("port")
    public String getPort() {
        return "get port: " + port;
    }

}
```

- `inventory-service` 调用`product-service`服务，单节点，端口号30001

 ```properties
# 端口号
server.port=30001
# 服务ID
spring.application.name=inventory-service
eureka.client.service-url.defaultZone=http://localhost:10001/eureka/
ribbon.eureka.enabled=true
 ```

`maven` 需要添加`Ribbon`依赖，`Eureka Client` 也要加，这里就不写了

```xml
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
		</dependency>
```

`RibbonConfig` 配置`RestTemplate`

```java
@Configuration
public class RibbonConfig {
    
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate(){
        return new RestTemplate();
    }
}
```

添加 `Controller` 调用

```java
@RestController
@RequestMapping("/api/v1/inventory")
public class InventoryController {

    @Autowired
    private RestTemplate restTemplate;
  
   	@Autowired
    private LoadBalancerClient loadBalancerClient;


     @GetMapping("port")
    public String getPort() {
        for (int i = 0; i < 10; i++) {
            System.out.println(restTemplate.getForObject("http://product-service/api/v1/product/port", String.class));
        }


        return "ok";
    }

    @GetMapping("port1")
    public String getPort1() {
        for (int i = 0; i < 10; i++) {
            ServiceInstance instance = loadBalancerClient.choose("product-service");
            System.out.println(String.format("https://%s:%s", instance.getHost(), instance.getPort()));
        }
        return "ok";
    }
}
```

## 1. 默认轮询策略 RoundRobinRule 和 ZoneAvoidanceRule

- 启动服务，通过`PostMain`测试，控制台输出

```java
// 访问 http://localhost:30001/api/v1/inventory/port
get port: 20002
get port: 20001
get port: 20002
get port: 20001
get port: 20002
get port: 20001
get port: 20002
get port: 20001
get port: 20002
get port: 20001
  
//访问 http://localhost:30001/api/v1/inventory/port1
https://192.168.0.8:20001
https://192.168.0.8:20002
https://192.168.0.8:20001
https://192.168.0.8:20002
https://192.168.0.8:20001
https://192.168.0.8:20002
https://192.168.0.8:20001
https://192.168.0.8:20002
https://192.168.0.8:20001
https://192.168.0.8:20002
```

默认使用的是`RoundRobinRule` 轮询策略接下来看看源码

`ClientHttpRequestInterceptor`

```java
public class LoadBalancerInterceptor implements ClientHttpRequestInterceptor {
      public ClientHttpResponse intercept(final HttpRequest request, final byte[] body, final ClientHttpRequestExecution execution) throws IOException {
        URI originalUri = request.getURI();
        String serviceName = originalUri.getHost();
        Assert.state(serviceName != null, "Request URI does not contain a valid hostname: " + originalUri);
        // RestTemplate 通过 LoadBalancerInterceptor 获取服务
        return (ClientHttpResponse)this.loadBalancer.execute(serviceName, this.requestFactory.createRequest(request, body, execution));
    }
}
```

`RibbonLoadBalancerClient`

```java
public class RibbonLoadBalancerClient implements LoadBalancerClient {
  	@Override
	public <T> T execute(String serviceId, LoadBalancerRequest<T> request)
			throws IOException {
		return execute(serviceId, request, null);
	}
  
  	public <T> T execute(String serviceId, LoadBalancerRequest<T> request, Object hint)
			throws IOException {
      // 01.获得注入的 ILoadBalancer Bean
		ILoadBalancer loadBalancer = getLoadBalancer(serviceId);
      // 02.获得负载后的server
		Server server = getServer(loadBalancer, hint);
		if (server == null) {
			throw new IllegalStateException("No instances available for " + serviceId);
		}
		RibbonServer ribbonServer = new RibbonServer(serviceId, server,
				isSecure(server, serviceId),
				serverIntrospector(serviceId).getMetadata(server));

		return execute(serviceId, ribbonServer, request);
	}
  
  //02.处被调用
  protected Server getServer(ILoadBalancer loadBalancer, Object hint) {
		if (loadBalancer == null) {
			return null;
		}
		// Use 'default' on a null hint, or just pass it on?  
		return loadBalancer.chooseServer(hint != null ? hint : "default");
	}
}
```

`BaseLoadBalancer`

```java
public class BaseLoadBalancer extends AbstractLoadBalancer implements
        PrimeConnections.PrimeConnectionListener, IClientConfigAware {
  
  private final static IRule DEFAULT_RULE = new RoundRobinRule();
  //默认的负载策略(轮询)
  protected IRule rule = DEFAULT_RULE;
  
  //所有服务列表
      @Monitor(name = PREFIX + "AllServerList", type = DataSourceType.INFORMATIONAL)
    protected volatile List<Server> allServerList = Collections
            .synchronizedList(new ArrayList<Server>());
  //UP的服务列表
    @Monitor(name = PREFIX + "UpServerList", type = DataSourceType.INFORMATIONAL)
    protected volatile List<Server> upServerList = Collections
            .synchronizedList(new ArrayList<Server>());
  
  /* Ignore null rules 可以自定义负载策略*/ 
  public void setRule(IRule rule) {
        if (rule != null) {
            this.rule = rule;
        } else {
            /* default rule */
            this.rule = new RoundRobinRule();
        }
        if (this.rule.getLoadBalancer() != this) {
            this.rule.setLoadBalancer(this);
        }
    }
  
   public Server chooseServer(Object key) {
        if (counter == null) {
            counter = createCounter();
        }
        counter.increment();
        if (rule == null) {
            return null;
        } else {
            try {
              	//主要看这里  默认 ZoneAvoidanceRule 
                return rule.choose(key);
            } catch (Exception e) {
                logger.warn("LoadBalancer [{}]:  Error choosing server for key {}", name, key, e);
                return null;
            }
        }
    }
}
```

`ZoneAvoidanceRule 继承 PredicateBasedRule`

```java
public abstract class PredicateBasedRule extends ClientConfigEnabledRoundRobinRule {
  @Override
    public Server choose(Object key) {
        ILoadBalancer lb = getLoadBalancer();
      	//根据策略选择服务
        Optional<Server> server = getPredicate().chooseRoundRobinAfterFiltering(lb.getAllServers(), key);
        if (server.isPresent()) {
            return server.get();
        } else {
            return null;
        }       
    }
}
```

`AbstractServerPredicate`

```java
public abstract class AbstractServerPredicate implements Predicate<PredicateKey> {
 //筛选给定的服务器列表和负载平衡器键之后，以循环方式选择服务器
      public Optional<Server> chooseRoundRobinAfterFiltering(List<Server> servers, Object loadBalancerKey) {
        List<Server> eligible = getEligibleServers(servers, loadBalancerKey);
        if (eligible.size() == 0) {
            return Optional.absent();
        }
        return Optional.of(eligible.get(incrementAndGetModulo(eligible.size())));
    }
}
```

- 指定负载均衡规则 `RibbonConfig` 中添加

```java
@Configuration
public class RibbonConfig {

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate(){
        return new RestTemplate();
    }

    @Bean
    public IRule iRule(){
        //return new WeightedResponseTimeRule(); //权重
        //return new BestAvailableRule(); //最佳
        //return new RandomRule(); //随机
        return new RoundRobinRule(); //轮询
    }
}
```

`RoundRobinRule`

```java
public class RoundRobinRule extends AbstractLoadBalancerRule {
  
  public Server choose(ILoadBalancer lb, Object key) {
        if (lb == null) {
            log.warn("no load balancer");
            return null;
        }

        Server server = null;
        int count = 0;
        while (server == null && count++ < 10) {
          	// UP service
            List<Server> reachableServers = lb.getReachableServers();
          	// 所有 service
            List<Server> allServers = lb.getAllServers();
            int upCount = reachableServers.size();
            int serverCount = allServers.size();

            if ((upCount == 0) || (serverCount == 0)) {
                log.warn("No up servers available from load balancer: " + lb);
                return null;
            }
						//随机算法，所有service 取模
            int nextServerIndex = incrementAndGetModulo(serverCount);
          	//nextServerIndex 索引获取service
            server = allServers.get(nextServerIndex);
						//继续循环获取
            if (server == null) {
                /* Transient. */
                Thread.yield();
                continue;
            }
						// server 是否up 则返回
            if (server.isAlive() && (server.isReadyToServe())) {
                return (server);
            }

            // Next.
            server = null;
        }
				//循环10次后如果还是未得到，则报警
        if (count >= 10) {
            log.warn("No available alive servers after 10 tries from load balancer: "
                    + lb);
        }
        return server;
    }
  
}
```

## 2. 随机策略 RandomRule



```java
public class RandomRule extends AbstractLoadBalancerRule {
  /**
     * Randomly choose from all living servers
     */
    @edu.umd.cs.findbugs.annotations.SuppressWarnings(value = "RCN_REDUNDANT_NULLCHECK_OF_NULL_VALUE")
    public Server choose(ILoadBalancer lb, Object key) {
        if (lb == null) {
            return null;
        }
        Server server = null;

        while (server == null) {
            if (Thread.interrupted()) {
                return null;
            }
          	//存活服务列表
            List<Server> upList = lb.getReachableServers();
          	//所有服务列表
            List<Server> allList = lb.getAllServers();

            int serverCount = allList.size();
            if (serverCount == 0) {
                /*
                 * No servers. End regardless of pass, because subsequent passes
                 * only get more restrictive.
                 */
                return null;
            }
						//ThreadLocalRandom.current().nextInt(serverCount); 随机策略
            int index = chooseRandomInt(serverCount);
            server = upList.get(index);

            if (server == null) {
                /*
                 * The only time this should happen is if the server list were
                 * somehow trimmed. This is a transient condition. Retry after
                 * yielding.
                 */
                Thread.yield();
                continue;
            }

            if (server.isAlive()) {
                return (server);
            }

            // Shouldn't actually happen.. but must be transient or a bug.
            server = null;
            Thread.yield();
        }

        return server;

    }
}
```

```java
//输出 完全随机
https://192.168.0.8:20001
https://192.168.0.8:20002
https://192.168.0.8:20002
https://192.168.0.8:20002
https://192.168.0.8:20002
https://192.168.0.8:20002
https://192.168.0.8:20002
https://192.168.0.8:20001
https://192.168.0.8:20001
https://192.168.0.8:20001
```

## 3. 最佳策略 BestAvailableRule

跳过带有“跳闸”断路器的服务器并选择具有最低并发请求的服务器的规则。

```java
public class BestAvailableRule extends ClientConfigEnabledRoundRobinRule {

    private LoadBalancerStats loadBalancerStats;
    
    @Override
    public Server choose(Object key) {
        if (loadBalancerStats == null) {
            return super.choose(key);
        }
      	//所有服务器列表
        List<Server> serverList = getLoadBalancer().getAllServers();
        int minimalConcurrentConnections = Integer.MAX_VALUE;
        long currentTime = System.currentTimeMillis();
        Server chosen = null;
      	//循环
        for (Server server: serverList) {
          	//服务器状态
            ServerStats serverStats = loadBalancerStats.getSingleServerStat(server);
          	//是否触发断路
            if (!serverStats.isCircuitBreakerTripped(currentTime)) {
              // 获取当前服务器的请求个数
                int concurrentConnections = serverStats.getActiveRequestsCount(currentTime);
              // 比较所有服务请求个数，返回较低请求的服务
                if (concurrentConnections < minimalConcurrentConnections) {
                    minimalConcurrentConnections = concurrentConnections;
                    chosen = server;
                }
            }
        }
      // 如果没有选上，调用父类ClientConfigEnabledRoundRobinRule的choose方法，也就是使用RoundRobinRule轮询的方式进行负载均衡 
        if (chosen == null) {
            return super.choose(key);
        } else {
            return chosen;
        }
    }
  
}
```

## 4. 加权策略 WeightedResponseTimeRule

​	使用平均/百分位响应时间为每个服务器分配动态“权重”的规则，然后以“加权循环”方式使用该规则。刚启动时如果统计信息不足，则上有RoundRobinRule策略，等统计信息足够，会切换到WeightedResponseTimeRule

```java
public class WeightedResponseTimeRule extends RoundRobinRule {
  
  public Server choose(ILoadBalancer lb, Object key) {
        if (lb == null) {
            return null;
        }
        Server server = null;

        while (server == null) {
            // get hold of the current reference in case it is changed from the other thread
            List<Double> currentWeights = accumulatedWeights;
            if (Thread.interrupted()) {
                return null;
            }
            List<Server> allList = lb.getAllServers();

            int serverCount = allList.size();

            if (serverCount == 0) {
                return null;
            }

            int serverIndex = 0;

            // last one in the list is the sum of all weights
            double maxTotalWeight = currentWeights.size() == 0 ? 0 : currentWeights.get(currentWeights.size() - 1); 
            // No server has been hit yet and total weight is not initialized
            // fallback to use round robin
						// 先通过轮询来获得权重
            if (maxTotalWeight < 0.001d || serverCount != currentWeights.size()) {
                server =  super.choose(getLoadBalancer(), key);
                if(server == null) {
                    return server;
                }
            } else {
              	// 根据权重获得响应时间最快的服务
                // generate a random weight between 0 (inclusive) to maxTotalWeight (exclusive)
                double randomWeight = random.nextDouble() * maxTotalWeight;
                // pick the server index based on the randomIndex
                int n = 0;
                for (Double d : currentWeights) {
                    if (d >= randomWeight) {
                        serverIndex = n;
                        break;
                    } else {
                        n++;
                    }
                }

                server = allList.get(serverIndex);
            }

            if (server == null) {
                /* Transient. */
                Thread.yield();
                continue;
            }

            if (server.isAlive()) {
                return (server);
            }

            // Next.
            server = null;
        }
        return server;
    }
  
  
      class ServerWeight {

        public void maintainWeights() {
            ILoadBalancer lb = getLoadBalancer();
            if (lb == null) {
                return;
            }
            
            if (!serverWeightAssignmentInProgress.compareAndSet(false,  true))  {
                return; 
            }
            
            try {
                logger.info("Weight adjusting job started");
                AbstractLoadBalancer nlb = (AbstractLoadBalancer) lb;
                LoadBalancerStats stats = nlb.getLoadBalancerStats();
                if (stats == null) {
                    // no statistics, nothing to do
                    return;
                }
                double totalResponseTime = 0;
                // find maximal 95% response time
              //根据相应时间动态计算权重 
                for (Server server : nlb.getAllServers()) {
                    // this will automatically load the stats if not in cache
                    ServerStats ss = stats.getSingleServerStat(server);
                    totalResponseTime += ss.getResponseTimeAvg();
                }
                // weight for each server is (sum of responseTime of all servers - responseTime)
                // so that the longer the response time, the less the weight and the less likely to be chosen
                Double weightSoFar = 0.0;
                
                // create new list and hot swap the reference
                List<Double> finalWeights = new ArrayList<Double>();
              //根据相应时间动态计算权重 
                for (Server server : nlb.getAllServers()) {
                    ServerStats ss = stats.getSingleServerStat(server);
                    double weight = totalResponseTime - ss.getResponseTimeAvg();
                    weightSoFar += weight;
                    finalWeights.add(weightSoFar);   
                }
                setWeights(finalWeights);
            } catch (Exception e) {
                logger.error("Error calculating server weights", e);
            } finally {
                serverWeightAssignmentInProgress.set(false);
            }

        }
    }
  
}
```

