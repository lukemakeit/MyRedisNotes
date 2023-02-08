### Topic命令

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230103165448180.png" alt="image-20230103165448180" style="zoom:50%;" />

示例:

```shell
# 查看所有topics
$ ./bin/kafka-topics.sh --bootstrap-server hadoop101:9092,hadoop102:9092 --list
__consumer_offsets
clicks
events
testGroup

# 创建topic,必须指定partition数 和 副本数
$ ./bin/kafka-topics.sh --bootstrap-server hadoop101:9092,hadoop102:9092 --topic first --create --partitions 1 --replication-factor 3
Created topic first.

# 查看topic 详情
# Replicas: 2,1,0 代表三个副本
# Leader: 2 代表是副本中的leader
$ ./bin/kafka-topics.sh --bootstrap-server hadoop101:9092,hadoop102:9092 --topic first --describe
Topic: first    TopicId: k4JEhCgKSEiKIl3fNuWQ_w PartitionCount: 1       ReplicationFactor: 3    Configs: segment.bytes=1073741824
        Topic: first    Partition: 0    Leader: 2       Replicas: 2,1,0 Isr: 2,1,0

# 修改分区数，分区数只能增加，不能减少
$ ./bin/kafka-topics.sh --bootstrap-server hadoop101:9092,hadoop102:9092 --topic first --alter --partitions 3

# 不能通过命令行方式修改副本数,会报错
$ ./bin/kafka-topics.sh --bootstrap-server hadoop101:9092,hadoop102:9092 --topic first --alter --replication-factor 2
```

### 生产者 VS 消费者

```shell
# 生产者
$ ./bin/kafka-console-producer.sh --broker-list hadoop101:9092,hadoop102:9092 --topic first
>hello
>java
>good

# 消费者
$ ./bin/kafka-console-consumer.sh --bootstrap-server  hadoop101:9092,hadoop102:9092 --from-beginning  --topic first
hello
java
good
```



