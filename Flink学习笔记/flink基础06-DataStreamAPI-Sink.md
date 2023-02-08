## Sink

其实前面示例中的的print()中就有addSink()操作:
```java
    @PublicEvolving
    public DataStreamSink<T> print() {
        PrintSinkFunction<T> printFunction = new PrintSinkFunction<>();
        return addSink(printFunction).name("Print to Std. Out");
    }
```

Flink官方目前支持的第三方系统连接器:

- Apache Kafka (source/sink)
- Apache Cassandra (sink)
- Amazon Kinesis Streams (source/sink)
- Elasticsearch (sink)
- FileSystem (Hadoop ihcluded) - Streaming only (sink)
- FileSystem (Hadoop included) - Streaming and Batch (sink)
- RabbitMQ (source/sink)
- Apache NiFi (source/sink)
- Twitter Streaming API (source)
- Google PubSub (source/sink)
- JDBC (sink)

除了Flink官方之外，Apache Bahir作为给Spark和Flink提供扩展支持的项目，也实现了一些第三方系统与Flink的连接器。

- Apache ActiveMQ (source/sink)
- Apache Flume (sink)
- Redis (sink)
- Akka (sink)
- Netty (source)

##### 输出到文件

```java
public class SinkToFileTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        // 从元素中读取数据
        DataStreamSource<Event> stream = env.fromElements(
                new Event("Mary", "./home", 1000L),
                new Event("Alice", "./cart", 2000L),
                new Event("Blob", "./prod?id=100", 3000L));
        // <String> 是输出的类型
        StreamingFileSink<String> fileSink = StreamingFileSink.<String>forRowFormat(
                new Path("/root/code/java/flink-demo01/output/"),
                new SimpleStringEncoder<>("UTF-8")) // 简单的String类型输出
                .withRollingPolicy(
                        DefaultRollingPolicy.builder()
                                .withMaxPartSize(1024 * 1024 * 1024) // 文件1GB后就切换
                                .withRolloverInterval(TimeUnit.MINUTES.toMillis(15)) // 每15分钟切换一次
                                .withInactivityInterval(TimeUnit.MINUTES.toMillis(5)) // 如果5分钟没交互,则关闭
                                .build())
                .build();

        stream.map(data -> data.toString()).addSink(fileSink);

        env.execute();
    }
}
```

**最后结果:**

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221126111242290.png" alt="image-20221126111242290" style="zoom:50%;" />

##### 输出到Kafka

下面测试场景，从Kafka中读取数据，经Flink转换操作后，而后将数据写入到Kakfa中。
```java
public class SinToKafka {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        // 1. 从kafka中读取数据
        Properties properties = new Properties();
        properties.setProperty("bootstrap.servers", "hadoop101:9092");
        properties.setProperty("group.id", "consumer-group");
        properties.setProperty("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        properties.setProperty("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        properties.setProperty("auto.offset.reset", "latest");
        DataStreamSource<String> kafkaStream = env
                .addSource(new FlinkKafkaConsumer<String>("clicks", new SimpleStringSchema(), properties));
        // 2. 用flink进行转换处理,
        // 调用map将string 转换成event
        SingleOutputStreamOperator<Event> ret = kafkaStream.map(new MapFunction<String, Event>() {
            @Override
            public Event map(String value) throws Exception {
                String[] fields = value.split(",");
                return new Event(fields[0].trim(), fields[1].trim(), Long.valueOf(fields[2].trim()));
            }
        });
        // 3. 结果数据写入到 kafka
        ret.map(data -> data.toString())
                .addSink(new FlinkKafkaProducer<String>("hadoop101:9092", "events", new SimpleStringSchema()));
        env.execute();
    }
}

mvn compile exec:java -Dexec.mainClass="com.luke.flink.datastreamdemo.SinToKafka"

// 启动kafka并输入
./bin/kafka-console-producer.sh --broker-list localhost:9092 --topic clicks
>Mary,./home,1000
>Alice,./cart,2000
>Bob,./prod?id=10,9000
>Bob,./cart,7000
>Bob,. /home,5000
  
//启动kafka并消费
./bin/kafka-console-consumer.sh --bootstrap-server  localhost:9092 --from-beginning  --topic events
Event [user=Mary, url=./home, timestamp=1970-01-01 00:00:01.0]
Event [user=Alice, url=./cart, timestamp=1970-01-01 00:00:02.0]
Event [user=Bob, url=./prod?id=10, timestamp=1970-01-01 00:00:09.0]
Event [user=Bob, url=./cart, timestamp=1970-01-01 00:00:07.0]
Event [user=Bob, url=. /home, timestamp=1970-01-01 00:00:05.0]
```

##### 输出到Redis

添加依赖:
```xml
    <dependency>
      <groupId>org.apache.bahir</groupId>
      <artifactId>flink-connector-redis_2.11</artifactId>
      <version>1.0</version>
    </dependency>
```



```java
public class SinkToRedis {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        // 从元素中读取数据
        DataStreamSource<Event> stream = env.fromElements(
                new Event("Mary", "./home", 1000L),
                new Event("Alice", "./cart", 2000L),
                new Event("Blob", "./prod?id=100", 5000L),
                new Event("Blob", "./prod?id=101", 300L),
                new Event("Alice", "./pencil", 6000L),
                new Event("Blob", "./prod?id=100", 4000L));
        // 创建一个jedis链接配置
        FlinkJedisPoolConfig config = new FlinkJedisPoolConfig.Builder().setHost("10.211.55.7").setPort(30000)
                .setPassword("redispwd001").build();
        // 写入redis
        stream.addSink(new RedisSink<Event>(config, new MyRedisMapper()));
        env.execute();
    }

    public static class MyRedisMapper implements RedisMapper<Event> {
        @Override
        public RedisCommandDescription getCommandDescription() {
            // 返回redis 命令的描述
            return new RedisCommandDescription(RedisCommand.HSET, "user_clicks_url");
        }

        @Override
        public String getKeyFromData(Event data) {
            // 子类key
            return data.user;
        }

        @Override
        public String getValueFromData(Event data) {
            // 子类value
            return data.url;
        }
    }
}
```

##### 输出到 MySQL

```java
/*pom.xml */
    <dependency>
      <groupId>org.apache.flink</groupId>
      <artifactId>flink-connector-jdbc_2.12</artifactId>
      <version>1.13.0</version>
    </dependency>
    <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
      <version>8.0.11</version>
    </dependency>

public class SinkToMysql {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        // 从元素中读取数据
        DataStreamSource<Event> stream = env.fromElements(
                new Event("Mary", "./home", 1000L),
                new Event("Alice", "./cart", 2000L),
                new Event("Blob", "./prod?id=100", 5000L),
                new Event("Blob", "./prod?id=101", 300L),
                new Event("Alice", "./pencil", 6000L),
                new Event("Blob", "./prod?id=100", 4000L));

        stream.addSink(JdbcSink.sink(
                "INSERT INTO clicks (user,url) VALUES(?,?)",
                ((statement, event) -> { // statement 就是上面填写的insert sql
                    statement.setString(1, event.user);
                    statement.setString(2, event.url);
                }), new JdbcConnectionOptions.JdbcConnectionOptionsBuilder()
                        .withUrl(
                                "jdbc:mysql://10.211.55.7:20000/db_hive?useUnicode=true&characterEncoding=utf-8&useSSL=false&allowPublicKeyRetrieval=true")
                        .withDriverName("com.mysql.cj.jdbc.Driver")
                        .withUsername("hive")
                        .withPassword("helloPass01")
                        .build()));

        env.execute();
    }
}

/* 没有运行成功,还需要再看看 */
```



