# Sentinel

> 服务雪崩：由于一个服务出现了问题导致阻塞，而其他依赖它的服务也跟着阻塞，导致了整个服务挂掉
> 容错方案：隔离、超时、限流、熔断、降级

## 基础

### 隔离

在有故障时，将问题的印象控制在模块内部，不波及其他模块。例如线程池隔离、信号量隔离。
线程池隔离：通常一个请求过来占用tomcat线程池中的一个线程，当发生异常，这个线程就一直卡在那里。后面的请求过来也是，直到tomcat的线程池被用完，tomcat就无法响应更多的请求了。**解决**：在每个服务都定义一个线程池，在请求过来发生异常时卡住，当这个线程池被用完，后面的请求发现没有空余的线程可用，就不去调用这个服务了。

### 超时

上游服务调用下游服务，设置最大响应时间，超过这个时间，下游没有响应，就断开请求，释放线程。

### 限流

例如控制服务每秒处理请求的个数

### 熔断

当下游的服务响应变慢或者失败，上游服务就切断对它的调用。

* 熔断关闭状态：服务没有故障，熔断器的状态，对调用方的调用不做限制
* 熔断开启状态：服务发生故障，调用方直接执行fallback方法
* 半熔断状态：尝试恢复服务调用，通过有限的调用来检测服务是否已经恢复正常，来决定是否关闭熔断状态。

### 降级

为服务提供一个托底方法，一旦发现服务不可调用，直接调用托底方法。

## 控制台使用

### 流控规则

![](https://raw.githubusercontent.com/HHHHire/HHHHire.github.io/master/_posts/images/sentinel1.png)

* 资源名：默认时请求路径，可自定义
* 针对来源：指定对哪个微服务进行限流，默认default，指对所有服务限流
* 阈值类型：QPS，每秒请求数量；线程数，并发线程数。当达到单机阈值时限流
* 流控模式
  * 直接：最简单模式，在本接口达到阈值，就限流
  * 关联：在关联接口达到阈值(在这里指定的阈值)，那么本接口(就是指定接口)就会限流
  * 链路：当从某个接口过来的请求达到阈值时，对那个接口限流
* 流控效果
  * 快速失败：直接失败，抛出异常
  * Warm Up：从开始阈值(最大阈值的1/3)到最大阈值(即你设定的阈值)有一个缓冲阶段，在这期间阈值会慢慢增加到最大阈值。
  * 排队等待：就是多出来的请求去等待，会设置一个超时时间，超过超时时间还没被执行的请求，就会被丢掉。

### 降级规则

![](https://raw.githubusercontent.com/HHHHire/HHHHire.github.io/master/_posts/images/sentinel2.png)

* RT(平均响应时间)：当资源的请求响应时间超过阈值(RT)时，资源进入准降级状态，接下来1s内如果5个请求都超时，那么在时间窗口(s)内进入降级状态，过了时间窗口又恢复。注意RT单位为ms，上限为4900ms。
* 异常比例：当资源每秒异常总数占总请求数的比例高于阈值时，接下来的时间窗口时间内进入降级状态。
* 异常数：当资源一分钟内的异常数高于阈值，则在时间窗口内进入降级状态。注意，因为检测时分钟级的，所以时间窗口也尽量大于60s，如果小于60s，那么有可能你刚从降级状态中出来，又要进去。

> 问题：流控和降级的页面返回都是一样的，怎么区分，可以自定义返回异常。。

### 热点规则

![](https://raw.githubusercontent.com/HHHHire/HHHHire.github.io/master/_posts/images/sentinel3.png)

```java
@GetMapping("/test3")
@SentinelResource("test3")
public String test3(@RequestParam(required = false) String name, @RequestParam(required = false) Integer age) {
    return name + age;
}
```

代码中的@SentinelResource中的名称要和热点规则中的资源名一样，指定参数索引，0表示第一个参数。请求中只要携带第一个参数，当1s内请求次数达到阈值，就会限制请求。但是如果请求中只携带第二个参数，没有影响。
高级选项参数例外项可以对参数的具体值进行流控，此处的流控的意思和前面的单机阈值相同，但是指定参数值不受前面的影响。

### 授权规则

![](https://raw.githubusercontent.com/HHHHire/HHHHire.github.io/master/_posts/images/sentinel4.png)

流控应用即为指定的可以访问或不可访问的应用名。实现RequestOriginParser接口

```java
@Component
public class RequestOriginParserDefine implements RequestOriginParser {
    @Override
    public String parseOrigin(HttpServletRequest httpServletRequest) {
        return httpServletRequest.getParameter("serviceName");
    }
}
```

在访问路径后加上serviceName如：localhost:8090/test?serviceName=pay

### 系统规则

系统保护规则是从应用级别的入口流量进行控制，有

* Load(系统负载只在linux有效)：当系统负载超过阈值，并且系统的并发线程数超过系统容量时触发系统保护。
* RT(响应时间)：当系统所有入口的流量的平均RT超过阈值时触发系统保护。
* 线程数：当系统所有入口流量的并发线程数超过阈值时触发
* 入口QPS(每秒请求数)：当单台机器上所有入口流量的QPS超过阈值...
* CPU使用率：...

## @SentinelResource

在定义了资源点后，用dashboard指定限流等等，可以用@SentinelResource来指定异常处理。

![](https://raw.githubusercontent.com/HHHHire/HHHHire.github.io/master/_posts/images/sentinel5.png)

```java
@Service
public class ResourceService {
    @SentinelResource(
            value = "msg",
            blockHandler = "blockHandler",
            fallback = "fallback"
    )
    public String message() {
        int i = 1;
        if (i++ % 3 == 0) {
            throw new RuntimeException();
        }
        return "that's it";
    }

    /**
     * BlockException进入
     */
    public String blockHandler(BlockException e) {
        return "被限流了" + e;
    }

    /**
     * Throwable进入
     */
    public String fallback(Throwable t) {
        return "有异常" + t;
    }
}
```

## 规则持久化

sentinel配置的规则只是存到dashboard的内存中，可以持久化到redis、nacos、zk、file等。这里采用file方式。并且只有流控规则的持久化，其他规则类似。

```java
public class RulePersistence implements InitFunc {
    @Value("${spring.application.name}")
    private String appName;

    @Override
    public void init() throws Exception {
        String ruleDir = System.getProperty("user.dir") + "/sentinel-rules/" + appName;
        String flowRulePath = ruleDir + "/flow-rule.json";
        System.out.println(flowRulePath);
        createDir(ruleDir);
        createFile(flowRulePath);

        ReadableDataSource<String, List<FlowRule>> dataSource = new FileRefreshableDataSource<>(flowRulePath, flowRuleListParser);
        FlowRuleManager.register2Property(dataSource.getProperty());
        FileWritableDataSource<List<FlowRule>> writableDataSource = new FileWritableDataSource<>(flowRulePath, this::encoding);
        WritableDataSourceRegistry.registerFlowDataSource(writableDataSource);
    }	

    private Converter<String, List<FlowRule>> flowRuleListParser = source -> JSON.parseObject(source, new TypeReference<List<FlowRule>>() {
    });

    private void createDir(String filePath) {...}

    private void createFile(String filePath) {...}

    private <T> String encoding(T t) {...}
}
```

同时还需要借助SPI来实现调用。

## Feign整合Sentinel

在配置文件中开启Feign对Sentinel的支持

```yml
feign:
  sentinel:
  	enabled: true
```

可以在@Feign的fallback属性中添加容错类，创建容错类并实现@Feign的接口的容错方法。
如果想要在容错类中拿到具体的错误信息，可以在@Feign的fallbackFactory添加容错工厂，创建容错工厂实现FallbackFactory<Feign接口>，可以拿到错误信息。

> fallback和fallbackFactory不能同时使用

