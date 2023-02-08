## Kafka消费者

#### kafka 消费方式

- pull (拉)模式

  **<font style="color:red">consumer采用从broker中主动拉取数据，Kafka采用这种方式</font>**

  **pull模式不足之处是，如果Kafka没有数据，消费者可能会陷入循环中，一直返回空数据**

- **push(推)模式**

  **Kafka没有采用这种方式，因为broker决定消息发送速率，很难适应所有消费者的消费速率**

  例如推送的速度是50m/s。Consumer1、Consumer2就来不及处理消息。

#### 消费者整体工作流程

![image-20230110064456442](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230110064456442.png)

#### 消费者组

Consumer Group (CG) :消费者组，由多个consumer组成。形成一个消费者组的条件，是所有消费者的groupid相同;
- **消费者组内每个消费者负责消费不同分区的数据，一个分区只能由一个组内消费者消费**;
- **消费者组之间互不影响**。所有的消费者都属于某个消费者组，即消费者组是逻辑上的一个订阅者;
- **如果向消费组中添加更多的消费者，超过主题分区数量，则有一部分消费者就会闲置，不会接收任何消息**
- **消费者组之间互不影响。**所有的消费者都属于某个消费者组，即**消费者组是逻辑上的一个订阅者**

#### 消费者组初始化流程

**coordinator: 辅助实现消费者组的初始化和分区的分配**

coordinator节点选择= `groupid的hashcode值% 50` (50是`__consumer__offsets`的分区数量)
例如: **groupid的hashcode值 = 1, 1%50= 1,那么`__consumer_offsets`主题的1号分区，1号分区在哪个broker上， 就选择这个节点的coordinator作为这个消费者组的老大。消费者组下的所有的消费者提交offset的时候就往这个分区去提交offset**

**再平衡:如果某个consumer挂了，触发再平衡，比如总共是partition 0 1 2 3 4  5 6。Consumer0负责partition0 1 2，Consumer1负责partition 3 4，Consumer负责partition 5 6，Consumer 0挂了，再平衡时候就是 Consumer1 负责partition 0 1 2 3 ，Consumer2负责partition 4 5 6**

![image-20230110070025815](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230110070025815.png)

#### 消费者组详细消费流程

![image-20230110070347512](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230110070347512.png)

#### 独立消费者案例(订阅主题)

**注意: 在消费者API代码中必须配置消费者组id, 命令行启动消费者不填写消费者组id会自动填写随机的消费者组id**

示例:

```java
package com.luke.kafka.producer;

import java.time.Duration;
import java.util.ArrayList;
import java.util.Properties;

import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.serialization.StringDeserializer;

public class CustomConsumer {
    public static void main(String[] args) {
        // 1. 配置
        Properties proper = new Properties();

        // 连接
        proper.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "hadoop101:9092,hadoop102:9092");

        // 反序列化
        proper.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        proper.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());

        // 配置消费者组id
        proper.put(ConsumerConfig.GROUP_ID_CONFIG, "test");

        // 2. 创建一个消费者
        KafkaConsumer<String, String> kafkaConsumer = new KafkaConsumer<String, String>(proper);

        // 3. 订阅主题 first
        ArrayList<String> topics = new ArrayList<String>();
        topics.add("first");
        kafkaConsumer.subscribe(topics);

        // 4. 消费数据
        while (true) {
            // 疫苗拉取一次
            ConsumerRecords<String, String> records = kafkaConsumer.poll(Duration.ofSeconds(1));
            for (ConsumerRecord<String, String> record : records) {
                System.out.println(record);
            }
        }
    }
}

// 输出结果:
ConsumerRecord(topic = first, partition = 1, leaderEpoch = 0, offset = 23, CreateTime = 1673392485391, serialized key size = -1, serialized value size = 12, headers = RecordHeaders(headers = [], isReadOnly = false), key = null, value = hello kafka0)
ConsumerRecord(topic = first, partition = 1, leaderEpoch = 0, offset = 24, CreateTime = 1673392485440, serialized key size = -1, serialized value size = 12, headers = RecordHeaders(headers = [], isReadOnly = false), key = null, value = hello kafka1)
ConsumerRecord(topic = first, partition = 1, leaderEpoch = 0, offset = 25, CreateTime = 1673392485445, serialized key size = -1, serialized value size = 12, headers = RecordHeaders(headers = [], isReadOnly = false), key = null, value = hello kafka2)
ConsumerRecord(topic = first, partition = 1, leaderEpoch = 0, offset = 26, CreateTime = 1673392485450, serialized key size = -1, serialized value size = 12, headers = RecordHeaders(headers = [], isReadOnly = false), key = null, value = hello kafka3)
ConsumerRecord(topic = first, partition = 1, leaderEpoch = 0, offset = 27, CreateTime = 1673392485453, serialized key size = -1, serialized value size = 12, headers = RecordHeaders(headers = [], isReadOnly = false), key = null, value = hello kafka4)
```

#### 订阅分区

```java
        // 3. 订阅主题对应的分区
        ArrayList<TopicPartition> topicPartitions = new ArrayList<TopicPartition>();
        topicPartitions.add(new TopicPartition("first", 0));
        kafkaConsumer.assign(topicPartitions);

        // 4. 消费数据
        while (true) {
            // 一秒拉取一次
            ConsumerRecords<String, String> records = kafkaConsumer.poll(Duration.ofSeconds(1));
            for (ConsumerRecord<String, String> record : records) {
                System.out.println(record);
            }
        }
```

#### 消费者组案例

```java
// 可以搞多个消费者，保证groupId相同即可
        // 配置消费者组id
        proper.put(ConsumerConfig.GROUP_ID_CONFIG, "test");
```

1. 一个consumer group中有多个consumer组成，一 个topic有 多个partition组成， 现在的问题是，到底由哪个consumer来消费哪个partition的数据。
2. Kafka有四种主流的分区分配策略: **Range、RoundRobin、Sticky、CooperativeSticky**
    可以通过配置参数`partition.assignment.strategy`,修改分区的分配策略。默认策略是`Range + CooperativeSticky`。 Kafka可以同时使用多个分区分配策略;

获取值的地址: [https://kafka.apache.org/documentation/#consumerconfigs_partition.assignment.strategy](https://kafka.apache.org/documentation/#consumerconfigs_partition.assignment.strategy)

- **分区分配策略之Range**

  Range是对每个topic而言的。

  首先对同一个topic里面的**分区按照序号进行排序**,并对**消费者按照字母顺序进行排序**。

  假如现在有7个分区，3个消费者，排序后的分区将会是`0,1,2,3,4,5,6`;消费者排序完之后将会是`C0,C1,C2`。

  通过**partitions数/consumer数**来决定每个消费者应该消费几个分区。如果除不尽，那么前面几个消费者将会多消费1个分区。

  例如，7/3=2余1，除不尽，那么消费者C0便会多消费1个分区。8/3=2余2，除不尽，那么C0和C1分别多消费一个。

  注意:如果只是针对1个topic而言，C0消费者多消费1个分区影响不是很大。**但是如果有N多个topic,那么针对每个topic, 消费者C0都将多消费1个分区，topic越多， C0消费的分区会比其他消费者明显多消费N个分区**

  **容易产生数据倾斜!**

  ```java
  proper.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,"org.apache.kafka.clients.consumer.RangeAssignor");
  ```

- **分区分配策略之RoundRobin**

  RoundRobin针对集群中<font style="color:red">所有Topic而言</font>

  RoundRobin轮询分区策略，是把<font style="color:red">所有的partition和所有的consumer</font>都列出来,然后<font style="color:red">按照hashcode进行排序</font>,最后通过<font style="color:red">轮询算法</font>来分配partition给到各个消费者。

  <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230111224630462.png" alt="image-20230111224630462" style="zoom:50%;" />

```java
proper.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,"org.apache.kafka.clients.consumer.RoundRobinAssignor");
```

- **分区分配策略之Sticky**

  **粘性分区定义**:可以理解为分配的结果带有“粘性的”。即在执行一次新的分配之前,考虑上一次分配的结果，尽量少的调整分配的变动，可以节省大量的开销。

  粘性分区是Kafka 从0.11.x 版本开始引入这种分配策略，首先会**尽量均衡的放置分区到消费者上面**，在出现同一消费者组内消费者出现问题的时候，会**尽量保持原有分配的分区不变化**。

### Offset

####  Offset保存位置

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230111231726795.png" alt="image-20230111231726795" style="zoom:50%;" />

`_consumer_ offsets` 主题里面采用key 和value的方式存储数据。**key 是`group.id+ topic+分区号`，value 就是当前offset 的值。每隔一段时间，kafka 内部会对这个topic 进行compact，也就是每个`group.id+topic+分区号`就保留最新数据**

**消费offset案例**

a. 思想: `__consumer_offsets`为Kafka中的topic,那就可以通过消费者进行消费;

b. 在配置文件`config/consumer.properties`中添加**配置`exclude.internal.topics=false`,默认是true, 表示不能消费系统主题。为了查看该系统主题数据，所以该参数修改为false**

c. 采用命令行方式，创建-一个新的topic
```shell
$ bin/kafka-topics.sh --bootstrap-server hadoop101:9092 --create  --topic atguigu --partitions 2 --replication-factor 2
```

d. 启动生产者往atguigu生产数据
```shell
$ bin/kafka-console-producer.sh --topic atguigu --bootstrap-server hadoop101:9092 
```

e. 启动消费者消费atguigu数据
```shell
$ bin/kafka-console-consumer.sh --bootstrap-server hadoop101:9092 --topic atguigu --group test
```
**注意:指定消费者组名称，更好观察数据存储位置(key 是group.id+topic+分区号)**

f. 查看消费者消费主题 `__consumer_offsets`
```shell
$ bin/kafka-console-consumer.sh --topic __consumer_offsets --bootstrap-server hadoop101:9092 --consumer.config config/consumer.properties --formatter  "kafka.coordinator.group.GroupMetadataManager\$OffsetsMessageFormatter" --from-beginning
```

#### 自动提交offset

为了使我们能够专注于自己的业务逻辑,Kafka提供了自动提交offset的功能。

自动提交offset的相关参数:
- <font style="color:red">`enable.auto.commit`</font>:是否开启自动提交offset功能，默认是true
- <font style="color:red">`auto.commit.interval.ms`</font>:自动提交offset的时间间隔，默认是5s

```java
        // 自动提交
        proper.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, true);
        // 提交周期
        // 链接: https://kafka.apache.org/documentation/#consumerconfigs_auto.commit.interval.ms
        proper.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, 1000);
```

##### 手动提交offset

虽然自动提交offset十分简单便利，但由于其是基于时间提交的，开发人员难以把握offset提交的时机。因此Kafka还提供了手动提交offset的API。

手动提交offset的方法有两种:分别是<font style="color:red">commitSync (同步提交)</font>和<font style="color:red">commitAsync ( 异步提交)</font>。两者的相同点是，都会将<font style="color:red">本次提交的一批数据最高的偏移量提交</font>;不同点是，<font style="color:red">同步提交阻塞当前线程</font>，一直到提交成功，并且会自动失败重试(由不可控因素导致，也会出现提交失败) ;而<font style="color:red">异步提交则没有失败重试机制，故有可能提交失败</font>

```java
        // 手动提交
        proper.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        // 手动提交offset
        kafkaConsumer.commitAsync();
```

#### 指定offset消费

<font style="color:red">`auto.offset.reset=earliest|latest|none`，默认是latest</font>

当Kafka 中没有初始偏移量(消费者组第一次消费)或服务器上不再存在当前偏移量时(例如该数据已被删除)，该怎么办?
-  **earliest**: 自动将偏移量重置为最早的偏移量，`--fromn-beginning`
- **latest(默认值)**:自动将偏移量重置为最新偏移量;
- **none**:如果未找到消费者组的先前偏移量，则向消费者抛出异常;

<font style="color:red">**这里没看太懂??** </font>

下面是从指定offset位置开始消费示例:

```java
        // 订阅topic
        ArrayList<String> topics = new ArrayList<String>();
        topics.add("first");
        kafkaConsumer.subscribe(topics);
        // 指定位置进行消费
        Set<TopicPartition> assignment = kafkaConsumer.assignment();

        // 保证coordinator 和 consumer leader的分区分配方案已经制定完毕
        // 一直等于0,代表 当前这个consumer 没有需要去消费的分区，或者说 分区以及被同组其他consumer分配了
        // 抑或说, 比如上一次进程启动,组是 test 将数据消费完了, 再次启动,组还是test,那肯定拉取不到数据的。除非有新来的数据。所以需要等待
				// 有评论说: 获取自动分配的分区计划集合没获取到,不是因为时间,是因为没有拉取就不会获得? 所以获取前需要可以拉取一次
				// 还有评论说: 消费者的消费流程 需要 Consumer Leader、coordinator等选出和参与, 有 45秒时间需要考虑?
        while (assignment.size() == 0) {
            // 拉取数据
            kafkaConsumer.poll(Duration.ofSeconds(1));
            // 重新获取 topic partition
            assignment = kafkaConsumer.assignment();
        }
        for (TopicPartition tp : assignment) {
            // 每个分区都进行offset指定
            kafkaConsumer.seek(tp, 100);
        }
```

#### 指定时间消费

需求:在生产环境中，会遇到最近消费的几个小时数据异常，想重新按照时间消费。例如要求按照时间消费前一天的数据，怎么处理?

```java
        // 保证coordinator 和 consumer leader的分区分配方案已经制定完毕
        while (assignment.size() == 0) {
            // 拉取数据
            kafkaConsumer.poll(Duration.ofSeconds(1));
            // 重新获取 topic partition
            assignment = kafkaConsumer.assignment();
        }

        // 希望把时间转换为对应的offset
        HashMap<TopicPartition, Long> topicPartitionLong = new HashMap<TopicPartition, Long>();
        // 封装对应集合
        for (TopicPartition tp : assignment) {
            topicPartitionLong.put(tp, System.currentTimeMillis() - 1 * 24 * 3600 * 1000);
        }

        // 通过时间获取对应的offset
        Map<TopicPartition, OffsetAndTimestamp> offsetsForTimes = kafkaConsumer.offsetsForTimes(topicPartitionLong);

        // 指定消费的offset
        for (TopicPartition tp : assignment) {
            OffsetAndTimestamp offsetAndTimestamp = offsetsForTimes.get(tp);
            kafkaConsumer.seek(tp, offsetAndTimestamp.offset());
        }

        // 4. 消费数据
        while (true) {
            // 一秒拉取一次
            ConsumerRecords<String, String> records = kafkaConsumer.poll(Duration.ofSeconds(1));
            for (ConsumerRecord<String, String> record : records) {
                System.out.println(record);
            }
        }
```

#### 漏消费和重复消费

- **重复消费:**已经消费了数据，但是offset没提交
- **漏消费:** 先提交offset后消费, 有可能造成数据的漏消费

场景1: **重复消费**，自动提交offset引起

![image-20230114155454058](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230114155454058.png)

场景2: **漏消费**。设置offset为手动提交，当offset被提交时，consumer得到的数据还在内存中未落盘，此时刚好消费者线程被kill掉，那么offset已经提交，但是数据未处理，导致这部分内存中的数据丢失

![image-20230114160454217](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230114160454217.png)

**如何解决呢? 怎么做到不漏消费也不重复消费呢? 用事务**

#### 消费者事务

如果想完成Consumer端的精准一次性消费，那么需要<font style="color:red">Kafka消费端将消费过程和提交offset过程做原子绑定</font>。此时我们需要将Kafka的offset保存到支持事务的自定义介质(比如MySQL)。这部分知识会在后续项目部分涉及。

![image-20230114162739421](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230114162739421.png)

#### 数据积压了，怎么办(消费者如何提高吞吐量)

- 如果是Kafka消费能力不足，则可以考虑**增加Topic的分区数**，并且同时提升消费组的消费者数量，**消费者数=分区数**。(两者缺一不可)

- 如果是下游的数据处理不及时:**提高每批次拉取的数量**。批次拉取数据过少(拉取数据/处理时间<生产速度) ,使处理的数据小于生产的数据，也会造成数据积压。

  从一次最多拉取500条，调整为一次最多拉取1000条。

