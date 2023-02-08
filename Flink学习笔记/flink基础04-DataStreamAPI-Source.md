## DataStream API(基础篇)

- DataStream(数据流)本身是Flink中一个用来表示数据集合的类;
- 编写Flink代码其实就是基于这种数据类型的处理，所以这套核心API就以DataStream命名;
- DataStream在用法上有些类似于常规的Java集合，但又有所不同。我们再代码中往往并不关心集合中具体数据，只是以API定义出一连串的操作处理他们，该过程称为数据流的转换(transformations);
- Flink程序的几部分构成:
  - **获取执行环境(execution environment)**
  - **读取数据源(source)**
  - **定义基于数据的转换操作(transformations)**
  - **定义计算结果的输出位置(sink)**
  - **触发程序执行(execute)**

#### 执行环境(execution environment)

**创建执行环境**

- 我们获取的执行环境，是`StreamExecutionEnvironment`类对象，这是所有Flink程序的基础;

- 创建执行环境的方式，就是调用这个类的静态方法:

  - **getExecutionEnvironment**

    该方法根据当前运行的上下文直接得到正确的结果: 
    **如果程序是独立运行，就返回一个本地执行环境;**
    **如果是创建了 jar 包，然后从命令行调用它并提交到集群执行，那么就返回集群的执行环境;**

    <font style="color:red">**这种智能的方式不需要我们额外做判断，用起来简单高效，是最常用的一种创建执行环境的方式。**</font>

    也就是说该方法会根据当前的运行方式，自行决定返回什么样的运行环境:

    ```java
    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
    ```

  - **createLocalEnvironment**

    该方法返回一个本地执行环境。可在调用时传入一个参数，指定默认的并行度，如果不传入，则默认并行度是本地`CPU`核心数。
    ```java
    StreamExecutionEnvironment localEnv = StreamExecutionEnvironment.createLocalEnvironment();
    ```

  - **createRemoteEnvironment**

    该方法返回集群执行环境。需在调用时指定`JobManager`的主机名和端口号，并指定要在集群中运行的Jar包

    ```java
    StreamExecutionEnvironment remoteEnv = StreamExecutionEnvironment.createRemoteEnvironment("host",1234,"path/to/jarFile.jar")
    ```

  在获取到程序执行环境后，我们还可以对执行环节进行灵活的配置。比如可以全局设置程序的并行度、禁用算子链，还可以定义程序的时间语义、配置容错机制等。

#### 执行模式(execution mode)

流处理模式、批处理模式;
Flink 1.12.0版本之前，批处理的执行环境和流处理的类似，调用类的`ExecutionEnvironment`静态方法，返回它的对象:

```java
// 批处理环境
ExecutionEnvironment batchEnv= ExecutionEnvironment.getExecutionEnvironment();

// 流处理环境
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
```

基于`ExecutionEnvironment`读入数据创建的数据集合，就是`DataSet,`对应的调用的一整套转换方法，就是`DataSet API`;
**从Flink 1.12.0 版本以后，Flink实现了API上的流批统一。DataStream API新增了一个重要特性: 可以支持不同的"执行模式(execution mode)", 通过简单的设置就可以让一段Flink程序在流处理和批处理之间切换。此后，DataSet API就没有存在的必要了 **

- **流处理模式(STREAMING)**

  默认模式

- **批处理模式(BATCH)**

  BATCH模式的配置方法:

  - 通过命令行配置:

    ```shell
    bin/flink run -Dexecution.runtime-mode=BATH ...
    ```

  - 通过代码配置:

    ```java
    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
    env.setRuntimeMode(RuntimeExecutionMode.BATCH);
    ```

    **建议: 不要在代码中配置，而是使用命令行，因为命令行更灵活。便于一套代码同时用于处理 流数据 和 批数据。**

    **什么时候选择BATCH模式?**

    <font style="color:blue">流处理模式对有界数据和无界数据都是有效的；而批处理模式仅能用于有界数据。看起来BATCH模式似乎被STREAMING模式全覆盖了，那还有存在的必要吗？能否所有情况都用流处理模式呢？</font>

    当然是可以的，但是有时这样并不高效，或者说结果不是我们想要的。比如前面wordcount程序中，流处理和批处理的不同: 在流处理中每行数据都会输出一次结果；而批处理时，只有数据全部处理完之后，才会一次性输出结果。流处理会得到更多中间结果。

- **自动模式(AUTOMATIC)**

  由程序根据输入数据源是否有界，来自动选择执行模式。

####  源算子(Source)

一般将数据的输入来源称为data source, 而读取数据的算子就是源算子(source operator)，所以source是我们整个处理程序的输入端。
一般通过`addSource()`方法 添加source;

```java
DataStream<String> stream = env.addSource(...);
```

- **方法输入一个对象参数，需要实现`SourceFunction`接口,返回DataStreamSource**

- 示例:

  ```java
  /* Event.java */
  package com.luke.flink.datastreamdemo;
  import java.sql.Timestamp;
  /* 
   * 定义的event的几个特点:
   * a.类是公有(public)的
   * b.有一个无参数的构造方法
   * c.所有属性公有(public)
   * d.所有属性的类型都是可序列化的;
  */
  public class Event {
      public String user;
      public String url;
      public Long timestamp;
  
      @Override
      public String toString() {
          return "Event [user=" + user + ", url=" + url + ", timestamp=" + new Timestamp(timestamp) + "]";
      }
  
      public Event(String user, String url, Long timestamp) {
          this.user = user;
          this.url = url;
          this.timestamp = timestamp;
      }
  
      public Event() {
      }
  }
  ```

  **测试数据源(data source)的调用:**

  ```java
  package com.luke.flink.datastreamdemo;
  
  import java.util.ArrayList;
  import java.util.Properties;
  
  import org.apache.flink.api.common.serialization.SimpleStringSchema;
  import org.apache.flink.streaming.api.datastream.DataStreamSource;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer;
  
  // 数据源测试
  public class SourceTest {
      public static void main(String[] args) throws Exception {
          StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
          env.setParallelism(1); // 设置全局并行度为1
  
          // 方式1: 从文本中读取数据
          DataStreamSource<String> stream1 = env.readTextFile("/root/code/java/flink-demo01/input/clicks.txt");
          stream1.print("1");
  
          // 方式2: 从集合中读取数据,测试常用
          ArrayList<Integer> nums = new ArrayList<>();
          nums.add(2);
          nums.add(3);
          DataStreamSource<Integer> numStream = env.fromCollection(nums);
  
          numStream.print("nums");
  
          ArrayList<Event> events = new ArrayList<>();
          events.add(new Event("Mary", "./home", 1000L));
          events.add(new Event("Alice", "./cart", 2000L));
          events.add(new Event("Blob", "./prod?id=100", 3000L));
          DataStreamSource<Event> stream2 = env.fromCollection(events);
          stream2.print("2");
  
          // 方式三: 从元素读取数据,测试常用
          DataStreamSource<Event> stream3 = env.fromElements(
                  new Event("Mary", "./home", 1000L),
                  new Event("Alice", "./cart", 2000L),
                  new Event("Blob", "./prod?id=100", 3000L));
  
          stream3.print("3");
  
          // 方式四: 从socket文本流中读取,测试
          // DataStreamSource<String> stream4 = env.socketTextStream("10.211.55.6", 7777);
          // stream4.print("4");
  
          // 方法五: 从kafka中读取数据
          Properties properties = new Properties();
          properties.setProperty("bootstrap.servers", "hadoop101:9092");
          properties.setProperty("group.id", "consumer-group");
          properties.setProperty("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
          properties.setProperty("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
          properties.setProperty("auto.offset.reset", "latest");
          DataStreamSource<String> kafkaStream = env
                  .addSource(new FlinkKafkaConsumer<String>("clicks", new SimpleStringSchema(), properties));
  
          kafkaStream.print("kafka");
  
          env.execute();
  
      }
  }
  ```
  
  添加连接kafka的链接依赖:
  ```xml
      <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-connector-kafka_${scala.binary.version}</artifactId>
        <version>${flink.version}</version>
      </dependency>
  ```
  
  kafka中创建topic并生产数据:
  
  ```shell
  $ ./bin/kafka-console-producer.sh --broker-list localhost:9092 --topic clicks
  >Mary,./home,1000
  >Alice,./cart,2000
  >Bob,./prod?id=100,3000
  ```
  
  输出的结果:
  ```shell
  nums> 2
  nums> 3
  3> Event [user=Mary, url=./home, timestamp=1970-01-01 00:00:01.0]
  3> Event [user=Alice, url=./cart, timestamp=1970-01-01 00:00:02.0]
  3> Event [user=Blob, url=./prod?id=100, timestamp=1970-01-01 00:00:03.0]
  2> Event [user=Mary, url=./home, timestamp=1970-01-01 00:00:01.0]
  2> Event [user=Alice, url=./cart, timestamp=1970-01-01 00:00:02.0]
  2> Event [user=Blob, url=./prod?id=100, timestamp=1970-01-01 00:00:03.0]
  1> Mary,./home,1000
  1> Alice,./cart,2000
  1> Bob,./prod?id=100,3000
  1> Bob,./cart,4000
  1> Bob,. /home,5000
  1> Mary,. /home,6000
  1> Bob,./cart,7000
  1> Bob,. /home,8000
  1> Bob,./prod?id=10,9000
  kafka> Mary,./home,1000
  kafka> Alice,./cart,2000
  kafka> Bob,./prod?id=100,3000
  ```
  
  ##### 自定义source: 主要实现`SourceFunction`接口
  
  ```java
  /* ClickLogs.java */
  package com.luke.flink.datastreamdemo;
  
  import java.util.Calendar;
  import java.util.Random;
  
  import org.apache.flink.streaming.api.functions.source.SourceFunction;
  
  public class ClickLogs implements SourceFunction<Event> {
      private Boolean running = true;
  
      @Override
      public void run(SourceContext<Event> ctx) throws Exception {
          Random random = new Random();
  
          String[] users = { "Mary", "Alice", "Blob", "Cary" };
          String[] urls = { "./cart", ". /home", "./home", "./prod?id=10", "./prod?id=100" };
  
          while (running) {
              String user = users[random.nextInt(users.length)];
              String url = urls[random.nextInt(urls.length)];
              Long timestamp = Calendar.getInstance().getTimeInMillis();
              ctx.collect(new Event(user, url, timestamp));
              Thread.sleep(1000L);
          }
      }
  
      @Override
      public void cancel() {
          running = false;
      }
  }
  
  /*  SourceCustomTest.java */
  package com.luke.flink.datastreamdemo;
  
  import org.apache.flink.streaming.api.datastream.DataStreamSource;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  
  public class SourceCustomTest {
      public static void main(String[] args) throws Exception {
          StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
          env.setParallelism(1);
          DataStreamSource<Event> customesrc = env.addSource(new ClickLogs());
          customesrc.print();
          env.execute();
      }
  }
  ```
  
  输出:
  
  <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221123230626547.png" alt="image-20221123230626547" style="zoom:50%;" />
  
  ##### 修改上面的`ClickLogs`让其支持`setParallelism(2)` 设置并行度，并发生成数据:
  
  ```java
  package com.luke.flink.datastreamdemo;
  import java.util.Random;
  import org.apache.flink.streaming.api.functions.source.ParallelSourceFunction;
  
  public class ParallelCustomSource implements ParallelSourceFunction<Integer> {
      private Boolean running = true;
      private Random random = new Random();
      @Override
      public void run(SourceContext<Integer> ctx) throws Exception {
          while (running) {
              ctx.collect(random.nextInt(100000));
          }
      }
      @Override
      public void cancel() {
          running = false;
      }
  }
  
  DataStreamSource<Integer> customesrc02 = env.addSource(new ParallelCustomSource());
  ```

#### Flink 支持的数据类型

**所有在Flink中支持的数据类型都是 父类 TypeInformation 的子类。**
(1) 基本类型
所有Java基本类型及其包装类，再加上Void、String、 Date、 BigDecimal 和BigInteger.
(2) 数组类型
包括基本类型数组(Primitive Array)和对象数组(Object Array)
(3) 复合数据类型

- Java 元组类型(Tuple): 这是Flink内置的元组类型，是JavaAPI的一-部分。最多25个字段，也就是从Tuple0~Tuple25，不支持空字段;
- Scala样例类及Scala元组:不支持空字段;
- 行类型(Row): 可以认为是具有任意个字段的元组，并支持空字段;
- POJO: Flink 自定义的类似于Java bean模式的类;

(4) 辅助类型
Option、Either、 List、 Map 等
(5) 泛型类型(Generic)
Flink支持所有的Java类和Scala类。不过如果没有按照上面POJO类型的要求来定义;