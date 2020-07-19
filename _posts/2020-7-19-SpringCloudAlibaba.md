# SpringCloudAlibaba

[学习demo](https://github.com/HHHHire/learn/blob/master/springcloud-alibaba-demo)

## 组件

* Nacos：服务发现、配置管理和服务管理平台
* Sentinel：流量控制、熔断降级、系统负载保护，保护服务的稳定性
* Dubbo：远程RPC调用
* RocketMQ：分布式消息系统，基于高可用分布式集群技术，提供低延时的、高可靠的消息发布与订阅服务
* Seata：一个易于使用的高性能微服务分布式事务解决方案 
* ACM：在分布式架构环境中对应用配置进行集中管理和推送的应用配置中心产品 
* OSS：阿里云对象存储服务
* SchedulerX：分布式任务调度产品
* SMS：短信服务

版本控制

| Spring Cloud Version        | Spring Cloud Alibaba Version | Spring Boot Version |
| --------------------------- | ---------------------------- | ------------------- |
| Spring Cloud Hoxton.SR3     | 2.2.1.RELEASE                | 2.2.5.RELEASE       |
| Spring Cloud Hoxton.RELEASE | 2.2.0.RELEASE                | 2.2.X.RELEASE       |
| Spring Cloud Greenwich      | 2.1.2.RELEASE                | 2.1.X.RELEASE       |
| Spring Cloud Finchley       | 2.0.2.RELEASE                | 2.0.X.RELEASE       |
| Spring Cloud Edgware        | 1.5.1.RELEASE                | 1.5.X.RELEASE       |

只要在父模块中引入依赖，在子模块中就不用在指定版本号，由dependencies统一管理。

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-alibaba-dependencies</artifactId>
    <version>${spring-cloud.alibaba.version}</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-dependencies</artifactId>
    <version>${spring-cloud.version}</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
```

## Nacos&Nacos Config

## 前哨（Sentinel）

> Sentinel以“流量”为切入点，在流量控制，断路和负载保护等多方面开展工作，以保证服务可用性。
>
> [Sentinel文档](https://sentinelguard.io/zh-cn/docs/introduction.html)

依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
```

@SentinelResource用于标识资源是否受速率限制或降级

```java
@RestController
public class TestController {
    @GetMapping("/mono")
    @SentinelResource("hello")
    public Mono<String> mono() {
	return Mono.just("simple string")
			.transform(new SentinelReactorTransformer<>("otherResourceName"));
    }
}
```

### 前哨仪表板

Sentinel仪表板是一个轻量级的控制台，提供诸如机器发现，单服务器资源监视，群集资源数据概述以及规则管理等功能。下载仪表板jar包，运行。

配置仪表板

```yml
spring:
  cloud:
    sentinel:
      transport:
        port: 8719 # HTTP Server 端口
        dashboard: localhost:8080 # dashboard 路径:端口
```

在application.yml中指定端口，将在应用程序的相应服务器上启动HTTP Server，并且与Sentinel仪表板交互。例如，如果在Sentinel仪表板中添加了速率限制规则，则该规则先被推送到HTTP Server并 由HTTP Server接收，然后HTTP Server将规则注册到Sentinel。 

### OpenFeign

Sentinel与OpenFeign组件兼容，若要使用OpenFeign首先要引入依赖，由前面的spring-cloud-dependencies指定版本。

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

在配置文件中启用Sentinel

```yml
feign.sentinel.enabled=true
```

使用openFeign

```java
@FeignClient(name = "nacos-provider", fallback = EchoServiceFallback.class, configuration = FeignConfiguration.class)
public interface EchoService {
    @GetMapping("/echo/{s}")
    String test(@PathVariable String s);
}

public class EchoServiceFallback implements EchoService {
    @Override
    public String test(@PathVariable String s) {
        return "echo fallback";
    }
}

public class FeignConfiguration {
    @Bean
    public EchoServiceFallback echoServiceFallback() {
        return new EchoServiceFallback();
    }
}

// 调用feign
@Resource
private EchoService echoservice;
@GetMapping("/feign")
public String feign() {
    echoService.test("lihua");
    return null;
}

// 最后须在启动类上添加注解@EnableFeignClients,否则扫描不到
```

### RestTemplate

Sentinel支持对RestTemplate的服务调用使用Sentinel进行保护，在构造RestTemplate bean的时候需要加上@SentinelRestTemplate注解。

@SentinelRestTemplate注解的属性支持限流(blockHandler，blockHandlerClass)和降级(fallback，fallbackClass)的处理。其中blockHandler和fallback属性的方法是blockHandlerClass或fallbackClass中的静态方法，

如下所示：

```java
@Bean
@com.alibaba.cloud.sentinel.annotation.SentinelRestTemplate(blockHandler = "handleException", blockHandlerClass = ExceptionUtil.class)
public RestTemplate restTemplate() {
    return new RestTemplate();
}

public class ExceptionUtil {
    public static ClientHttpResponse handleException(HttpRequest request, byte[] body, ClientHttpRequestExecution execution, BlockException exception) {
        return null;
    }
}
```

在使用RestTemplate调用被Sentinel熔断后，可以自定义返回信息。

Sentinel RestTemplate限流规则的粒度：

* 协议、主机、端口、路径
* 协议、主机、端口

### 动态数据源

### Zuul

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-alibaba-sentinel-gateway</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
</dependency>
```

### Gateway

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-alibaba-sentinel-gateway</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

### 断路器

为所有断路器提供默认配置，创建一个Customizer通过SentinelCircuitBreakerFactory或者ReactiveSentinelCircuit-BreakerFactory传递的bean，添加短路规则。

还可以提供一个特定的断路器配置。

## Dubbo

## RocketMQ Binder

### Spring Cloud Stream

Spring Cloud Stream是用于构建基于消息的微服务应用框架。

Spring Cloud Stream内部有两个概念：Binder和Binding

* Binder：跟外部消息中间件继承的组件，用来创建Binding，每个消息中间件都有自己的Binder实现。

例如RabbitMQ的RabbitMessageChannelBinder和RocketMQ的RocketMQMessage

* Binding：包括Input Binding和Output Binding。Binding在消息中间件与应用程序提供的Provider和Consumer直接提供了一个桥梁，实现了开发者只需使用应用程序的 Provider 或 Consumer 生产或消费数据即可，屏蔽了开发者与底层消息中间件的接触

#### Spring Cloud Alibaba RocketMQ Binder

引入RocketMQ Binder所需的依赖，二选一即可

```xml
<dependency>
  <groupId>com.alibaba.cloud</groupId>
  <artifactId>spring-cloud-stream-binder-rocketmq</artifactId>
</dependency>
<!-- 两个依赖二选一 -->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-rocketmq</artifactId>
</dependency>
```

RocketMQ Binder的实现依赖于`RocketMQ-Spring`框架，该框架是RocketMQ和Springboot的整合，主要提供了三个特性：

1. 使用RocketMQTemplate发送消息
2. 通过@RocketMQTransactionListener来监听事务消息和回查
3. 通过@RocketMQMessageListener来消费消息

使用：

1. 配置文件

```properties
# consumer端
spring.cloud.stream.rocketmq.binder.name-server=127.0.0.1:9876

spring.cloud.stream.bindings.input1.destination=test-topic
spring.cloud.stream.bindings.input1.content-type=text/plain
spring.cloud.stream.bindings.input1.group=test-group1
spring.cloud.stream.rocketmq.bindings.input1.consumer.orderly=true

server.port=28082
spring.application.name=rocketmq-consumer-example
```

```properties
# produce端
server.port=28081
spring.application.name=rocketmq-produce-example

spring.cloud.stream.rocketmq.binder.name-server=127.0.0.1:9876

# 定义name为output的binding
spring.cloud.stream.bindings.output1.destination=test-topic
spring.cloud.stream.bindings.output1.content-type=application/json
```

2. 启动类，指定Sink或Source

```java
@EnableBinding(MySink.class)
```

3. 配置接口，MySink接口，与启动类上的对应。MySource的接口也一样，不过是output

```java
public interface MySink {
    @Input("input1")
    SubscribableChannel input1();
}
```

4. 消息发送，这里采用的CommandLineRunner方式，在应用初始化后执行该代码。

```java
@Component
public class ProducerRunner implements CommandLineRunner {

    @Autowired
    private MessageChannel output1;

    @Override
    public void run(String... args) throws Exception {
        Map<String, Object> headers = new HashMap<>();
        headers.put(MessageConst.PROPERTY_TAGS, "tagStr");
        Message message = MessageBuilder.createMessage("hello", new MessageHeaders(headers));
        System.out.println("send...");
        output1.send(message);

    }
}
```

5. 消息接收

```java
@Service
public class ReceiveService {
    @StreamListener("input1")
    public void receiveInput1(String revieveMsg) {
        System.out.println("input1 receive: " + revieveMsg);
    }
}
```

6. 结果

![](../../images/springcloudAlibaba1.png)

![](../../images/springcloudAlibaba2.png)

