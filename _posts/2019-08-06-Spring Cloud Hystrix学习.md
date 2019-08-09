---
layout:     post
title:      Spring Cloud Hystrix 断路器学习
subtitle:   Spring Cloud Hystrix 断路器学习
date:       2019-08-06
author:     金志宏
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - Java
    - Netflix
    - Spring Cloud
    - 微服务
---
#Spring Cloud Hystrix 断路器学习

# 1. Hystrix 介绍

Netflix创建了一个名为[Hystrix](https://github.com/Netflix/Hystrix)的库，用于实现[断路器模式](https://martinfowler.com/bliki/CircuitBreaker.html)。在微服务架构中，通常有多层服务调用，如以下示例所示。

[][https://raw.githubusercontent.com/spring-cloud/spring-cloud-netflix/master/docs/src/main/asciidoc/images/Hystrix.png]

![Hystrix](https://raw.githubusercontent.com/spring-cloud/spring-cloud-netflix/master/docs/src/main/asciidoc/images/Hystrix.png)

较低级别的服务中的服务故障可能导致级联故障一直到用户。当对特定服务的调用超过 `circuitBreaker.requestVolumeThreshold` （默认20个请求），并且故障百分比大于（默认 > 50%）在由 `metrics.rollingStats.timeInMilliseconds` 定义的滚动窗口中，断路器会打开且不进行呼叫。在出现错误或者开路情况下，开发人员可以进行回退。 `FallBack`。

![FallBack](https://raw.githubusercontent.com/spring-cloud/spring-cloud-netflix/master/docs/src/main/asciidoc/images/HystrixFallback.png)



# 2. 使用 Hystrix

在 Spring Cloud 中使用 Hystrix 很简单，我们只要在 Maven 中引入依赖即可。

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
        </dependency>
```

在 Spring Boot 中使用 `@EnableCircuitBreaker` 注解打开断路器

```java
@SpringBootApplication
@EnableCircuitBreaker
public class SpringCloudEurekaConsumerApplication  {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudEurekaConsumerApplication.class, args);
    }

}
```

## 2.1 Provider

做个简单的 `Demo` 实例

`Eureka Server` 就不介绍怎么搭建了，第一个 `Eureka Client` 作为 `Provider` 端口号 20001 注册到本地端口号 10001 的注册中心。

```properties
# Eureka 客户端口号
server.port=20001
# 应用名
spring.application.name=service-provider
# Eureka 相关配置
# 1.注册地址 注册到 Eureka Server 服务
eureka.client.service-url.defaultZone=http://localhost:10001/eureka/
```

编写`Controller` 提供接口。定义两个方法，一个随机抛出异常，一个随机进行3-7秒的休眠，以测试超时。

```java
@RestController
public class DemoController {

    @GetMapping("/timeout")
    public String timeout() throws InterruptedException {
        //随机休眠 3-7 秒 等待超时
        int timeout = (int)(3000 + Math.random() * 4000);
        TimeUnit.MILLISECONDS.sleep(timeout);
        return "timeout() 返回成功休眠时间: " + timeout;
    }

    @GetMapping("/exception")
    public String exception() {
        //随机抛出异常
        if (System.currentTimeMillis() % 2 == 1) {
            throw new RuntimeException("random exception");
        }
        return "ok";
    }
}
```

## 2.2 Consumer

`application` 配置文件

``` properties
# Eureka 客户端口号
server.port=20011
# 应用名
spring.application.name=service-consumer
# Eureka 相关配置
# 1.注册地址 注册到 Eureka Server 服务
eureka.client.service-url.defaultZone=http://localhost:10001/eureka/
```

Spring Boot 启动类添加注解启用断路器

```java
@SpringBootApplication
@EnableCircuitBreaker
public class SpringCloudEurekaConsumerApplication  {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudEurekaConsumerApplication.class, args);
    }

}
```

`Service 接口` 在接口中定义了2个`timeout()`方法，一个使用了注解方式定义了`execution.isolation.thread.timeoutInMilliseconds`超时时间配置，单位是毫秒。如果未定义，则`HystrixCommand`使用的是默认配置1000毫秒。

```java
@Component
public class DemoService {

    public static final Logger LOGGER = LoggerFactory.getLogger(DemoService.class);

    private String timeout() throws IOException {
        LOGGER.info("start query timeout endpoint");
        URL url = new URL("http://localhost:20001/timeout");
        BufferedReader reader = new BufferedReader(new InputStreamReader(url.openStream()));
        String s;
        StringBuffer stringbuffer = new StringBuffer();
        while ((s = reader.readLine()) != null) {
            stringbuffer.append(s);
        }
        return stringbuffer.toString();
    }

    @HystrixCommand(fallbackMethod = "timeoutFallBack" )
    public String timeout1() throws IOException {
        String result = timeout();
        LOGGER.info("timeout1() -> " + result);
        return result;
    }

    @HystrixCommand(fallbackMethod = "timeoutFallBack",
    commandProperties = {
            @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds",value = "5000")
    })
    public String timeout2() throws IOException {
        String result = timeout();
        LOGGER.info("timeout2() -> " + result);
        return result;
    }

    public String timeoutFallBack(){
        String result = "服务超时，降级不可用";
        LOGGER.info(result);
        return result;
    }

    @HystrixCommand(fallbackMethod = "exceptionFallBack")
    public String exception() throws IOException {
        LOGGER.info("start query timeout endpoint");
        URL url = new URL("http://localhost:20001/exception");
        byte[] result = new byte[1024];
        url.openStream().read(result);
        return new String(result);
    }

    public String exceptionFallBack() {
        String result = "服务出错，降级不可用";
        LOGGER.info(result);
        return result;
    }
}
```

`Controller` 定义Controller的RestFul接口，接下来我们就可以测试。

```java
@RestController
public class DemoController {
    
    @Autowired
    private DemoService demoService;


    @GetMapping("/timeout1")
    public String timeout() throws IOException {
        return demoService.timeout1();
    }

    @GetMapping("/timeout2")
    public String timeout2() throws IOException {
        return demoService.timeout2();
    }

    @GetMapping("/exception")
    public String exception() throws IOException {
        return demoService.exception();
    }
}
```

测试结果

1. 访问 http://localhost:20011/timeout1 约1秒后返回结果为：服务超时，降级不可用
2. 访问 http://localhost:20011/timeout2 会出现2种情况，如果休眠时间超过5秒，则返回：服务超时，降级可用，如果休眠时间小于5秒则返回：timeout2() 返回成功休眠时间: 3410
3. 访问 http://localhost:20011/exception 每次都会触发降级

```java
//输出
start query timeout endpoint          
timeoutFallBack() -> 服务超时，降级不可用       
timeout1() -> timeout() 返回成功休眠时间: 4547
start query timeout endpoint          
timeoutFallBack() -> 服务超时，降级不可用       
timeout1() -> timeout() 返回成功休眠时间: 3040
start query timeout endpoint          
timeout2() -> timeout() 返回成功休眠时间: 3410
start query timeout endpoint          
timeoutFallBack() -> 服务超时，降级不可用       
timeout2() -> timeout() 返回成功休眠时间: 5098
```

从返回可以看出，对于 `timeout1()` 因为使用了默认配置，所以对于接口至少3-7秒的调用时间，必然会进`timeoutFallBack()`触发降级，而对于 `timeout2()` 则5000毫秒以上延迟触发降级，否则调用成功。虽然触发降级，但发现`timeout1() 和 timeout2()`最终还是会被执行完。

## 2.3 源码分析

超时熔断的实现原理

[![eqbXq0.png](https://s2.ax1x.com/2019/08/10/eqbXq0.png)](https://imgchr.com/i/eqbXq0)



被 `@HystrixCommand` 注解的接口会通过实例一个 `GenericCommand` ，他继承 `AbstractHystrixCommand`

```java
public class GenericCommand extends AbstractHystrixCommand<Object> {
  //HystrixCommandBuilder builder 模式
  public GenericCommand(HystrixCommandBuilder builder) {
    //超类构造器
     super(builder);
  }
}
```

`AbstractHystrixCommand`

```java
public abstract class AbstractHystrixCommand<T> extends HystrixCommand<T> {
    protected AbstractHystrixCommand(HystrixCommandBuilder builder) {
      	// 超类构造器 builder 模式 初始化参数
        super(builder.getSetterBuilder().build());
        // 接口方法
        this.commandActions = builder.getCommandActions();
        this.collapsedRequests = builder.getCollapsedRequests();
        this.cacheResultInvocationContext = builder.getCacheResultInvocationContext();
        this.cacheRemoveInvocationContext = builder.getCacheRemoveInvocationContext();
        this.ignoreExceptions = builder.getIgnoreExceptions();
        this.executionType = builder.getExecutionType();
    }
}
```

`AbstractCommand`

```java
abstract class AbstractCommand<R> implements HystrixInvokableInfo<R>, HystrixObservable<R> {
  
  // 初始化构造器
  protected AbstractCommand(HystrixCommandGroupKey group, HystrixCommandKey key, HystrixThreadPoolKey threadPoolKey, HystrixCircuitBreaker circuitBreaker, HystrixThreadPool threadPool,
            HystrixCommandProperties.Setter commandPropertiesDefaults, HystrixThreadPoolProperties.Setter threadPoolPropertiesDefaults,
            HystrixCommandMetrics metrics, TryableSemaphore fallbackSemaphore, TryableSemaphore executionSemaphore,
            HystrixPropertiesStrategy propertiesStrategy, HystrixCommandExecutionHook executionHook) {

        this.commandGroup = initGroupKey(group);
        this.commandKey = initCommandKey(key, getClass());
    //这里初始化properties
        this.properties = initCommandProperties(this.commandKey, propertiesStrategy, commandPropertiesDefaults);
        this.threadPoolKey = initThreadPoolKey(threadPoolKey, this.commandGroup, this.properties.executionIsolationThreadPoolKeyOverride().get());
        this.metrics = initMetrics(metrics, this.commandGroup, this.threadPoolKey, this.commandKey, this.properties);
        this.circuitBreaker = initCircuitBreaker(this.properties.circuitBreakerEnabled().get(), circuitBreaker, this.commandGroup, this.commandKey, this.properties, this.metrics);
        this.threadPool = initThreadPool(threadPool, this.threadPoolKey, threadPoolPropertiesDefaults);

        //Strategies from plugins
        this.eventNotifier = HystrixPlugins.getInstance().getEventNotifier();
        this.concurrencyStrategy = HystrixPlugins.getInstance().getConcurrencyStrategy();
        HystrixMetricsPublisherFactory.createOrRetrievePublisherForCommand(this.commandKey, this.commandGroup, this.metrics, this.circuitBreaker, this.properties);
        this.executionHook = initExecutionHook(executionHook);

        this.requestCache = HystrixRequestCache.getInstance(this.commandKey, this.concurrencyStrategy);
        this.currentRequestLog = initRequestLog(this.properties.requestLogEnabled().get(), this.concurrencyStrategy);

        /* fallback semaphore override if applicable */
        this.fallbackSemaphoreOverride = fallbackSemaphore;

        /* execution semaphore override if applicable */
        this.executionSemaphoreOverride = executionSemaphore;
    }
  
  
  // 自定义配置设置
      private static HystrixCommandProperties initCommandProperties(HystrixCommandKey commandKey, HystrixPropertiesStrategy propertiesStrategy, HystrixCommandProperties.Setter commandPropertiesDefaults) {
        if (propertiesStrategy == null) {
            return HystrixPropertiesFactory.getCommandProperties(commandKey, commandPropertiesDefaults);
        } else {
            // used for unit testing
            return propertiesStrategy.getCommandProperties(commandKey, commandPropertiesDefaults);
        }
    }
  
}

private Observable<R> executeCommandAndObserve(final AbstractCommand<R> _cmd) {
  
  //如果开启超时检查
   Observable<R> execution;
        if (properties.executionTimeoutEnabled().get()) {
            execution = executeCommandWithSpecifiedIsolation(_cmd)
                    .lift(new HystrixObservableTimeoutOperator<R>(_cmd));
        } else {
            execution = executeCommandWithSpecifiedIsolation(_cmd);
        }

}


 private static class HystrixObservableTimeoutOperator<R> implements Operator<R, R> {
   
   //观察者模式，订阅执行状态
    		@Override
        public Subscriber<? super R> call(final Subscriber<? super R> child) {
          
          // 这里比较乱，定义超时监听器
          TimerListener listener = new TimerListener() {
            	@Override
                public void tick() {
                    // if we can go from NOT_EXECUTED to TIMED_OUT then we do the timeout codepath
                    // otherwise it means we lost a race and the run() execution completed or did not start
                    if (originalCommand.isCommandTimedOut.compareAndSet(TimedOutStatus.NOT_EXECUTED, TimedOutStatus.TIMED_OUT)) {
                        // report timeout failure
                        originalCommand.eventNotifier.markEvent(HystrixEventType.TIMEOUT, originalCommand.commandKey);

                        // shut down the original request
                        s.unsubscribe();

                        timeoutRunnable.run();
                        //if it did not start, then we need to mark a command start for concurrency metrics, and then issue the timeout
                    }
                }

                @Override
                public int getIntervalTimeInMilliseconds() {
                    return originalCommand.properties.executionTimeoutInMilliseconds().get();
                }
          }
          //返回自定义配置的超时时间
          @Override
                public int getIntervalTimeInMilliseconds() {
                    return originalCommand.properties.executionTimeoutInMilliseconds().get();
                }
        }
   
 }



```



`HystrixCommandProperties` 参数配置

```java
public abstract class HystrixCommandProperties {

	//默认1000毫秒熔断
	private static final Integer default_executionTimeoutInMilliseconds = 1000; 

	//自定义配置
	private final HystrixProperty<Integer> executionTimeoutInMilliseconds;
  
}
```

`HystrixPropertiesFactory` 参数工厂 

```java
public class HystrixPropertiesFactory {
  //通过 CommandKey.name() 获得参数缓存
      private static final ConcurrentHashMap<String, HystrixCommandProperties> commandProperties = new ConcurrentHashMap<String, HystrixCommandProperties>();
  
  //获得参数
  public static HystrixCommandProperties getCommandProperties(HystrixCommandKey key, HystrixCommandProperties.Setter builder) {
        HystrixPropertiesStrategy hystrixPropertiesStrategy = HystrixPlugins.getInstance().getPropertiesStrategy();
        String cacheKey = hystrixPropertiesStrategy.getCommandPropertiesCacheKey(key, builder);
        if (cacheKey != null) {
            HystrixCommandProperties properties = commandProperties.get(cacheKey);
            if (properties != null) {
                return properties;
            } else {
                if (builder == null) {
                    builder = HystrixCommandProperties.Setter();
                }
                // create new instance
                properties = hystrixPropertiesStrategy.getCommandProperties(key, builder);
                // cache and return
                HystrixCommandProperties existing = commandProperties.putIfAbsent(cacheKey, properties);
                if (existing == null) {
                    return properties;
                } else {
                    return existing;
                }
            }
        } else {
            // no cacheKey so we generate it with caching
            return hystrixPropertiesStrategy.getCommandProperties(key, builder);
        }
    }
  
}
```

``

```java
// HystrixCommand 工厂
public class HystrixCommandFactory {
    private static final HystrixCommandFactory INSTANCE = new HystrixCommandFactory();

    private HystrixCommandFactory() {
    }

    public static HystrixCommandFactory getInstance() {
        return INSTANCE;
    }

  // 创建 HystrixCommand
    public HystrixInvokable create(MetaHolder metaHolder) {
        Object executable;
        if (metaHolder.isCollapserAnnotationPresent()) {
            executable = new CommandCollapser(metaHolder);
        } else if (metaHolder.isObservable()) {
            executable = new GenericObservableCommand(HystrixCommandBuilderFactory.getInstance().create(metaHolder));
        } else {
            executable = new GenericCommand(HystrixCommandBuilderFactory.getInstance().create(metaHolder));
        }

        return (HystrixInvokable)executable;
    }
  
}
```

`HystrixCommand` 执行

```java
public abstract class HystrixCommand<R> extends AbstractCommand<R> implements HystrixExecutable<R>, HystrixInvokableInfo<R>, HystrixObservable<R> {
   		// 方法执行
      public R execute() {
        try {
            return queue().get();
        } catch (Exception e) {
            throw Exceptions.sneakyThrow(decomposeException(e));
        }
    }
  
}
```

`HystrixTimer`

```java
public class HystrixTimer {
  // TimerListener 引用 
  public Reference<TimerListener> addTimerListener(final TimerListener listener) {
        startThreadIfNeeded();
        // add the listener

        Runnable r = new Runnable() {

            @Override
            public void run() {
                try {
                    listener.tick();
                } catch (Exception e) {
                    logger.error("Failed while ticking TimerListener", e);
                }
            }
        };

        ScheduledFuture<?> f = executor.get().getThreadPool().scheduleAtFixedRate(r, listener.getIntervalTimeInMilliseconds(), listener.getIntervalTimeInMilliseconds(), TimeUnit.MILLISECONDS);
        return new TimerReference(listener, f);
    }
}
```







>[Spring Coud 官网][https://cloud.spring.io/spring-cloud-netflix/reference/html/#_circuit_breaker_hystrix_clients]
>
>



