### Flink 简介

Apache Flink是一个**框架和分布式**处理引擎，用于对**无界和有界数据流进行状态计算。**

#### Flink框架处理流程

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221023075440178.png" alt="image-20221023075440178" style="zoom:67%;" />

#### 传统数据处理框架

- **事务处理(OLTP 在线事务处理)**

  <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221023080857976.png" alt="image-20221023080857976" style="zoom:50%;" />

- **分析处理(OLAP 在线分析处理)**

  数据丢到hive中，数据量可以很大，但是不够实时。

#### 流处理的演变

** 历史lambda架构中的两套系统**

- 批处理: 结果较慢，但保证了数据顺序，结果正确;
- 流处理: 结果实时更新，保证了数据快速响应，但结果不一定最终准确

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221023082051753.png" alt="image-20221023082051753" style="zoom:67%;" />

**新一代流处理器——Flink**

(事件时间 和 处理时间)

#### Flink应用架构

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221023082515612.png" alt="image-20221023082515612" style="zoom: 50%;" />

#### Flink的分层API

- 越顶层越抽象，表达含义越简明，使用越方便
- 越底层越具体，表达能力越丰富，使用越灵活

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221023083039636.png" alt="image-20221023083039636" style="zoom:33%;" />

#### Flink VS Spark

**数据处理架构**

- Spark: 底层批处理，流处理就是更小的批，**微批次**
- Flink 对批处理: 有界数据 和 无解数据

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221023083700141.png" alt="image-20221023083700141" style="zoom:33%;" />

**数据处理模型**

- Spark 采用RDD模型，Spark streaming的DStream实际上 也就是一组组小批数据RDD的集合
- Flink基本数据模型是数据流，以及事件(Event)序列;

**运行时架构**

- Spark是批计算，将DAG划分为不同的stage，一个完成后才计算下一个;
- flink是标准的流执行模式，一个事件在一个节点处理完后可以直接发往下一个节点进行处理;

#### 经典批处理Demo编写(Flink 1.12 批流统一后很少用了)

这里是**DataSet API**，目前已处于**软弃用(soft deprecated)**状态了。

```shell
$ mvn archetype:generate  -DgroupId=com.luke  -DartifactId=flink-demo01  -DarchetypeArtifactId=maven-archetype-quickstart -Dversion=0.0.1-snapshot -DinteractiveMode=false
```

编辑`pom.xml`引入依赖:
```xml
  <build>
    <plugins>
      <plugin>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.6.1</version>
        <configuration>
          <source>1.8</source>
          <target>1.8</target>
        </configuration>
      </plugin>
      <plugin>
        <artifactId>maven-assembly-plugin</artifactId>
        <configuration>
          <descriptorRefs>
            <descriptorRef>jar-with-dependencies</descriptorRef>
          </descriptorRefs>
        </configuration>
        <executions>
          <execution>
            <id>make-assembly</id>
            <phase>package</phase>
            <goals>
              <goal>single</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
  
  <!-- 统一设置版本属性,相当于全局变量 -->
  <properties>
    <flink.version>1.13.0</flink.version>
    <java.version>1.8</java.version>
    <scala.binary.version>2.12</scala.binary.version>
    <slf4j.version>1.7.35</slf4j.version>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
  </properties>
  <dependencies>
    <dependency>
      <groupId>org.apache.flink</groupId>
      <artifactId>flink-java</artifactId>
      <version>${flink.version}</version>
    </dependency>
    <dependency>
      <groupId>org.apache.flink</groupId>
      <artifactId>flink-streaming-java_${scala.binary.version}</artifactId>
      <version>${flink.version}</version>
    </dependency>
    <dependency>
      <groupId>org.apache.flink</groupId>
      <artifactId>flink-clients_${scala.binary.version}</artifactId>
      <version>${flink.version}</version>
    </dependency>
    <!--日志管理相关依赖-->
    <dependency>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-api</artifactId>
      <version>${slf4j.version}</version>
    </dependency>
    <dependency>
      <groupId>org.apache.logging.log4j</groupId>
      <artifactId>log4j-to-slf4j</artifactId>
      <version>2.14.0</version>
    </dependency>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>3.8.1</version>
      <scope>test</scope>
    </dependency>
  </dependencies>
```

设置`src/main/resources/log4j.properties`

```properties
log4j.rootLogger=error, stdout
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.ConversionPattern=%-4r [%t] %-5p %c %x - %m%n
```

文件: `input/words.txt`
```txt
hello hello world
good nice a b a
```

main代码:

```java
package com.luke.flink.wordcount;

import org.apache.flink.api.common.typeinfo.Types;
import org.apache.flink.api.java.ExecutionEnvironment;
import org.apache.flink.api.java.operators.AggregateOperator;
import org.apache.flink.api.java.operators.DataSource;
import org.apache.flink.api.java.operators.FlatMapOperator;
import org.apache.flink.api.java.operators.UnsortedGrouping;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.util.Collector;
import java.lang.Exception;

public class BatchWordCount {
    public static void main(String[] args) throws Exception {
        // 1. 创建执行环境
        ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

        // 2. 从文件中读取数据
        // 逐行读取
        DataSource<String> lineDataSource = env.readTextFile("/root/code/java/flink-demo01/input/words.txt");

        // 3. 每行数据进行分词, 转换成二元组,比如 hello => (hello,1)
        // flatmap的输出需要定义一个 Collector<>
        FlatMapOperator<String, Tuple2<String, Long>> wordAndOneTuple = lineDataSource
                .flatMap((String line, Collector<Tuple2<String, Long>> out) -> {
                    // 分词
                    String[] words = line.split("\\s+");
                    // 转换成二元组输出
                    for (String word : words) {
                        out.collect(Tuple2.of(word, 1L)); // 必须是 1L
                        // out.collect(); 如果有两条 out.collect(),则代表一个单词多次输出
                    }
                }).returns(Types.TUPLE(Types.STRING, Types.LONG)); // 定义返回类型?
        // 4. 按照word进行分组
        // groupBy(0) 用第一个词做group
        UnsortedGrouping<Tuple2<String, Long>> wordAndOneGroup = wordAndOneTuple.groupBy(0);

        // 5. 分组内进行聚合统计,二元组中以第二个字段的值做累加
        AggregateOperator<Tuple2<String, Long>> sum = wordAndOneGroup.sum(1);

        sum.print();
    }
}
```

**运行命令:**

```shell
$ mvn compile exec:java -Dexec.mainClass="com.luke.flink.wordcount.BatchWordCount"
```

结果输出:

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221025065445730.png" alt="image-20221025065445730" style="zoom:50%;" />

#### 有界流处理Demo编写(从文件读取数据,模拟批处理)

从Flink 1.12开始，官方推荐直接使用DataStream API，**默认是流处理方式执行**，用户也可以在提交任务时通过**将执行模式设置为BATCH来进行 批处理**:

`$ bin/flink run -Dexecution.runtime-mode=BATCH BatchWordCount.jar`。

**流处理转换为批处理 原理就是: 有界流处理就是批处理。**

Wordcount代码详情:

```java
package com.luke.flink.wordcount;

import org.apache.flink.api.common.typeinfo.Types;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.KeyedStream;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.util.Collector;

/*
 * 有界流处理 就是 批处理
*/
public class BoundedStreamWordCount {
    public static void main(String[] args) throws Exception {
        // 1. 创建流式的执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        // 2. 读取文件
        DataStreamSource<String> lineDataStream = env.readTextFile("/root/code/java/flink-demo01/input/words.txt");

        // 3. 转换计算
        // 每行数据进行分词, 转换成二元组,比如 hello => (hello,1)
        // flatmap的输入是String line,一行的内容;输出需要定义一个 Collector<>,这里是 Collector<String,Long>
        SingleOutputStreamOperator<Tuple2<String, Long>> wordAndOneTuple = lineDataStream
                .flatMap((String line, Collector<Tuple2<String, Long>> out) -> {
                    // 分词
                    String[] words = line.split("\\s+");
                    // 转换成二元组输出
                    for (String word : words) {
                        out.collect(Tuple2.of(word, 1L)); // 必须是 1L
                    }
                }).returns(Types.TUPLE(Types.STRING, Types.LONG)); // 定义返回类型?

        // 4.分组group
        // data -> data.f0 用的是lambda表达式, 从Tuple2中拿到属性 f0,根据f0来分组
        KeyedStream<Tuple2<String, Long>, String> keyedStream = wordAndOneTuple.keyBy(data -> data.f0);

        // 5.求和, 对Tuple2<String,Long>中Position=1的元素做求和
        SingleOutputStreamOperator<Tuple2<String, Long>> sum = keyedStream.sum(1);

        // 6.打印
        sum.print();

        // 7.启动执行(流处理会等待数据不断来,不断打印?)
        env.execute();
    }
}
```

执行命令:
```java
mvn compile exec:java -Dexec.mainClass="com.luke.flink.wordcount.BoundedStreamWordCount"
```

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221025065316150.png" alt="image-20221025065316150" style="zoom: 50%;" />

- 输出结果前的: `1>`、`3>`是因为本地在模拟Flink在分布式框架上跑，所以启动了多线程，因为是3核的机器，所以编号是`1`到`3`;
- 输出结果中`hello,1`和`hello,2`是肯定在一个线程上处理的，所以相同的key前面的编号相同;

#### 无界流处理，通过netcat监听端口，不断打印

1. 在另外的机器上启用一个`netcat`: `nc -lk 7777`;
2. 编写无界流处理代码:

```java
public class StreamWordCount {
    public static void main(String[] args) throws Exception {
        // 1. 创建流式的执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        // 2. 去读流数据
        DataStreamSource<String> lineDataStream = env.socketTextStream("10.211.55.6", 7777);
       
       // 3. 后面和有界流处理方式一模一样
       .... 
   }
}
```

3. 执行:

   ```java
   mvn compile exec:java -Dexec.mainClass="com.luke.flink.wordcount.StreamWordCount"
   ```

4. 结果情况

   <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221025071135977.png" alt="image-20221025071135977" style="zoom:50%;" />
