# RocketMQ

> [RocketMQ](https://github.com/apache/rocketmq/blob/master/docs/cn/README.md)

## 基本概念

![](https://raw.githubusercontent.com/HHHHire/HHHHire.github.io/master/_posts/images/RocketMQ2.png)

如图所示，整体可以分为四个角色：NameServer、Broker、Producer、Consumer

* **Broker**(邮递员)：RocketMQ的核心，负责消息的接受、存储、投递。

* **NameServer**(邮局)：消息队列的协调者，Broker向它注册路由信息，Producer和Consumer向它获取路由信息，查找各主题相应的Broker IP列表。
* **Producer**(消息生产者-寄件人)：消息生产者，需要从NameServer获取Broker信息，与Broker建立连接，向Broker服务器发送消息。
* **Consumer**(消息消费者-收件人)：消息消费者，需要从NameServer获取Broker信息，与Broker建立连接，从Broker服务器获取消息。有两种消费方式：拉取消费pull、推送消费push。
* **Topic**(主题)：表示一类消息的集合，每个主题包含若干条消息，每条消息只能属于一个主题。用来区分不同类型的消息，发送和接收消息前都需要先创建Topic，针对Topic来发送和接收消息 。
* **Pull Consumer**(拉取式消费)：消费者主动去Broker服务器拉取消息。
* **Push Consumer**(推送式消费)：Broker服务器收到消息后主动推送给消费者，实时性高。
* **Message Queue**(邮件)： 为了提高性能和吞吐量，引入了Message Queue，一个Topic可以设置一个或多个Message Queue，这样消息就可以并行往各个Message Queue发送消息，消费者也可以并行的从多个 Message Queue读取消息 。
* **Message**：消息载体
* **Produce Group**：生产者组，简单来说就是多个发送同一类消息的生产者称之为一个生产者组。
* **Consumer Group**：消费者组，消费同一类消息的多个 consumer 实例组成一个消费者组，消费者组内的消费者必须订阅同一个Topic 。
* **Clustering**(集群消费)：该模式下的消费者会平摊消息，即每条消息只被一个消费者消费
* **Broadcasting**(广播消费)：该模式下每个消费者都能消费全部的消息。
* **Tag**(标签)：在同一主题下区分不同的消息。

## 不同类型的消息

### 普通消息

RocketMQ提供三种消息发送方式：可靠同步发送、可靠异步发送、单向发送

* 可靠同步发送：消息发送方发送消息后，在接收方返回响应之后才会继续发送。例如：重要邮件通知
* 可靠异步发送：发送方发送消息不等接收方的响应，继续发送，通过接口回调接收接收方的响应，并作处理
* 单向发送：发送方只是发送消息，不等接收方响应，也不去回调接口。

### 顺序消息



### 事务消息

![](https://raw.githubusercontent.com/HHHHire/HHHHire.github.io/master/_posts/images/RocketMQ1.png)

### 概念：

* `半事务消息`： 暂不能投递的消息，发送方已经成功地将消息发送到了RocketMQ服务端，但是服务端未收到生产者对该消息的二次确认，此时该消息被标记成“暂不能投递”状态，处于该种状态下的消息即`半事务消息`。 
* `消息回查`： 由于网络闪断、生产者应用重启等原因，导致某条事务消息的二次确认丢失，RocketMQ服务端通过扫描发现某条消息长期处于“半事务消息”时，需要主动向消息生产者询问该 消息的最终状态（Commit 或是 Rollback），该询问过程即消息回查，即图中的第5步。 

### 事务消息发送流程(图中黑色流程)：

1. 发送方将半事务消息发送至RocketMQ服务端。 
2. RocketMQ服务端将消息持久化之后，向发送方返回Ack确认消息已经发送成功，此时消息为半事务消息。
3. 发送方开始执行本地事务逻辑。
4. 发送方根据本地事务执行结果向服务端提交二次确认（Commit 或是 Rollback），服务端收到 Commit 状态则将半事务消息标记为可投递，订阅方最终将收到该消息；服务端收到 Rollback 状态则删除半事务消息，订阅方将不会接受该消息。 

### 事务消息回查流程(图中红色流程)：

1. 在断网或者是应用重启的特殊情况下，上述步骤4提交的二次确认最终未到达服务端，经过固定时间后服务端将对该消息发起消息回查。 
2. 发送方收到消息回查后，需要检查对应消息的本地事务执行的最终结果。
3. 发送方根据检查得到的本地事务的最终状态再次提交二次确认，服务端仍按照步骤4对半事务消息进行操作。 