---
banner: "[[pixel-banner-image.png]]"
title: MQ消息队列
tags:
  - RabbitMQ
  - SnowflakeId
  - RocketMQ
categories: 编程
date: 2025-02-17T10:02:00
---

# MQ消息队列
## 简介
- 消息队列(Message Queue)是在消息的传输过程中保存消息的容器。
![17536471244591753647124102.png|700x808](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17536471244591753647124102.png)
- **Broker**是mq的核心组件, 主要功能是负责 **接收生产者发送的消息**, **将消息投递给对应的消费者**
- **connection 连接与 channel 信道**:
    - AMQP的连接是长连接，它是一个使用 TCP作为可靠传输的应用层协议, 而信道channel是建立在真实的长TCP连接内的虚拟链接
    - AMQP命令都是通过信道发出去的，不管是发布消息、订阅队列还是接收消息，都是通过信道完成的
    - 类似于进程和线程, 一个服务对应一个connection, 服务中的每一个线程对应一个channel
## 消息传输协议对比
| 协议        | 传输层 | 模式       | 消息持久化  | 可靠性 | 特点                 | 常见实现                                        | 优点                                |
| --------- | --- | -------- | ------ | --- | ------------------ | ------------------------------------------- | --------------------------------- |
| MQTT      | TCP | 发布/订阅    | 支持 QoS | 高   | IoT 数据上报、移动端消息推送   | Mosquitto、EMQX、HiveMQ、阿里云物联网平台              | 轻量级的通讯协议, 即使在网络条件较差的情况下也能保持通信的稳定性 |
| AMQP      | TCP | 发布/订阅/队列 | 支持     | 高   | 支持事务、确认机制、复杂路由     | RabbitMQ、Apache Qpid                        | 可靠性强、功能丰富、支持复杂业务逻辑                |
| Kafka     | TCP | 发布/订阅/日志 | 内建持久化  | 高   | 自定义二进制协议、高吞吐量、顺序消费 | Apache Kafka                                | 高性能、高吞吐、顺序性好、可回溯消息                |
| HTTP      | TCP | 请求-响应短链接 | 不支持    | 中   | 短时请求-响应通信          | 所有 Web 服务器/客户端（Nginx、Tomcat、浏览器等）           | 实现简单、生态完善、无客户端限制                  |
| WebSocket | TCP | 双向长连接    | 不支持    | 中   | 实时通信、低延迟推送         | 浏览器/WebSocket Server、Netty、Spring WebSocket | 实时性强、双向通信、延迟低                     |
## 作用
- **解耦**: 快递员(生产者)把快递放到快递柜里边(Message Queue)去，我们(消费者)从快递柜里边去拿东西，这就是一个异步，如果耦合，那么这个快递员相当于直接把快递交给你，这事固然好，但是万一你不在家，那么快递员就会一直等你，这就浪费了快递员的时间
- **异步**: 用户下单后，系统可以先返回一个下单成功的消息，然后将订单信息放入消息队列中，后台系统再去处理订单信息, 而是相当于**异步线程**处理其他消费者信息
- **削峰**: 将瞬时的高峰流量转化为持续的低流量，从而保护系统不会因为瞬时的高流量而崩溃, 使用mq对消息进行分段保存, **由消费者每次拉取一定数量的数据进行处理(prefetch)**
![17406620441921740662044061.png|700x420](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17406620441921740662044061.png)
## 特点
- **可靠性**。支持持久化，传输确认，发布确认(ack序列码)等保证了MQ的可靠性。
- **灵活的分发消息策略**。
    - RabbitMQ的一大特点。在消息进入MQ前由Exchange(交换机)进行路由消息。分发消息策略有：简单模式、工作队列模式、发布订阅模式、路由模式、通配符模式。
    - RocketMQ使用 主题（Topic）+ 标签（Tag）+ 消费组（Consumer Group）来完成消息分类和消费。轮询发送(默认)
- **支持集群**。多台MQ服务器可以组成一个集群，形成一个逻辑Broker(代理)。
- **多种协议**。MQ支持多种消息队列协议，比如 STOMP、MQTT 等等。
- **支持多种语言客户端**。RabbitMQ几乎支持所有常用编程语言，包括 Java(amqpTemplate客户端)、.NET、Ruby 等等。
- **可视化管理界面**。
    - RabbitMQ提供了一个易用的用户界面程序**Erlang**(端口是15672)
    - RocketMQ的控制台是一个**SpringBoot项目**,需要手动打开; 或者使用打包好的jar包
## 分析 RabbitMQ与RocketMQ
### RabbitMQ(推模式)
**RabbitMQ是采用 Erlang语言实现AMQP协议的消息中间件**
- **虚拟主机**: 每个vhost本质上就是一个mini版的RabbitMQ服务器，拥有自己的队列、交换器、绑定和权限机制
- **交换机**: 一个重要组件,各自有自己的路由键, 负责接收生产者发送的消息，并根据一定的路由规则（如路由键、绑定等）将消息转发到一个或多个队列 ---> 控制消息的分发规则(交换机是消息的中转站)
- **路由键**: 帮助交换机决定该消息应该被转发到哪个队列(规则), 是交换机的关键
- **队列**: 队列是RabbitMQ中的一个存储结构，用于保存消息，直到消费者从队列中获取并处理这些消息。
- **例子**: 假设有一个生产者将消息发送到一个`direct`类型的交换机，该交换机的路由键是`"order.processed"`。这个交换机会将消息转发到一个与该路由键绑定的队列，比如叫做`orderQueue`。消费者只需要监听 `orderQueue`，即可接收到 `"order.processed"` 类型的订单消息进行处理。
![17536391946501753639194226.png|700x209](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17536391946501753639194226.png)
### RocketMQ(拉模式)
- **主题**: Topic
- **路由键**: Tag + 可选 Key, RocketMQ 中的 Tag 是对消息的简单分类，Key 可以做业务标识。没有复杂的路由表达式
- **消费组**: 消费组中包含多个消费者，同一个组内的消费者是竞争消费的关系，每个消费者负责消费组内的一部分消息。默认情况，如果一条消息被消费者 Consumer1 消费了，那同组的其他消费者就不会再收到这条消息。
- **消费位置**: Offset, 为每个消费组在每个队列上维护一个消费位置, 用于为当前消费组保存上次消费到的位置
- **队列**: MessageQueue, RocketMQ 的每个 Topic 会自动被分为多个消息队列，Producer 可指定发往哪个队列（可用于顺序消息）
- **例子**: 创建一个 `Topic`主题：`OrderTopic`, 生产者发送消息到 `OrderTopic`，并设置 `Tag`(路由键) 为 `processed`, 消费者订阅 `OrderTopic`，并使用 `Tag` = "`processed`" 作为过滤条件, Broker 接收并分发消息到对应的 `MessageQueue`（队列片段), 消费者所在的`Consumer Group`拉取消息进行处理
![17475484290661747548428888.png|700x400](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17475484290661747548428888.png)
## RocketMQ集群
- 一个 `Topic` 分布在多个 `Broker`上，一个 `Broker` 可以配置多个 `Topic` ，它们是多对多的关系
### 步骤
- 首先, 我们在Broker上做了主从+集群, 分`Master`和`Slaver`, Slaver会定时从Master备份数据(同步复制: 只有消息同步双写到主从节点上时才返回写入成功, 异步复制)，如果 `master` 宕机，则 `slave` 提供消费服务，但是不能写入消息
- 其次, 我们对`NameServer`也做了集群, 但是不分主从节点, 每隔30s每一个`broker`会向所有的`NameServer`发送ping心跳检测(单个 Broker 和所有 NameServer 保持长连接), 心跳信息包括了Topic配置信息以及路由表信息
- 在生产者需要向`Broker`发送消息的时候，需要先从`NameServer`获取关于 `Broker` 的路由表信息，然后通过轮询的方法去向每个队列中生产数据以达到 负载均衡 的效果。
- 消费者通过NameServer获取路由表消息, 向broker拉取消息消费`Consumer` 可以以两种模式启动 广播和集群, 广播模式下，一条消息会发送给同一个消费组中的所有消费者 ，集群模式下消息只会发送给一个消费者。
![17581623162691758162315810.png|698x506](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17581623162691758162315810.png)
## Kafka集群
### 单节点优化方案
- 单节点Kafka的在高流量突增和存储压力增大时, 如果broker崩溃会联锁导致整个系统崩塌
- 考虑使用主从节点部署
	- 主broker负责写
	- 从broker负责读
	- 从节点定期从主节点同步消息
- 这样就算其中一个broker瘫痪也能使用另外一个broker
### 集群优化方案
- 部署多个broker服务, topic采用分布式方式存储, 一个topic数据被分散到不同broker服务中
- 采用生产者集群和消费者集群, 提高生产消费能力;
- broker集群定期向注册中心集群(ZooKeeper)发送心跳检测以及整个集群状态路由表(如: broker, topic); RocketMQ的注册中心则为NameServer;
- 生产者集群和消费者集群监听注册中心, 确认每次生产和消费的消息是哪一个topic下的哪一个partation, 确保消费者拉取的同一性;
![17580141768161758014176045.png](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17580141768161758014176045.png)
## 死信队列
- 存储那些无法被正常消费的消息,通常由交换机和路由键指定到特殊的队列(即死信队列)
- **消息过期**：如果消息设置了`expiration`属性，当消息在队列中存放超过指定时间后会被丢弃，并进入死信队列。
- **队列溢出**：当队列已满且无法再接收新的消息时，新进入的消息将会被丢弃并进入死信队列。
- **消息被拒绝**：当消费者拒绝处理某条消息时，可以选择将消息标记为“死信”。这种情况下，消息会进入死信队列。
- **消费者未确认**：如果消息被消费者消费后未发送确认（ack），并且RabbitMQ设置了`requeue`为`false`，则消息会被丢弃并进入死信队列。
要为队列或交换机添加死信参数（Dead Letter Exchange）配置，应该关注以下几个部分：
1. **Dead letter exchange**：这里可以填写死信交换机的名称。当队列中的消息因某种原因（如超时、被拒绝等）不能被消费时，它们将被转发到这个死信交换机。你需要在此输入目标死信交换机的名称。
2. **Dead letter routing key**：这是可选的参数，用于指定死信队列的路由键。消息会被转发到死信交换机，并根据路由键路由到相应的队列。你可以填写路由键，或者留空使用默认设置。
    
在配置过程中，如果你为队列设置了 **Dead letter exchange**，确保死信交换机已经存在，并且设置正确。你还可以根据需求配置其他参数，如 **Message TTL**（消息的过期时间）和 **Auto expire**（队列的过期时间）。
### 死信参数添加步骤：

1. 在 RabbitMQ 管理界面创建或编辑队列或交换机。
2. 在 `Arguments` 部分找到并配置：
    - **Dead letter exchange**：填写死信交换机的名称。
    - **Dead letter routing key**：可选，填写死信路由键的名称。
3. 完成设置后，保存队列或交换机配置。
### 其他相关参数：
- **Auto expire**：队列自动过期的时间。
- **Message TTL**：设置消息的过期时间，消息在 TTL 到期后会被丢弃或进入死信队列。
- **Overflow behaviour**：设置当队列达到最大长度时的行为。
- **Single active consumer**：如果设置为 `true`，该队列只允许一个消费者进行消费。
- **Max length / Max length bytes**：设置队列的最大长度或最大字节数，超出后会触发 `Overflow behaviour`。
## 分发消息策略
### 1. 简单队列模式 (Direct Mode)
- 特点：生产者生产消息, 由多个消费者去竞争消费, 每个消息只消费一次
- 使用场景：**适用于简单的点对点通信，生产者与消费者直接交互**。
- 实现方式：生产者将消息发送到某个特定的队列，消费者从该队列中接收消息。
- 特点：没有复杂的路由逻辑，简单高效，但不适合多消费者的场景。
### 2. 工作队列模式 (Work Queue)
- 特点：**生产者将消息发送到一个共享的队列(一对多)，多个消费者从该队列中获取消息进行并行处理。**
- 使用场景：适用于**负载均衡**，多个消费者共同处理任务，避免某个消费者负担过重。
- 实现方式：所有消费者都监听同一个队列，RabbitMQ会以公平的方式将消息分发给消费者（即一个消费者处理一条消息，直到处理完才会继续接收新的消息）。
- 特点：消费者可以并行处理任务，确保负载均衡和异步处理，适用于后台任务处理、分布式任务调度等。
### 3. 发布订阅模式 (Publish/Subscribe)
- 特点：**生产者将消息发布到一个交换机(主题)，然后多个消费者订阅该交换机(主题)，消费者将接收来自交换机(主题)的所有消息, 也就是一个消息可以多次消费。**
- 使用场景：**适用于广播消息场景，例如日志收集、事件通知等**。
- 实现方式：生产者将消息发布到一个广播类型的交换机（如`fanout`交换机），交换机会将消息广播给所有绑定在其上的队列，所有消费者都会收到该消息。
- 特点：通过交换机将消息发送到多个队列，实现多消费者接收消息，适用于需要广播的场景。
### 4. 路由模式 (Routing)
- 特点：生产者将消息发送到交换机，并通过指定的路由键来决定消息的去向。消费者根据路由键来决定是否接收消息。
- 使用场景：适用于根据消息的某些属性进行路由，灵活性较高，能够支持更多的消息传递场景。
- 实现方式：使用`direct`交换机，生产者将消息发送到交换机，并指定路由键，交换机会根据路由键将消息转发给与该路由键匹配的队列。消费者通过监听特定的队列来接收消息。
- 特点：灵活的消息路由机制，生产者与消费者通过路由键关联，可以将消息发送到特定的队列
## 全局唯一Id生成策略
### Snowflake(64bit位的long类型)
- 解决的问题: 分布式场景下主键ID重复性问题
![17603292854441760329284549.png|700x214](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17603292854441760329284549.png)
![17536471744521753647173892.png|700x396](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17536471744521753647173892.png)

```java
// 反解析 雪花id 右移22位机器位 加上元年时间戳得到当前时间
public class SnowflakeIdParser {

    // Hutool 默认起始时间戳
    private static final long EPOCH = 1577808000000L;

    public static void main(String[] args) {
        long id = 1950232867263090688L;

        // 反解析时间戳（去掉序列号和机器ID部分，即右移22位）
        long timePart = (id >> 22);

        // 计算实际生成时间
        long generateTime = timePart + EPOCH;

        Date date = new Date(generateTime);
        System.out.println("生成时间: " + date);

        // 反解析机器ID和序列号（根据Hutool Snowflake结构）
        // Hutool Snowflake结构：
        // 41位时间戳 | 10位工作机器ID | 12位序列号

        long sequence = id & ((1 << 12) - 1); // 最后12位
        long workerId = (id >> 12) & ((1 << 10) - 1); // 中间10位

        System.out.println("工作机器ID: " + workerId);
        System.out.println("序列号: " + sequence);
    }
}
```
### 百度的UidGenerate
- 本质上也是基于雪花算法的64bit的id, 不同的是修改了64bit的分配比例
- 时间戳可使用年数从原本的约69年下降至6年, 但是可分配的机器bit和序列毫秒bit增多, **解决了在高并发、大规模机器部署场景下，传统雪花算法机器位不够用的问题**
![17603299984431760329997773.png](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17603299984431760329997773.png)
### 时钟回拨问题
- 出现原因
	1. 修改了系统时间
	2. 有时候不同的机器上需要同步时间，可能不同机器之间存在误差，那么可能会出现时间回拨问题
- 解决方案
	1. 回拨时间小的时候，不生成 ID，**阻塞等待**到时间点到达。
	2. 上面的方案只适合时钟回拨较小的，如果间隔过大，阻塞等待，肯定是不可取的，因此要么**超过一定大小的回拨直接报错，拒绝服务**
	3. 使用中间服务器来定时同步所有服务器的时钟
## 总结
### MQ的存储容量上限
- RabbitMQ如果消息数量超过 `x-max-length`，RabbitMQ 会**移除最旧的消息**，以便插入新消息。
	- map.put("x-max-length", 1000);          // 最多 1000 条消息
- RabbitMQ如果总大小超过 `x-max-length-bytes`，RabbitMQ 也会**删除旧消息**直到满足条件。
	- map.put("x-max-length-bytes", 1000000); // 总字节数不超过1MB(约 1,000,000 字节)
- RabbitMQ所以只能存储 1,000,000 bytes ÷ 16,000 bytes ≈ 62.5 条
- RocketMQ如果消息大小超过4K, 会将消息压缩发送
### 如何保证消息可靠性?
| 阶段           | 机制           | 保证手段           |
| ------------ | ------------ | -------------- |
| 生产端的可靠投递     | 同步发送、发送确认、重试 | 保证消息能到达 Broker |
| Broker服务端    | 磁盘持久化、主从复制   | 保证消息不丢失、可恢复    |
| 消费端的确认消费 ACK | 消费确认、消息重试机制  | 保证消息不会漏处理或重复处理 |

### 如何保证消息的幂等性(消息被重复消费)?
- 消息必须携带唯一标识来保证幂等性
- 通过使用雪花算法生成唯一消息id
- 在消费时通过Redis判断是否存在以消息id为key的键值对, 存在就证明被消费过, 不再消费
- 使用 **数据库插入法** ，基于数据库的唯一键来保证重复数据不会被插入多条。
- 这些实现幂等的方法，也同样适用于，**在其他场景中来解决重复请求或者重复调用的问题** 。比如将 HTTP 服务设计成幂等的，**解决前端或者 APP 重复提交表单数据的问题** ，也可以将一个微服务设计成幂等的，解决 `RPC` 框架自动重试导致的 **重复调用问题** 。
### RocketMQ如何实现延迟消息:
- 将消息发送到临时Topic中的队列, 定时任务轮询, 然后把这些队列投递到对应的正确Topic中队列进行消费
### 如何处理消息积压?
- 溯源
	- 我们可以先检查 **是否是消费者出现了大量的消费错误** ，或者打印一下日志查看是否是哪一个线程卡死，出现了锁资源不释放等等的问题
- Broker端(队列端)
	- 部署Broker集群, 增加存储水平
	- 新建一些临时Broker用来存放消息
- 消费端
	- 扩容消费者, 增加消费能力, 通过也应该扩容同一主题下的队列数量, 因为自始至终一个队列只能被一个消费者消费, 消费者多了, 而队列没多, 也会导致消费者被搁置
	- 优化消费逻辑: 消息积攒一批，一次性查询或写入数据库, 减少数据库IO
- 逻辑端
	- 修改**prefetch count**, RabbitMQ官方给出的建议是prefetch count一般设置在100~300之间
		- prefetch count过大导致内存溢出问题
		- prefetch count过小导致吞吐量过低
	- 使用 延时队列TTL + 死信队列
### 如何实现延迟消息?
- Broker受到消息后先发送到特定的延时队列中, 开启一个定时任务轮询检测是否到时间, 若是, 再将消息投递到对应的供消费者消费的队列中
### 发送方确认机制
- 配置
```yaml
spring:  
  rabbitmq:  
    template:  
      retry:  
        enabled: true             # 开启发送失败时的重试机制  
        initial-interval: 10s     # 初始重试间隔为10秒  
        multiplier: 1             # 每次重试间隔不增长（固定间隔）  
        max-attempts: 3           # 最多重试3次  
    publisher-confirm-type: correlated     # 开启发送确认  
    publisher-returns: true      # 开启Return 回调
```
- 发送消息时带上 CorrelationData: 可用于回调中追踪消息
```java
@Autowired  
private RabbitTemplate rabbitTemplate;  
​  
public void sendMessage() {  
    String message = "Hello RabbitMQ";  
    CorrelationData correlationData = new CorrelationData(UUID.randomUUID().toString());  
​  
    rabbitTemplate.convertAndSend(  
        "orderExchange",        // 交换机名  
        "order.processed",      // 路由键  
        message,  
        correlationData         // 可用于回调中追踪消息   
    );  
}
```
- 配置回调
```java
@Configuration  
public class RabbitConfig {  
​  
    @Bean  
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {  
        RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);  
​  
        // 当生产者将消息发送到交换机后，会触发这个回调  
        rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> {  
            if (ack) {  
                System.out.println("发送成功！CorrelationData: " + correlationData);  
            } else {  
                System.err.println("发送失败！Cause: " + cause);  
            }  
        });  
​  
        // 确认队列是否收到  
        rabbitTemplate.setReturnsCallback(returned -> {  
            System.err.println("消息未路由到队列: " + returned.getMessage());  
        });  
​  
        return rabbitTemplate;  
    }  
}
```
### 消费者消息确认和重试机制
- 配置
```yaml
spring:  
  rabbitmq:  
    listener:  
      simple:  
        acknowledge-mode: manual   # 消息手动确认（业务处理成功后显式 ack）  
        prefetch: 1                # 控制每次消费的条数  
        retry:  
          enabled: true           # 启用消费失败后的重试机制  
          max-attempts: 3     # 最大重试次数  
          initial-interval: 1000ms  
          multiplier: 2       # 间隔时间增长倍数  
          max-interval: 10000ms  
    connection-timeout: 10s       # 连接超时设置
```

| 方法                            | 含义          | 用法                             |
| ----------------------------- | ----------- | ------------------------------ |
| `basicAck(tag, false)`        | 手动确认        | 正常消费成功后调用                      |
| `basicNack(tag, false, true)` | 拒绝 + 是否重回队列 | 消息失败处理时使用                      |
| `basicReject(tag, false)`     | 拒绝，不支持批量    | 等价于 `basicNack(requeue=false)` |

- 消费者确认
```java
@Component  
public class OrderConsumer {  
​  
    @RabbitListener(queues = "orderQueue")  
    public void receiveMessage(MsgBody msgBody, Message message, Channel channel) {  
        try {  
            System.out.println("✅ 收到消息：" + msgBody);  
​  
            // 处理业务逻辑…  
​  
            // 手动确认消息（消息处理成功）: deliveryTag:RabbitMQ自动分配,用于标识消费者Channel中的消息  
            channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);  
​  
        } catch (Exception e) {  
            System.err.println("消息处理失败：" + e.getMessage());  
​  
            // 拒绝消息，并决定是否重回队列  
            try {  
                // 是否拒绝多个消息 是否重新入队  
                channel.basicNack(message.getMessageProperties().getDeliveryTag(), false, false);   
            } catch (IOException ioException) {  
                ioException.printStackTrace();  
            }  
        }  
    }  
}
```
### 为什么MQ不直接使用Http协议呢？
- 因为Http请求和响应报文是比较复杂的，包含了cookie、数据的加密解密、状态码、晌应码等附加的功能，但是对于个消息而言，我们并不需要这么复杂，也没有这个必要性，它其实就是负责数据传递，存储，分发就够，要追求的是高性能。尽量简洁，快速。
- 大部分情况下Http大部分都是短链接，在实际的交互过程中，一个请求到响应很有可能会中断，中断以后就不会就行持久化，就会造成请求的丢失。这样就不利于消息中间件的业务场景，因为消息中间件可能是一个长期的获取消息的过程，所以必须保持长链接, 目的是为了保证消息和数据的高可靠和稳健的运行
### 小芝士
- 一般来说要控制 主题中的队列数量与消费者一致
- 一个队列在同一时刻只能被消费者组内的一个消费者消费
- 每一个消费者组都要维护自己的消费位置(offSet指针), 如果不单独维护，每个消费组就没办法知道自己消费到哪里了，会互相干扰
- RabbitMQ和ActiveMQ不像Kafka和RocketMQ一样支持主从以及分布式集群, 而是只支持主从
- 如果业务场景对并发量(吞吐量)要求不是太高, 又因为RabbitMQ延迟在微秒级别, 时效性最高，RabbitMQ 一定是你的首选
- Pull模式的另外一个好处是consumer可以自主决定是否批量的从broker拉取数据
- 如果为了避免consumer崩溃而采用较低的推送速率，将可能导致一次只推送较少的消息而造成浪费。Pull模式下，consumer就可以根据自己的消费能力去决定这些策略。
