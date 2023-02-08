### 生产者原理

- **batch.size:只有数据基类到batch.size之后，sender才会发送数据。默认16k**
- **linger.ms: 如果数据迟迟未达到batch.size,sender等待linger.ms设置的时间到了之后就会发送数据。单位ms，默认是0ms，表示没有延迟。**

#### 生产者数据发送流程

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230103175138136.png" alt="image-20230103175138136" style="zoom:67%;" />

- **DQuene 是分区器的队列，每个分区都有一个队列。到达batch.size、linger.ms条件后，就由sender线程朝kafka集群发;**
- **sender中也是会缓存请求的，默认最多缓存 5个请求。比如request1 发送成功后那么sender本地就少缓存一个，否则最多缓存5个失败的，就不断重试了;**

简单示例:

```java
package com.luke.kafka.producer;

import java.util.Properties;

import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.common.serialization.StringSerializer;

public class CustomProducer {
    public static void main(String[] args) {
        // 配置
        Properties proper = new Properties();

        proper.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "hadoop101:9092,hadoop102:9092");
        proper.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        proper.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        // 1. 创建kafka生产者对象
        KafkaProducer<String, String> producer = new KafkaProducer<String, String>(proper);
        // 2. 发送数据
        producer.send(new ProducerRecord<String, String>("first", "hello wolrd"));

        // 3. 关闭资源
        producer.close();
    }
}
```

**异步发送:**

```java
        // 2. 发送数据
        producer.send(new ProducerRecord<String, String>("first", "hello wolrd"), new Callback() {
            public void onCompletion(RecordMetadata metadata, Exception exception) {
                if (exception == null) {
                    System.out.println("主题:" + metadata.topic() + "  分区:" + metadata.partition());
                }
            }
        });

// 输出:
主题:first  分区:1
```

**同步发送:**

```java
throws InterruptedException, ExecutionException

producer.send(new ProducerRecord<String, String>("first", "hello kafka")).get();
```

#### 生产者分区

Kafka分区器的好处:

- **便于合理使用存储资源**，每个partition在一个Broker上存储，可以把海量的数据按照分区切割成一块一块的数据存储在多台Broker上。合理控制分区的任务，可以实现**负载均衡**的效果
- **提高并行度,** 生产者可以分区为单位发送数据；消费者可以以分区为单位进行**数据消费**

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230105204120565.png" alt="image-20230105204120565" style="zoom: 33%;" />

- **默认分区策略:**

  - 如果用户指定了partition,则使用它

  - 如果没有指定partition 但是指定了key，则根据key的hash值  和 分区数确定一个分区

  - 如果没有指定partition、也没有指定key，则使用黏性分区(sticky partition)，一批一批的存储

![image-20230105204843711](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230105204843711.png)

示例:

```java
        // 2. 发送数据,此时partition=0,key为空
        producer.send(new ProducerRecord<String, String>("first", 0, "", "hello wolrd"), new Callback() {
            public void onCompletion(RecordMetadata metadata, Exception exception) {
                if (exception == null) {
                    System.out.println("主题:" + metadata.topic() + "  分区:" + metadata.partition());
                }
            }
        });
```

#### 自定义分区器

```java
package com.luke.kafka.producer;

import java.util.Map;

import org.apache.kafka.clients.producer.Partitioner;
import org.apache.kafka.common.Cluster;

public class MyPartitioner implements Partitioner {

    public void configure(Map<String, ?> arg0) {

    }

    public void close() {
    }

    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
        // 获取数据 hello、world等
        String msgVal = value.toString();
        int partitionNum;

        if (msgVal.contains("atguigu")) {
            partitionNum = 0;
        } else {
            partitionNum = 1;
        }
        return partitionNum;
    }

}
```

```java
        // 关联自定义分区器
        proper.put(ProducerConfig.PARTITIONER_CLASS_CONFIG, MyPartitioner.class.getName());
        // 1. 创建kafka生产者对象
        KafkaProducer<String, String> producer = new KafkaProducer<String, String>(proper);
        producer.send(new ProducerRecord<String, String>("first", "hello", "hello wolrd"), new Callback() {
            public void onCompletion(RecordMetadata metadata, Exception exception) {
                if (exception == null) {
                    System.out.println("topic:" + metadata.topic() + "  分区:" + metadata.partition());
                }
            }
        });
        producer.send(new ProducerRecord<String, String>("events", "atguigu", "hello atguigu"), new Callback() {
            public void onCompletion(RecordMetadata metadata, Exception exception) {
                if (exception == null) {
                    System.out.println("topic:" + metadata.topic() + "  分区:" + metadata.partition());
                }
            }
        });
//输出:
topic:first  分区:1
topic:events  分区:0
```

#### 生产者如何提高吞吐量

- `batch.size`: 批次大小，默认16k
- `linger.ms`:等待时间，修改为5-100ms
- `compression.type`: 压缩snappy
- `RecordAccumulator`: 缓冲区大小，修改为64M

```java
        // 缓冲区大小 32m
        proper.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 33554432);

        // 批次大小
        proper.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);

        // linger.ms
        proper.put(ProducerConfig.LINGER_MS_CONFIG, 1);

        // 压缩
        proper.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");
```

#### 数据的可靠性

数据发送完成后，应答acks:

- **0:** 生产者发送过来的数据，不需要等待数据落地应答;
- **1:** 生产者发送过来的数据，Leader收到数据后应答;
- **-1 (all)**: 生产者发送过来的数据，Leader+ 和 Isr队列里面所有节点收齐数据后应答。-1和all等价

![image-20230105214414699](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230105214414699.png)





![image-20230105214726953](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230105214726953.png)

![image-20230105220421731](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230105220421731.png)

配置:

```java
        // acks
        proper.put(ProducerConfig.ACKS_CONFIG, "-1");

        // 重试次数,默认是INT最大值 也就是一直重试
        proper.put(ProducerConfig.RETRIES_CONFIG, 3);
```

#### 数据去重

- 至少一次(At Least Once) = **ACK级别设置为-1 +分区副本大于等于2 + ISR里应答的最小副本数量大于等于2**
- 最多一次(At Most Once) = **ACK级别设置为0**

- 总结:
  At Least Once可以保证数据不丢失，但是不能保证数据不重复;

  AtMostOnce可以保证数据不重复，但是不能保证数据不丢失;

- 精确一次(ExactlyOnce):对于一些非常重要的信息，比如和钱相关的数据，要求数据既不能重复也不丢失

**Kafka 0.11版本以后，引入了一项重大特性:幂等性和事务**

#### 幂等性原理

幂等性就是指Producer不论向Broker发送多少次重复数据，Broker端 都只会持久化一条，保证了不重复。

**精确一次(Exactly Once) = 幂等性+至少一次( ack=-1+分区副本数>=2 + ISR最小副本数量>=2)**

重复数据的判断标准:<font style="color:red">具有`<PID, Partition, SeqNumber>`相同主键的消息提交时，Broker只会持久化一条。其中PID是Kafka每次重启都会分配一个新的; Partition 表示分区号; Sequence Number是单调自增的</font>

所以幂等性**只能保证的是在单分区单会话内不重复**

![image-20230105222103723](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230105222103723.png)

**如何使用幂等性？**

开启参数**`enable.idempotence`默认为true，false关闭。**

#### 生产者事务

**<font style="color:red">开启事务，必须开启幂等性</font>**

![image-20230105223413717](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230105223413717.png)

**Kafka的事务一共有如下5个API:**

```java
// 1 初始化事务
void initTransactions();

// 2 开启事务
void beginTransaction() throws Producer FencedException;

// 3 在事务内提交已经消费的偏移量(主要用于消费者)
void sendoffsetsToTransaction (Map<TopicPartition,OffsetAndMetadata> offsets, String  consumerGroupId) throws Producer FencedException;

// 4 提交事务
void commitTransaction() throws ProducerFencedException;

// 5放弃事务(类似于回滚事务的操作)
void abortTransaction() throws ProducerFencedException;
```

示例:

```java
        //指定事务ID
				proper.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "transactional_id_02");

        // 1. 创建kafka生产者对象
        KafkaProducer<String, String> producer = new KafkaProducer<String, String>(proper);

        producer.initTransactions();
        producer.beginTransaction();

        try {
            for (int i = 0; i < 5; i++) {
                producer.send(new ProducerRecord<String, String>("first", "hello kafka" + i)).get();
            }
            producer.commitTransaction();
        } catch (Exception e) {
            producer.abortTransaction();
        } finally {
            // 3. 关闭资源
            producer.close();
        }
```

#### 生产经验——数据有序

- 单分区内，有序。如 一个topic 一个分区，那么在这个分区内数据是有序的。当然还有其他条件;
- 多分区，分区与分区间无序。如果要做有序保证，只能Consumer自己将数据拉取到本地后，自己做处理；

#### 生产经验——数据乱序

1) kafka在1.x版本之前保证数据单分区有序，条件如下:
`max.in.flight.requests.per.connection=1` (不需要考虑是否开启幂等性)。

2) **kafka在1.x及 以后版本保证数据单分区有序**，条件如下:
- 未开启幂等性
`max.in.f1ight.requests.per.connection`需要设置为1;

- 开启幂等性
  `max.in.flight.requests.per.connection`需要设置小于等于5;

  原因说明:因为在kafka 1.x以后，启用幂等后，kafka服 务端会缓存producer发来的最近5个request的元数据， 故无论如何，都可以保证最近5个request的数据都是有序的。

![image-20230106064133679](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230106064133679.png)
