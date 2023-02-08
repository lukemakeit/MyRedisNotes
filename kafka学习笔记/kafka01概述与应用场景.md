### Kafka 概述与应用场景

Kafka传统定义: Kafka是一个**分布式**的基于**发布/订阅模式**的**消息队列(Message Queue)**，主要应用于大数据实时处理领域。

- **发布/订阅: 消息的发布者不会将消息直接发送给特定的订阅者，而不是将发布的消息分为不同的类别，订阅者只接收感兴趣的消息。**

- Kafka最新定义: Kafka是 一个开源的**分布式事件流平台**( Event Streaming Platform)，被数千家公司用于**高性能数据管道、流分析、数据集成和关键任务应用**

- **日志服务器Flume与Hadoop保存能力不匹配;**

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221030075332662.png" alt="image-20221030075332662" style="zoom: 25%;" />

- 消息队列的应用场景

  - **缓冲/削峰**

    <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221030080103423.png" alt="image-20221030080103423" style="zoom: 33%;" />

  - **解耦**

    <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221030080227624.png" alt="image-20221030080227624" style="zoom: 25%;" />

  - **异步通信**

    <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221030080431540.png" alt="image-20221030080431540" style="zoom: 25%;" />

- **消息队列的两种模式**

  <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221030080743268.png" alt="image-20221030080743268" style="zoom: 33%;" />

#### 几个关键字

1. **broker: 服务器 hadoop100、hadoop101、hadoop102**
2. **topic 主题，对数据分类**
3. **parition  分区，将数据切割保存到多个分区上。尽量减少单个partition大小**
4. **副本: 可靠性，副本分为  leader、follower。每个partition都有自己的副本，也都有自己的 leader、follower。**
5. **生产者、消费者: 只针对leader操作**
6. **消费者**
   - 消费者和消费者之间相互独立
   - 消费者组(某个分区只能由一个消费者消费，也就是消费者组的最大成员个数就是 partition数)
7. **zookeeper**
   - 存储了 在线的服务器，如 broker.ids 0 1 2
   - 存储了那个副本是 leader

### Kafka基础架构

1. 为方便扩展，并提高吞吐量，**一个topic分为多个partition**

2. 配合分区的设计，提出消费者组的概念，**组内每个消费者并行消费。注意:一个partition中的数据只能由一个消费者消费，所以一份topic 其对应的消费者组最大有效成员个数是 分区数；**

3. 为提高可用性，为每个partition增加若F副本，类似Hadoop NameNode HA;

4. ZK中记录谁是leader，谁在消费，Kafka2.8.0以后也可以配置不采用ZK;

   消费者只和leader交互，follower是不参与消息消费的。只有在leader挂掉后，follower有机会称为leader;

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230103161705583.png" alt="image-20230103161705583" style="zoom:67%;" />



