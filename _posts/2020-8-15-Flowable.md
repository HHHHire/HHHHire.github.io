---
layout:     post
title:      Flowable
subtitle:   Flowable基础
date:       2020-8-15
author:     HUA
header-img: img/home-bg-o.jpg
catalog: 	 true
tags:
    - 流程
---
# Flowable

## 定义：

流程定义：可以看成是重复执行的流程蓝图，一个流程定义可以启动多个流程实例

流程实例：一个具体的流程请求

流程由 **sequenceFlow** 控制，指定从哪步到哪步。

ProcessEngine是Flowable的核心

## 开始：

配置

ProcessEngineConfiguration配置类的实现：

*  **StandaloneProcessEngineConfiguration** ：流程引擎独立运行，Flowable自行处理事务，启动时检查数据库是否存在，不存在报错
*  **StandaloneInMemProcessEngineConfiguration** ：Flowable自行处理事务，默认使用h2内存数据库，启动时创建
*  **SpringProcessEngineConfiguration** ：集成spring环境时使用
*  **JtaProcessEngineConfiguration** ：引擎独立运行，使用Jta事务

数据库配置：

JDBC；JDBC数据源，MyBatis连接池；JNDI数据源

作业执行器

事件处理器(监听器)：用户可以自定义事件监听器，根据不同的事件响应。在创建流程实例时添加自定义事件监听器。也可以在流程文件中添加。但是只能作为<extensionElements>的子元素，声明在<process>上，不能在个别节点上。
事件分发。

## API

![](https://raw.githubusercontent.com/HHHHire/HHHHire.github.io/master/_posts/images/flowable1.png)

> ProcessEngine与(RepositoryService、RuntimeService)这些服务对象都是线程安全的

### 流程引擎API

通过`ProcessEngines.getDefaultProcessEngine()`获取流程引擎，它只会创建一个对象，重复调用也只会返回同一个流程引擎，线程安全。

**RepositoryService**提供了管理与控制`部署`和`流程定义`，提供静态信息

**RuntimeService**用于启动管理流程定义的新流程实例，读取存储流程变量

**TaskService**包含任务相关的，查询任务，创建独立的任务(没有关联到流程实例的任务)，将任务和用户关联，认领与完成任务

**IdentityService**用于管理组和用户

**FormService**可选服务，引入开始表单和任务表单

**HistoryService**暴露Flowable引擎收集的所有历史数据，提供查询

**ManagementService**可以读取数据库表与表原始数据的信息，也提供了对作业(job)的查询与管理

**DynamicBpmnService**用于修改流程定义中的部分内容，而不需要重新部署它

### 查询API

1. 通过TaskService添加各种查询条件，最终以AND连接
2. 通过原生的sql语句，更加灵活，可以通过managementService动态获取表名

### 变量

流程变量可以在创建流程实例时就指定，也可以在运行时再指定。

### 瞬时变量

与变量类似，但是不会去持久化，如果存在同名的变量和瞬时变量，那么会默认调用瞬时变量

## 集成Spring

在<Beans>标签下添加bean，创建流程的相关bean。
在使用ProcessEngineFactoryBean时，默认BPMN流程中的表达式是可以获取所有的Spring bean。

```xml
<!--例如这里在流程中调用了名称为hello的bean的hello()方法-->
<serviceTask id="print" flowable:expression="#{hello.hello()}" />
```

可以通过设置属性beans为空map，这样就获取不到Spring bean。

```xml
<bean id="map" class="java.util.HashMap"/>

<bean id="processEngineConfiguration" class="org.flowable.spring.SpringProcessEngineConfiguration">
    ...
    <property name="beans" ref="map"/>
</bean>
```

**自动部署资源** ：自动部署流程定义
在processEngineConfiguration下配置deploymentResource指定流程文件，还可以配置deploymentMode自定义部署。

## 集成springboot

只需要引入`flowable-spring-boot-starter`和`h2`依赖就行了，在/resource/processes目录下添加流程定义文件oneTask.bpmn20.xml，启动主类，springboot就会自动部署该目录下的所有流程定义。除此之外，springboot还自动创建h2数据库，并传递给流程引擎配置。还暴露ProcessEngine、FormEngine为bean，所有的服务RuntimeService、RepositoryService这些都暴露为bean。
可以直接自动注入flowable的服务，或者通过Context.getProcessEngineConfiguration().getRuntimeService()类似的调用。

### 更换数据源

h2数据库只是内存数据库，若要持久还需更换数据库，只需引入相应依赖，在配置文件中添加连接信息即可。

### 自定义引擎配置

实现EngineConfigurationConfigurer这个接口，获取引擎配置对象，使用@Bean注解，让它在流程引擎之前被调用

```java
public class MyConfigurer implements EngineConfigurationConfigurer<SpringProcessEngineConfiguration> {
    public void configure(SpringProcessEngineConfiguration processEngineConfiguration) {
        
    }
}
```

异步执行器

### 流程

startEvent是流程的入口点；
userTask表示流程中的人工任务，可以分配任务；流程在到达人工任务时会等待，等待处理完成后才继续往下走。如果任务分配给组，则组里的每个用户都可以处理这个任务。
流程到达空事件时结束；
各元素通过顺序流链接，即sourceRef到targetRef。

## 4.BPMN 2.0 结构

> 流程对象变量一个还多个，局部变量，边界事件？

### 4.1 事件

> 事件分为两种：抛出和捕获。抛出：当流程执行到这时，抛出一个事件，即触发一个触发器；捕获：当流程执行到这时，等待一个事件发生，即等待触发器动作。事件需要有事件定义。

#### 4.1.1 定时器事件 <timerEventDefinition>

定时器只能包含下列的一种属性，且只能在异步执行器启动的时候才能触发：

* timeDate：指定时间触发
* timeDuration：等待多久再触发

```xml
<!--等待十天-->
<timerEventDefinition>
    <timeDuration>P10D</timeDuration>
</timerEventDefinition>
```

* timecycle：指定重复周期触发，可以使用cron表达式，endDate可以指定停止日期，到了就不再重复

```xml
<!--每10小时触发，总共三次，到了endDate就会停止-->
<timerEventDefinition>
    <timeCycle flowable:endDate="2015-02-25T16:42:11+00:00">R3/PT10H</timeCycle>
</timerEventDefinition>
```

#### 4.1.2 错误事件

#### 4.1.3 信号事件

信号事件是一个全局事件，它会将其传播到所有激活的处理器。默认情况下，信号事件会向引擎全局范围进行广播。也就是说，一个流程实例发出信号事件，所有其他的流程实例都可以捕获该事件进行处理。
抛出->捕获->查询。通过intermediateThrowEvent抛出异常，intermediateCatchEvent捕获异常，通过`runtimeService.createExecutionQuery().signalEventSubscriptionName("alert").list()`查询

```xml
<!--声明信号-->
<signal id="alertSignal" name="alert"/>

<process id="oneTaskProcess" name="The One Task Process">
	...
    <!--信号事件抛出-->
    <intermediateThrowEvent id="throwSignalEvent" name="throwAlert">
        <!--信号事件定义-->
        <signalEventDefinition signalRef="alertSignal"/>
    </intermediateThrowEvent>
    <sequenceFlow id="flow3" sourceRef="throwSignalEvent" targetRef="catchSignalEvent"/>

    <!--信号事件捕获-->
    <intermediateCatchEvent id="catchSignalEvent" name="catchAlert">
        <signalEventDefinition signalRef="alertSignal"/>
    </intermediateCatchEvent>
    <sequenceFlow id="flow4" sourceRef="catchSignalEvent" targetRef="theEnd"/>
    ...
</process>
```

#### 4.1.4 消息事件

消息事件总是指向单个接收者

#### 4.1.5 启动事件

流程的起点，定义了如何启动流程，可以通过initiator来指明保存认证用户ID的变量名，流程启动，用户的ID就存在这个变量中。

空启动事件：没有指定 启动流程实例触发器的启动事件。通过runtimeService调用API方法启动。

定时器启动事件、消息启动器、信号启动器、错误启动器(不能用于启动流程实例，可触发事件子流程)

#### 4.1.6 结束事件

空结束事件：当到达这个事件时，没有指定抛出的结果

错误结束事件：结束执行的分支，并抛出错误。错误由匹配的错误边界中间事件捕获，如果没有就抛异常。

终止结束事件：结束第一个流程(子流程)，可以添加terminateAll属性，不论是否在子流程都会终结根流程。

取消结束事件：抛出取消事件由取消边界事件捕获，取消事务，触发补偿。

#### 4.1.7 边界事件

边界事件是捕获型事件，事件监听特定类型的触发器，当捕获到事件时，终止活动，沿着事件的出口顺序流

#### 4.1.8 中间事件

捕获、抛出中间事件

### 4.2 顺序流<sequenceFlow>

> 流程中在两个元素之间的连接器，在流程执行过程中，如果顺序流没有条件，在一个元素被访问后，会沿着它的所有出口出口顺序流继续**并行**执行。

条件顺序流用UEL表达式，会执行结果为true的顺序流，并行执行，但是不同的网关不同。可以指定一个默认顺序流，在其他条件顺序流都为false时，忽略默认顺序流上的条件并执行。

### 4.3 网关

> 控制流程的走向

#### 4.3.1 排他网关

根据出口顺序流上的条件进行计算，取最先定义的且计算结果为true的一条执行。

#### 4.3.2 并行网关

不计算出口顺序流上的条件，直接执行多条顺序流。并行网关可以合并和分支。如下图所示，分支：所有的出口顺序流程并发执行；合并：所有到达并行网关的并行执行都会在网关处等待，直到每一条入口顺序流都到达了，然后流程经过该合并网关合并成一条。 

![](https://raw.githubusercontent.com/HHHHire/HHHHire.github.io/master/_posts/images/flowable2.png)

#### 4.3.3 包容网关

包容网关会计算出口顺序流上的条件，然后根据条件去选择一条或多条出口顺序流。

#### 4.3.4 基于事件的网关

网关的每一个出口顺序流都要连接到一个捕获中间事件。一个基于事件的网关，必须要有两条或者更多的出口顺序流

![](https://raw.githubusercontent.com/HHHHire/HHHHire.github.io/master/_posts/images/flowable3.png)

当执行到达基于事件的网关时，流程执行暂停。流程实例订阅alert信号事件，并创建一个10分钟后触发的定时器。流程引擎会等待10分钟，并同时等待信号事件。如果信号在10分钟内触发，则会取消定时器，流程沿着信号继续执行，激活Handle alert用户任务。如果10分钟内没有触发信号，则会继续执行，并取消信号订阅。 

### 4.4 任务

#### 4.4.1 用户任务<userTask>

* <documentation>：用户任务描述
* 到期日期：flowable:dueDate
* 用户指派：用户任务指派给指定用户<humanPerformer>，指定后该任务只能在该用户的个人任务列表中看到。任务也可以放在候选任务列表中<potentialOwner>。更方便的方式：
  * assignee(办理人)
  * candidateUsers(候选用户)
  * candidateGroups(候选组)

#### JAVA服务任务

**指定**：

* `flowable:class`：根据全限定类型，指定流程执行时调用的类，通过该方式，该类的实例在所有的流程实例中都可以使用
* `flowable:delegateExpression`：使用bean名称，${beanName}
* `flowable:expression`：使用bean名称加方法名，#{beanName.method()}

**实现**：如果要实现在流程中调用的类，要实现JavaDetegate(委托)接口的excute方法。当流程执行到该步时，就会执行excut方法。

**字段注入**：通过<extensionElements>指定字段名和值，或者可以动态注入。

字段注入的线程安全

**服务任务的结果**：可以通过resultVariable指定为流程变量返回，再通过API获取流程变量

**异常处理**：

* 直接抛出：可以在Java委托中，直接抛出异常，引擎会捕获这个异常并转发到合适的处理器
* 异常映射：指定异常和errorCode，如果抛出该异常，引擎捕获它，并将它转换为带有给定errorCode的BPMN错误，然后像普通BPMN错误一样处理。
* 异常顺序流：在抛出异常时，通过API让它走指定的顺序流。

#### 邮件任务

邮件任务的实现为一种特殊的服务任务，将type定义为mail。类似的http任务也是一样。

### 4.5  子流程与调用服务

#### 4.5.1 子流程

子流程(sub-process)包含其他活动、网关、事件等的活动。

> 子流程只能有一个空启动事件，不能有其他类型的启动事件；顺序流不能跨越子流程的边界

事件子流程：通过事件触发的子流程，不能在事件子流程中使用空启动事件

事务子流程：子流程中的活动要么全部成功，要么就失败