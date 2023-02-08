## Broker

#### Zookeeper中保存点信息

![image-20230106065134372](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230106065134372.png)

#### Broker的选举过程

![image-20230106065716895](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230106065716895.png)

#### 生产经验——节点服役和退役

1. 新服役节点Broker3后，执行负载均衡的操作

   - 创建一个要均衡的主题

     ```json
     vim topics-to-move.json
     {
     	"topics":[
     		{"topic":"first"}
     	],
     	"version":1
     }
     
     也可以多个
     {
     	"topics":[
     		{"topic":"first"},{"topic":"events"}
     	],
     	"version":1
     }
     ```

   - 生产负载均衡的计划

     ```java
     bin/kafka-reassign-partitions.sh  --bootstrap-server  hadoop101:9092  --topics-to-move-json-file  topics-to-move. json --broker-list "0,1,2,3" --generate
     ```

     ![image-20230106071537284](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230106071537284.png)

   - 创建副本存储计划( 所有副本存储在broker0、broker1、broker2、broker3中)

     ```
     vim increase-replication-factor.json
     输入内容就是上面的 "未来" 部分
     ```

     <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230106071737528.png" alt="image-20230106071737528" style="zoom:50%;" />
     
   - **执行副本存储计划**
   
     ```shell
     $ bin/kafka-reassign-partitions.sh --bootstrap-server hadoop101:9092 --reassignment-json-file increase-replication-factor.json --execute
     ```
     
   - **验证副本存储计划**
   
     ```shell
     $ bin/kafka-reassign-partitions.sh --bootstrap-server hadoop101:9092 --reassignment-json-file increase-replication-factor.json --verify
     ```
     
     <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230106222126526.png" alt="image-20230106222126526" style="zoom: 50%;" />
     
     ![image-20230106222404689](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230106222404689.png)
   
   ##### 退役旧节点
   
   - **执行负载均衡操作**
   
     先按照退役一台节点，**生成执行计划**，然后按照服役时操作流程**执行负载均衡**。
   
     - **先创建一个要均衡的topic(每个topic逐个处理?)**
   
       ```java
       vim topics-to-move.json
       {
       	"topics":[
       		{"topic":"first"}
       	],
       	"version":1
       }
       ```
   
     - **创建指定计划**
   
       ```shell
       $ bin/kafka-reassign-partitions.sh --bootstrap-server hadoop101:9092 --topics-to-move-json-file topics-to-move.json --broker-list "0,1,2" --generate
       ```
   
       <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230106223207497.png" alt="image-20230106223207497" style="zoom:50%;" />
   
     - **创建副本存储计划**
   
       ```shell
       vim decrease-replication-factor.json
       
       // 未来计划
       ```
   
     - **执行副本存储计划**
   
       ```shell
       $ bin/kafka-reassign-partitions.sh --bootstrap-server hadoop101:9092  --reassignment-json-file decrease-replication-factor.json --execute
       ```
   
     - **验证副本存储计划**
   
       ```shell
       $ bin/kafka-reassign-partitions.sh --bootstrap-server hadoop101:9092 --reassignment-json-file decrease-replication-factor.json --verify
       ```
   

### Kafka的副本

#### 副本基本信息

- Kafka副本作用: 提高数据可靠性

- Kafka默认副本1个，生产环境一般配置2个，保证数据可靠性；太多副本会增加磁盘存储空间，增加网络上数据传入，降低效率；

- Kakfa副本分为: Leader和Follower。Kafka生产者只会把数据发送Leader，然后Follower找Leader进行数据同步;

- kafka中所有副本统称为 AR（Assigned Replicas）

  **AR=ISR + OSR**

  - **ISR**,表示和Leader保持同步的Follower集合。如果Follower长时间未向Leader发送通信请求或同步数据，则该Follower 将被踢出ISR。该时间阈值由`replica.lag.time.max.ms`参数设定，默认30s。Leader 发生故障之后，就会从ISR中选举新的Leader;

  - **OSR**，表示Follower与Leader副本同步时，延迟过多的副本。那OSR中的节点什么时候恢复到 ISR呢?

#### Leader选举过程

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230107094429064.png" alt="image-20230107094429064" style="zoom: 33%;" />

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230107094531892.png" alt="image-20230107094531892" style="zoom: 33%;" />

![image-20230107095206564](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230107095206564.png)

#### Follower 故障处理细节

- **LEO(Log End Offset): 每个副本的最后一个offset，LEO其实就是最新的offset +1**

- **HW(High Water mark): 所有副本中最小的LEO。比如下面图中就是broker1 中LEO**

- 消费者只能拉取到 HW(高水位线)这个offset之前的消息，比如下图中的 5 6 7 拉取不到；

- 下图详细描述：

  a. Follower(Broker2)发生故障后会被临时踢出ISR;
  b. 这个期间Leader(Broker0)和Follower(Broker1)继续接收数据;
  c. 待该Follower(Broker2)恢复后，Follower会读取本地磁盘记录的上次的HW(图中是5)，并将log文件高于HW的部分截取掉，从HW开始
  向Leader进行同步。
  d. 等该**Follower(Broker2)的LEO大于等于该Partition的HW(7+1=8)**， 即Follower(Broker2)追上tLeader之 后，就可以重新加入ISR了;

![image-20230107101026016](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230107101026016.png)

#### Leader故障处理细节

- Leader故障后，会从ISR中剔除；

- 然后根据 ISR(存在) 和 AR(排序) 原则，选择一个new Leader，比如就是下面的Broker1;

- **为了保证多个副本之间的数据一致性，其余Follower会先将各自的log文件高于HW的部分截掉(比如broker2的 5 6数据就要截掉了)，然后炒年糕new Leader同步数据;**

- **注意: 这只能保证副本之间的数据一致性，并不能保证数据不丢失或不重复。**

  <img src="/Users/lukexwang/Library/Application Support/typora-user-images/image-20230107102235628.png" alt="image-20230107102235628" style="zoom:33%;" />

#### 分区副本分配

如果kafka 服务器只有4个节点，那么设置kafka 的分区数大于服务器台数，在kafka底层如何分配存储副本呢?

- 创建16分区，3个副本

  ```shell
  $ ./bin/kafka-topics.sh --bootstrap-server hadoop101:9092 --create --partitions 16 --replication-factor 3 --topic second
  ```

- 查看分区和副本的情况

  ```shell
  $ ./bin/kafka-topics.sh --bootstrap-server hadoop101:9092 --describe --topic second
  Topic: second   TopicId: j0wjIbyQSPyceZMi8EElyA PartitionCount: 16      ReplicationFactor: 3    Configs: segment.bytes=1073741824
          Topic: second   Partition: 0    Leader: 1       Replicas: 1,0,2 Isr: 1,0,2
          Topic: second   Partition: 1    Leader: 0       Replicas: 0,2,1 Isr: 0,2,1
          Topic: second   Partition: 2    Leader: 2       Replicas: 2,1,0 Isr: 2,1,0
          Topic: second   Partition: 3    Leader: 1       Replicas: 1,2,0 Isr: 1,2,0
          Topic: second   Partition: 4    Leader: 0       Replicas: 0,1,2 Isr: 0,1,2
          Topic: second   Partition: 5    Leader: 2       Replicas: 2,0,1 Isr: 2,0,1
          Topic: second   Partition: 6    Leader: 1       Replicas: 1,0,2 Isr: 1,0,2
          Topic: second   Partition: 7    Leader: 0       Replicas: 0,2,1 Isr: 0,2,1
          Topic: second   Partition: 8    Leader: 2       Replicas: 2,1,0 Isr: 2,1,0
          Topic: second   Partition: 9    Leader: 1       Replicas: 1,2,0 Isr: 1,2,0
          Topic: second   Partition: 10   Leader: 0       Replicas: 0,1,2 Isr: 0,1,2
          Topic: second   Partition: 11   Leader: 2       Replicas: 2,0,1 Isr: 2,0,1
          Topic: second   Partition: 12   Leader: 1       Replicas: 1,0,2 Isr: 1,0,2
          Topic: second   Partition: 13   Leader: 0       Replicas: 0,2,1 Isr: 0,2,1
          Topic: second   Partition: 14   Leader: 2       Replicas: 2,1,0 Isr: 2,1,0
          Topic: second   Partition: 15   Leader: 1       Replicas: 1,2,0 Isr: 1,2,0
  ```

  这里可以看`Replicas`的第一列都是`1,0,2`不断重复。其实我也没完全搞懂，反正尽量均匀分配啦；

- **手动调整分区副本存储**

  在生产环境中，每台服务器的配置和性能不一-致，但是Kafka只会根据自己的代码规则创建对应的分区副本，就会导致个别服务器存储压力较大。所有需要手动调整分区副本的存储。

  **需求:创建一个新的topic，4个分区，两个副本，名称为three。将该topic的所有副本都存储到broker0和broker1两台服务器上**

  ![image-20230107110120749](/Users/lukexwang/Library/Application Support/typora-user-images/image-20230107110120749.png)

  a. 创建一个新的 topic，名称为 three

  ```shell
  $ ./bin/kafka-topics.sh --bootstrap-server hadoop101:9092 --create --partitions 4 --replication-factor 2 --topic three
  ```

  b. 查看分区副本存储情况

  ```shell
  $ bin/kafka-topics.sh --bootstrap-server hadoop101:9092 --describe --topic three
  Topic: three    TopicId: 4yLz-lctR_K6FV60PfrDxQ PartitionCount: 4       ReplicationFactor: 2    Configs: segment.bytes=1073741824
          Topic: three    Partition: 0    Leader: 0       Replicas: 0,2   Isr: 0,2
          Topic: three    Partition: 1    Leader: 2       Replicas: 2,1   Isr: 2,1
          Topic: three    Partition: 2    Leader: 1       Replicas: 1,0   Isr: 1,0
          Topic: three    Partition: 3    Leader: 0       Replicas: 0,1   Isr: 0,1
  ```

  c. 创建副本存储计划(所有副本都指定存储在broker0、broker1中)

  ```json
  vim increase-replication-factor.json
  {
  	"version":1,
  	"partitions": [
  	{ "topic": "three", "partition":0,"replicas": [0,1] } ,
  	{"topic": "three", "partition":1, "replicas": [0,1] },
  	{"topic": "three", "partition":2, "replicas": [1,0] } ,
  	{"topic": "three", "partition":3, "replicas": [1,0] }
  	]
  }
  ```

  d. 执行副本存储计划

  ```shell
  ./bin/kafka-reassign-partitions.sh --bootstrap-server hadoop101:9092 --reassignment-json-file increase-replication-factor.json --execute
  Current partition replica assignment
  
  {"version":1,"partitions":[{"topic":"three","partition":0,"replicas":[0,2],"log_dirs":["any","any"]},{"topic":"three","partition":1,"replicas":[2,1],"log_dirs":["any","any"]},{"topic":"three","partition":2,"replicas":[1,0],"log_dirs":["any","any"]},{"topic":"three","partition":3,"replicas":[0,1],"log_dirs":["any","any"]}]}
  
  Save this to use as the --reassignment-json-file option during rollback
  Successfully started partition reassignments for three-0,three-1,three-2,three-3
  ```

  e. 验证副本存储计划

  ```shell
  $ ./bin/kafka-reassign-partitions.sh --bootstrap-server hadoop101:9092 --reassignment-json-file increase-replication-factor.json --verify
  Status of partition reassignment:
  Reassignment of partition three-0 is complete.
  Reassignment of partition three-1 is complete.
  Reassignment of partition three-2 is complete.
  Reassignment of partition three-3 is complete.
  
  Clearing broker-level throttles on brokers 0,1,2
  Clearing topic-level throttles on topic three
  ```

  f. 查看分区副本存储情况

  ```shell
  $ bin/kafka-topics.sh --bootstrap-server hadoop101:9092 --describe --topic three
  Topic: three    TopicId: 4yLz-lctR_K6FV60PfrDxQ PartitionCount: 4       ReplicationFactor: 2    Configs: segment.bytes=1073741824
          Topic: three    Partition: 0    Leader: 0       Replicas: 0,1   Isr: 0,1
          Topic: three    Partition: 1    Leader: 0       Replicas: 0,1   Isr: 1,0
          Topic: three    Partition: 2    Leader: 1       Replicas: 1,0   Isr: 1,0
          Topic: three    Partition: 3    Leader: 0       Replicas: 1,0   Isr: 0,1
  ```

- **Leader Partition负载均衡**

  正常情况下，**Kafka本身会自动把`Leader Partition`均匀分散在 各个机器上**，来保证每台机器的读写吞吐量都是均匀的。但是如果<font style="color:red">某些`broker`宕机，会导致`Leader Partition`过于集中在其他少部分几台broker上。</font>这会导致少数几台broker的读写请求压力过高，其他宕机的broker重启之后都是follower partition,读写请求很低，**造成集群负载不均衡**

  如何解决?

  - **参数`auto.leader.rebalance.enable`，默认是true，自动做Leader Partition平衡**;
  - **什么时候需要触发均衡分布呢? `leader.imbalance.per.broker.percentage`，默认是10%。每个broker允许的不平衡是leader的比率。如果每个broker超过了这个值，控制器会触发leader的平衡。**
  - **`leader.imbalance.check.interval.seconds`，默认值是300秒。检查leader负载是否均衡的时间间隔;**

  如何计算不平衡的比例?

  针对`broker0`节点，分区2的AR优先副本是0节点，但是0节点却不是Leader节点，所以不平衡数加1，AR副本总数是4.
  **所以broker0节点不平衡率为1/4>10%，需要再平衡**

  **broker2和broker3节点和broker0不平衡率一样， 需要再平衡**

  Broker1的不平衡数为0，不需要再平衡。

  <img src="/Users/lukexwang/Library/Application Support/typora-user-images/image-20230107145053005.png" alt="image-20230107145053005" style="zoom:50%;" />

  **一般情况下不建议开启`auto.leader.rebalance.enable`，如果要开启也建议适当扩大`leader.imbalance.per.broker.percentage`**

- **增加副本因子**

  在生产关系中，当某个topic的重要登记需要提升，我们考虑增加副本。副本数的增加需要先指定计划，然后在根据计划执行。

  - 创建topic

    ```shell
    $ ./bin/kafka-topics.sh --bootstrap-server hadoop101:9092 --create --partitions 3 --replication-factor 1 --topic four
    ```

  - 手动增加副本存储

    - **创建副本存储计划(所有副本都指定存储在broker0、broker1、broker2中)**

      ```shell
      vim increase-replication-factor.json
      // 输入如下内容
      {"version":1, 
      	"partitions": [ 
      		{ "topic": "four", "partition":0, "replicas": [0, 1,2]} , 
      		{"topic": " four", "partition":1, "replicas": [0,1,2]} ,
       		{"topic": "four", "partition":2, "replicas": [0,1,2]} 
      ]
      }
      ```

    - **执行副本存储计划**

      ```shell
      $ ./bin/kafka-reassign-partitions.sh --bootstrap-server hadoop101:9092 --reassignment-json-file increase-replication-factor.json --execute
      ```

### Kafka 文件存储

#### 文件存储机制

Topic是逻辑上的概念，而partition是物理 上的概念，<font style="color:red">每个partition对应于一个log文件</font>， 该log 文件中存储的就是Producer生产的数据。<font style="color:red">Producer生产的数据会被不断追加到该log文件末端</font>，为防止log文件过大导致数据定位效率低下，Kafka采取了<font style="color:red">分片和索引机制</font>,将<font style="color:red">每个partition分为多个segment。每个segment包括</font>:“.index"文件、 “.log"文 件和.timeindex等文件。这些文件位于一个文件夹下，该文件夹的命名规则为: **topic名称+分区序号，例如: first-0**

**其实我搞不懂为何还加了一个 逻辑 log层，log层不是物理存在的，有必要吗？Partition就是多个segment组成的不行吗？**

![image-20230107152200887](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230107152200887.png)

```shell
$ ls -lrht /data/kafka/log/first-0/*
-rw-r--r-- 1 root root  43 Jan  3 08:58 /data/kafka/log/first-0/partition.metadata
-rw-r--r-- 1 root root 10M Jan  3 08:58 /data/kafka/log/first-0/00000000000000000000.timeindex
-rw-r--r-- 1 root root 10M Jan  3 08:58 /data/kafka/log/first-0/00000000000000000000.index
-rw-r--r-- 1 root root   8 Jan  3 09:18 /data/kafka/log/first-0/leader-epoch-checkpoint
-rw-r--r-- 1 root root 490 Jan  5 13:07 /data/kafka/log/first-0/00000000000000000000.log

$ ./bin/kafka-run-class.sh kafka.tools.DumpLogSegments --files /data/kafka/log/first-0/00000000000000000000.index
Dumping /data/kafka/log/first-0/00000000000000000000.index
offset: 0 position: 0

$ ./bin/kafka-run-class.sh kafka.tools.DumpLogSegments --files /data/kafka/log/first-0/00000000000000000000.log
Dumping /data/kafka/log/first-0/00000000000000000000.log
Starting offset: 0
baseOffset: 0 lastOffset: 0 count: 1 baseSequence: 0 lastSequence: 0 producerId: 1000 producerEpoch: 0 partitionLeaderEpoch: 0 isTransactional: false isControl: false deleteHorizonMs: OptionalLong.empty position: 0 CreateTime: 1672737508744 size: 73 magic: 2 compresscodec: none crc: 1685517854 isvalid: true
baseOffset: 1 lastOffset: 1 count: 1 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 0 isTransactional: false isControl: false deleteHorizonMs: OptionalLong.empty position: 73 CreateTime: 1672923733273 size: 79 magic: 2 compresscodec: none crc: 87281478 isvalid: true
baseOffset: 2 lastOffset: 2 count: 1 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 0 isTransactional: false isControl: false deleteHorizonMs: OptionalLong.empty position: 152 CreateTime: 1672923811651 size: 81 magic: 2 compresscodec: none crc: 2181863943 isvalid: true
baseOffset: 3 lastOffset: 3 count: 1 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 0 isTransactional: false isControl: false deleteHorizonMs: OptionalLong.empty position: 233 CreateTime: 1672923884010 size: 81 magic: 2 compresscodec: none crc: 516602344 isvalid: true
...
```

**Log文件和Index文件详解**

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230107153144233.png" alt="image-20230107153144233" style="zoom:50%;" />

面试题: kafka 存储日志时，索引是按照什么方式进行组织的？

答: 按照稀疏索引进行存储，就是log文件中存储 4KB数据，index文件中才记录一条索引。

**文件清除策略**

Kafka中<font style="color:red">默认日志保存时间为7天</font>，可通过调整如下参数修改保存时间:

- `log.retention.hours` 最低优先级小时，默认7天
- `log.retention.minutes` 分钟
- `log.retention.ms` 最高优先级毫秒
- `log.retention.check.interval.ms` 负责设置检查周期，默认5分钟

那如果日志一旦超过了设置时间，如何处理呢? Kafka提供了日志清理策略有**delete和compact**两种。

- **delete日志删除: 将过期数据删除**

  - **`log.cleanup.policy=delete` 所有数据启用删除策略**

    - **基于时间:默认打开，<font style="color:red">以segment中所有记录中最大时间戳作为该文件时间戳</font>**

    - **基于大小:默认关闭。**超过设置的所有日志总大小，删除最早的segment。`log.retention.bytes`默认等于-1，表示无穷大。

      比如硬盘只有100GB，存满了，现在基于大小删除，那就是可以删除最旧的内容。`log.retention.bytes:-1`代表关闭，磁盘无穷大。

  - 思考: 如果一个segment中有部分数据过期，另一部分数据没过期，怎么处理?

    <img src="/Users/lukexwang/Library/Application Support/typora-user-images/image-20230107154312926.png" alt="image-20230107154312926" style="zoom:50%;" />

- **compact日志压缩: 对于相同key的不同value值，只保留最后一个版本**

  - `log.cleanup.policy=compact` 所有数据启用压缩策略

    压缩后的offset可能是不连续的，比如上图中没有6，当从这些offset消费消息时，将会拿到比这个offset大的offset对应的消息，实际上会拿到offset为7的消息，并从这个位置开始消费。

    <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230107154753207.png" alt="image-20230107154753207" style="zoom:50%;" />

    **这种策略只适合特殊场景，比如消息的key是用户ID, value是用户的资料，通过这种压缩策略，整个消息集里就保存了所有用户最新的资料**


### 高效读写

- **Kafka 本身是分布式集群，可以采用分区技术，并行度高**

- **读数据采用稀疏索引，可以快速定位要消费的数据**

- **顺序写磁盘**

  Kafka的producer生产数据，要写入到log文件中，写的过程是**一直追加**到文件末端, 为顺序写。官网有数据表明，同样的磁盘，顺序写能到600M/s，而随机写只有100K/s。 这与磁盘的机械机构有关，顺序写之所以快，是因为其省去了大量磁头寻址的时间;

  <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230110062433099.png" alt="image-20230110062433099" style="zoom:50%;" />

- **页缓存+零拷贝技术**

  **零拷贝**: Kafka的数据加工处理操作交由Kafka生产者和Kafka消费者处理。**Kafka Broker应用层不关心存储的数据，所以就不用走应用层，传输效率高**

  **PageCache页缓存**: Kafka 重度依赖底层操作系统提供的PageCache功能。当上层有写操作时，操作系统只是将数据写入PageCache。当读操作发生时，先从PageCache中查找，如果找不到，再去磁盘中读取。实际上PageCache是把尽可能多的空闲内存都当做了磁盘缓存来使用。

  <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230110062945057.png" alt="image-20230110062945057" style="zoom:50%;" />

- 