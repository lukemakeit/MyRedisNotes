[toc]

## TableAPI

引入依赖:

```xml
    <dependency>
      <groupId>org.apache.flink</groupId>
      <!--flink-table-api-java-bridge_${scala.binary.version}-->
      <!--这个包是table-api 和 dataStream之间的桥接器,负责两者之间的转换 -->
      <artifactId>flink-table-api-java-bridge_2.12</artifactId>
      <!--${flink.version}-->
      <version>1.13.0</version>
    </dependency>
```

如果我们希望在本地的IDE中运行Table API和SQL，还需引入如下依赖:

```xml
    <dependency>
      <groupId>org.apache.flink</groupId>
      <artifactId>flink-table-planner-blink_2.12</artifactId>
      <version>1.13.0</version>
    </dependency>
    <dependency>
      <groupId>org.apache.flink</groupId>
      <artifactId>flink-streaming-scala_2.12</artifactId>
      <version>1.13.0</version>
    </dependency>
```

这里主要添加的依赖是一个“计划器”(planner)，它是TableAPI的核心组件，负责提供运行时环境，并生成程序的执行计划。这里我们用到的是新版的blink planner。 由于Flink安装包的lib目录下会自带planner，所以在生产集群环境中提交作业不需要

示例:

```java
package com.luke.flink.datastreamdemo;

import java.time.Duration;

import org.apache.flink.api.common.eventtime.SerializableTimestampAssigner;
import org.apache.flink.api.common.eventtime.WatermarkStrategy;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.table.api.Table;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
import org.apache.flink.table.expressions.Expression;

public class TableSimpleDemo {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);

        // 1. 读取数据,得到DataStream
        SingleOutputStreamOperator<Event> stream = env.addSource(new ClickLogs())
                .assignTimestampsAndWatermarks(WatermarkStrategy.<Event>forBoundedOutOfOrderness(Duration.ZERO)
                        .withTimestampAssigner(new SerializableTimestampAssigner<Event>() {
                            @Override
                            public long extractTimestamp(Event element, long recordTimestamp) {
                                return element.timestamp;
                            }
                        }));

        // 2. 创建一个表执行环境
        StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);

        // 3. 将DataStream转换成table
        Table eventTable = tableEnv.fromDataStream(stream);

        // 4. 直接写SQL进行转换,timestamp是sql中的关键字,所以需要加反引号
        Table resultTable = tableEnv.sqlQuery("select user,url,`timestamp` from " + eventTable);
        
        // 下面命令isEqual() 没运行成功
        // Table ret2 = eventTable.select($("user"),
        // $("url")).where($("user").isEqual("Alice"));

        tableEnv.toDataStream(resultTable).print("result1:");
        // tableEnv.toDataStream(ret2).print("result2:");

        // 5. 基于table直接转换

        env.execute();
    }

    private static Expression $(String string) {
        return null;
    }
}
// 输出
result1:> +I[Mary, ./home, 1671325840148]
result1:> +I[Bob, ./home, 1671325841155]
result1:> +I[Alice, ./prod?id=100, 1671325842156]
result1:> +I[Bob, . /home, 1671325843159]
result1:> +I[Cary, ./prod?id=100, 1671325844177]
...
```

### 基本API

在Flink中，Table API和SQL可以看作联结在一起的一套API，这套API的核心概念就是“表”(Table)。在我们的程序中，输入数据可以定义成一张表;然后对这张表进行查询，就可以得到新的表，这相当于就是流数据的转换操作;最后还可以定义一张用于输出的表，负责将处理结果写入到外部系统。
**我们可以看到，程序的整体处理流程与DataStreamAPI非常相似，也可以分为读取数据源(Source)、转换(Transform)、输出数据(Sink) 三部分;只不过这里的输入输出操作不需要额外定义，只需要将用于输入和输出的表定义出来，然后进行转换查询就可以了**

**程序基本架构**

```java
//创建表环境
TableEnvironment tableEnv = .. .;

//创建输入表，连接外部系统读取数据, connector 就是连接外部系统
tableEnv. executeSql ( "CREATE TEMPORARY TABLE inputTable . .. WITH ( ' connector ')") ;

//注册一个表，连接到外部系统，用于输出
tableEnv. executeSql ( "CREATE TEMPORARY TABLE outputTable . . . WITH ( ' connector ')") ;

//执行SQL对表进行查询转换，得到一个新的表
Table tablel = tableEnv. sqlQuery ("SELECT . . .FROM inputTable... ") ;

//使用Table API对表进行查询转换，得到-个新的表
Table table2 = tableEnv. from ("inputTable") .select(...);

// 将得到的结果写入输出表
TableResult tableRet=table.executeInsert("outputTable");
```

这样就可以完全抛开`DataStream API`，直接用SQL语句实现全部的流处理过程。

##### 创建表环境

使用Table API 和SQL 需要一个特别的运行时环境，就是所谓的"表环境"(TableEnvironment)。它主要负责:

- 注册Catalog 和表
- 执行SQL 查询
- 注册用户自定义函数(UDF)
- DataStream 和 表之间的转换

**这里的Catalog就是“目录”，与标准SQL中的概念是一致的，主要用来管理所有数据库(database)和表(table) 的元数据(metadata)。 通过Catalog可以方便地对数据库和表进行查询的管理,所以可以认为我们所定义的表都会“挂靠”在某个目录下，这样就可以快速检索。在表环境中可以由用户自定义Catalog,并在其中注册表和自定义函数(UDF)。默认的Catalog就叫作default catalog**

```java
package com.luke.flink.datastreamdemo;

import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.table.api.EnvironmentSettings;
import org.apache.flink.table.api.TableEnvironment;

public class TableCommonApiTest {
    public static void main(String[] args) {
        // 方法一
        // StreamExecutionEnvironment env =
        // StreamExecutionEnvironment.getExecutionEnvironment();
        // env.setParallelism(1);

        // StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);

        // 方法二 定义环境配置来创建执行环境
        EnvironmentSettings setting = EnvironmentSettings.newInstance().inStreamingMode().useBlinkPlanner().build();

        // 方法三 基于blink版本planner进行批处理
        EnvironmentSettings setting3 = EnvironmentSettings.newInstance().inBatchMode().useBlinkPlanner().build();

        TableEnvironment tableEnv = TableEnvironment.create(setting);
    }
}
```

##### 创建表

**为了方便地查询表，表环境中会维护一个目录(Catalog) 和表的对应关系。所以表都是通过Catalog来进行注册创建的。表在环境中有一一个唯一的 ID,由三部分组成:目录(catalog)名，数据库(database)名，以及表名。在默认情况下，目录名为`default_ catalog`,数据库名为`default_ database`。所以如果我们直接创建一个叫作`MyTable`的表，它的ID就是**:

```
default catalog.default database.MyTable
```
具体创建表的方式，有通过连接器(connector) 和虚拟表( virtual tables)两种。

- 连接器表(Connector Tables)

  **最直观的创建表的方式，就是通过连接器(connector) 连接到一个外部系统，然后定义出对应的表结构。例如我们可以连接到Kafka或者文件系统,将存储在这些外部系统的数据以“表”的形式定义出来，这样对表的读写就可以通过连接器转换成对外部系统的读写了。当我们在表环境中读取这张表,连接器就会从外部系统读取数据并进行转换;而当我们向这张表写入数据，连接器就会将数据输出(Sink)到外部系统中**

  在代码中，我们可以调用表环境的`executeSql()`方法， 可以传入一个DDL作为参数执行SQL操作。这里我们传入一个CREATE语句进行表的创建，并通过WITH关键字指定连接到外部系统的连接器:

  ```java
  tableEnv.executeSql("CREATE [TEMPORARY] TABLE MyTable ... WITH ('connector'=...)");
  ```

  如果希望使用自定义的目录名和表名，可以再环境中进行设置:

  ```java
  tableEnv.useCatalog("custom_catalog");
  tableEnv.useDatabase("custom_database");
  ```

  这样表完整的ID就变成了`custom_catalog.custom_database.MyTable`。之后创建的表，ID都以`custom_catalog.custom_database`。

  ```java
          // 2. 创建表
          String createDDL = "CREATE TABLE  clickTable (" +
                  "user_name STRING, " +
                  "url STRING, " +
                  "ts BIGINT " +
                  ") + WITH (" +
                  " 'connector' = 'filesystem'," +
                  " 'path' = 'input/clicks.txt'," +
                  " 'format' = 'csv' " + // 定义分隔符
                  ")";
          tableEnv.executeSql(createDDL);
  ```

  

- 虚拟表

  在环境中注册之后，我们就可以再SQL中直接使用这张表进行查询转换了。

  ```java
  Table newTable = tableEnv.sqlQuery("SELECT ... FROM MyTable ...");
  ```

  这里调用了表环境的`sqlQuery()`方法，直接传入一条SQL语句作为参数执行查询，得到的结果是一个Table对象。Table 是Table API中提供的核心接口类，就代表了一个Java中定义的表实例。

  得到的`newTable`是一个中间转换结果，如果之后又希望直接使用这个表执行SQL，又该怎么做呢?
  **由于newTable是一个Table对象，并没有在表环境中注册;所以我们还需要将这个中间结果表注册到环境中，才能在SQL中使用**

  ```
  tableEnv. createTemporaryvliew ("NewTable", newTable) ;
  ```
  **我们发现，这里的注册其实是创建了一个“虚拟表”(Virtual Table)**。这个概念与SQL语法中的视图(View)非常类似，所以调用的方法也叫作创建“虛拟视图”(`createTemporaryView`)。视图之所以是“虚拟”的，是因为我们并不会直接保存这个表的内容，并没有“实体”;只是在用到这张表的时候，会将它对应的查询语句嵌入到SQL中。

  **虚拟表(Virtual Table)为我们使用Table API提供了 思路。**

##### 表的查询

创建好了表，接下来就对应着表进行查询转换了。 对一个表的查询(Query)操作，就对应着数据的转换(Transform)处理。

Flink为我们提供了两种查询方式: `SQL`和`Table API`。

- **执行SQL 查询**

  基于表执行SQL语句，是我们最为熟悉的查询方式。**Flink 基于Apache Calcite 来提供对SQL的支持，Calcite 是一个为不同的计算平台提供标准SQL查询的底层工具，很多大数据框架比如`Apache Hive`、`Apache Kylin`中的SQL支持都是通过集成Calcite来实现的**

  ```java
  // 创建表环境
  TableEnvironment tableEnv = ...;
  
  // 创建表
  tableEnv.executeSql("CREATE TABLE EventTable ... WITH ('connector' = ...)");
  
  // 查询用户Alice的点击事件，并提取表中前两个字段
  Table aliceVisitTable = tableEnv.sqlQuery(
  "SELECT user,url " +
  "FROM EventTable " +
  "WHERE user = 'Alice' "
  )
  ```

  目前Flink支持标准SQL中绝大部分用法，并提供了丰富的计算函数。

  例如，我们可以通过`GROUP BY`关键字定义分组聚合，调用`COUNT()`、`SUM()`这样的函数来统计计算:

  ```java
  Table urlCountTable = tableEnv.sqlQuery(
  	"SELECT user,COUNT(url) "+
  	"FROM EventTable "+
  	"GROUP BY user"
  );
  ```

  上面例子中得到的是一个新的Table对象，我们可以再次将它注册为虚拟表继续在SQL中调用。另外，我们也可以直接将查询的结果写入到已经注册的表中，这需要调用表环境的`executeSql()`方法来执行DDL，传入的是INSERT语句:

  ```java
  // 输出到表, 方法一
  TableResult tableRet=table.executeInsert("outputTable");
  
  // 输出到表, 方法二
  tableEnv.executeSql(
  "INSERT INTO OutputTable "+
   "SELECT user,url "+
   "FROM EventTable "+
   "WHERE user='Alice' "
  );
  ```

- **调用Table API 进行查询**

  另外一种查询方式就是调用Table API。这是嵌入在Java和Scala语言内的查询API，核心就是Table接口类,通过一步步链式调用Table的方法,就可以定义出所有的查询转换操作。每一步方法调用的返回结果，都是一个Table。

  **由于TableAPI是基于Table的Java实例进行调用的，因此我们首先要得到表的Java对象。基于环境中已注册的表，可以通过表环境的`from()`方法非常容易地得到一个Table对象:**

  ```
  Table eventTable = tableEnv. from ("EventTable") ;
  ```
  **传入的参数就是注册好的表名。注意这里`eventTable`是一个Table对象，而EventTable是在环境中注册的表名**

  得到Table对象之后，就可以调用API进行各种转换操作了，得到的是一个新的Table 对象:

  ```
  Table maryCl ickTable = eventTable. where($ ("user").isEqual ("Alice")).select ($("url")， $ ("user"));
  ```
  这里每个方法的参数都是一个“表达式”(Expression)， 用方法调用的形式直观地说明了想要表达的内容; `$`符号用来指定表中的一一个字段。上面的代码和直接执行SQL是等效的。

  TableAPI是嵌入编程语言中的DSL,SQL中的很多特性和功能必须要有对应的实现才可以使用，因此跟直接写SQL比起来肯定就要麻烦一些。目前Table API支持的功能相对更少，可以预见未来Flink社区也会以扩展SQL为主，为大家提供更加通用的接口方式;所以我们接下来也会以介绍SQL为主，简略地提及Table API。

  ```java
          // 方法三 基于blink版本planner进行批处理
          // EnvironmentSettings setting3 = EnvironmentSettings.newInstance().inStreamingMode().useBlinkPlanner().build();
          // TableEnvironment tableEnv = TableEnvironment.create(setting3);
  
          // 方法一
          StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
          env.setParallelism(1);
  
          StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);
  
          // 2. 创建表
          String createDDL = "CREATE TABLE  clickTable (" +
                  " user_name STRING, " +
                  " url STRING, " +
                  " ts BIGINT " +
                  ")  WITH (" +
                  " 'connector' = 'filesystem'," +
                  " 'path' = 'input/clicks.txt'," +
                  " 'format' = 'csv' " + // 定义分隔符
                  ")";
          tableEnv.executeSql(createDDL);
          // 3. 创建一张用于输出的表
          String createOutDDL = "CREATE TABLE outTable (" +
                  "`user_name` STRING, " +
                  "`cnt` BIGINT NOT NULL " +
                  ") WITH (" +
                  " 'connector' = 'filesystem'," +
                  " 'path' = 'output/a.txt'," +
                  " 'format' = 'csv' " + // 定义分隔符
                  ")";
          tableEnv.executeSql(createOutDDL);
  
          Table queryRet = tableEnv.sqlQuery("SELECT user_name,COUNT(url) as cnt FROM clickTable GROUP BY user_name");
          tableEnv.sqlQuery("SELECT user_name,COUNT(url) as cnt FROM clickTable GROUP BY user_name").printSchema();
  
          queryRet.executeInsert("outTable");
  
  // 结果输出:
  Mary,2
  Alice,1
  Bob,6
  ```
  
  这里可能会报错: 
  ```java
  doesn't support consuming update changes which is produced by node GroupAggregate(groupBy=[user_name]
  ```
  
  **因为`group by`行为是有update操作的。但是文件系统只能追加，不能更新，所以报错。遇到这个错误可以将`.inStreamingMode()`改成`.inBatchMode()`也就是将流处理变成 批处理，只打印一次结果 就不报错了。**
  
  **如何在控制台打印结果呢?**
  
  ```java
          // 4. 创建一张用于控制台的表
          String createPrintOutDDL = "CREATE TABLE printOutTable (" +
                  "`user_name` STRING, " +
                  "`cnt` BIGINT NOT NULL " +
                  ") WITH (" +
                  " 'connector' = 'print'" + // 这里比较关键
                  ")";
          tableEnv.executeSql(createPrintOutDDL);
  // 结果输出:
  2> +I[Bob, 1] // 第一条是insert
  2> -U[Bob, 1] // 后面是用 delete + update？还是 update set cnt=cnt-1 和 update set cnt=2?
  2> +U[Bob, 2]
  2> -U[Bob, 2]
  2> +U[Bob, 3]
  2> -U[Bob, 3]
  3> +I[Alice, 1]
  1> +I[Mary, 1]
  1> -U[Mary, 1]
  1> +U[Mary, 2]
  2> +U[Bob, 4]
  2> -U[Bob, 4]
  2> +U[Bob, 5]
  2> -U[Bob, 5]
  2> +U[Bob, 6]
  ```

##### 表和流的转换

- **将表(Table)转换成流(DataStream)**

  ````java
          Table aliceVisitTable = tableEnv.sqlQuery(
                  "SELECT user_name,url FROM clickTable WHERE user_name = 'Alice'");
  
  				// 将表转换成数据流
          tableEnv.toDataStream(aliceVisitTable).print();
  ````

- **调用`toChangelogStream()`方法**

  ```java
          Table queryRet = tableEnv.sqlQuery("SELECT user_name,COUNT(url) as cnt FROM clickTable GROUP BY user_name");
  
          tableEnv.toChangelogStream(queryRet).print(); // 更新日志流
  // 打印
  +I[Mary, 1]
  +I[Alice, 1]
  +I[Bob, 1]
  -U[Bob, 1]
  +U[Bob, 2]
  -U[Bob, 2]
  +U[Bob, 3]
  ...
  ```

- **将流DataStream转换成表Table**

  - **调用`fromDataStream`方法**

    ```java
            // 方法一
            StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
            env.setParallelism(1);
    
            StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);
    
            // 读取数据源
            DataStreamSource<Event> eventStream = env.addSource(new ClickLogs());
    
            // 将数据流转换成表
            Table eventTable = tableEnv.fromDataStream(eventStream);
    ```

    **由于流中的数据本身就是定义好的`POJO`类型的Event，所以我们将流转换成表之后，每行数据就对应着一个Event，而表中的列名队员着Event的属性**

    同时我们还可以在`fromDataStream()`方法中增加参数，用来提取哪些属性作为表中的字段名，并可以任意指定位置:

    ```java
    import static org.apache.flink.table.api.Expressions.*;
    
    {
    // 提取Event中的timestamp 和 url 作为表中的列
    Table eventTable2 = tableEnv.fromDataStream(eventStream,$("timestamp"),$("url"));
    }
    ```
  
    还可以用`as`方法进行重命名:
  
    
  
    ```java
    import static org.apache.flink.table.api.Expressions.*;
    
    Table eventTable2 = tableEnv.fromDataStream(eventStream,$("timestamp").as("ts"),$("url"));
    ```
  
  - **调用`createTemporaryView()`方法**
  
    调用`fromDataStream()`方法简单直观，可以直接实现`DataStream`到Table的转换。不过如果我们希望能直接在SQL中引用这张表，就还需要调用`.createTemporaryView()`方法来创建虚拟视图了。
  
    当然对于该场景，也有一种更简洁的调用方式，直接调用`createTemporaryView()`方法创建虚拟表，传入的两个参数，第一个是表名，第二个是`DataStream`，之后传入多个参数指定字段名。
  
    ```java
    import static org.apache.flink.table.api.Expressions.*;
    
    tableEnv.createTemporaryView("EventTable", eventStream, $("timestamp"), $("url"));
    ```
    
    接下来就可以在SQL中引用表 EventTable了。
    
  - **fromChangelogStream()**
  
    表环境还提供了一个方法`fromChangelogStream()`, 可以将一个更新日志流转换成表。这个方法要求流中的数据类型只能是Row，而且每一个数据都需要指定当前行的更新类型(RowKind);所以一般是由连接器帮我们实现的，直接应用比较少见，感兴趣的读者可以查看官网的文档说明。
  
- **支持的数据类型**

  前面示例中的DataStream，流中的数据类型都是定义好的POJO类。如果DataStream中的类型是简单的基本类型，还可以直接转换成表吗?这就涉及了Table中支持的数据类型。

  整体来看，DataStream 中支持的数据类型，Table 中也是都支持的，只不过在进行转换时需要注意一些细节。
  
  - 原子类型
  
    在Flink中，基础数据类型( Integer、Double、 String) 和通用数据类型(也就是不可再拆分的数据类型)统一称作“原子类型”。
    原子类型的DataStream， 转换之后就成了只有一列的Table，列字段(field)的数据类型可以由原子类型推断出。另外，还可以在`fromDataStream()`
    方法里增加参数，用来重新命名列字段。
  
    ```java
    import static org.apache.flink.table.api.Expressions.*;
    
    StreamTableEnvi ronment tableEnv = . . . ;
    DataStream<Long> stream = . . .;
    
    //将数据流转换成动态表，动态表只有一个字段，重命名为myLong
    Table table = tableEnv.fromDataStream (stream,$("myLong"));
    ```

  - Tuple 类型
  
    将原子类型不做重命名时，默认的字段名是`f0`，其实就是将原子类型当做了`Tuple1`来处理。
  
    Table支持Flink中定义的元组类型 Tuple，对应在表中字段名默认就是元组中元素的属性名`f0`、`f1`、`f2`...。所有字段都可以被重新排序，也可以提取其中一部分字段。字段还可以通过表达式`as()`方法来进行重命名。
  
    ```java
    import static org.apache.flink.table.api.Expressions.*;
    
    StreamTableEnvironment tableEnv = . . . ;
    DataStream<Tuple2<Long, Integer>> stream = .. .;
    
    //将数据流转换成只包含f1字段的表
    Table table = tableEnv.fromDataStream (stream, $("f1")) ;
    
    //将数据流转换成包含f0和f1字段的表，在表中f0和f1位置交换
    Table table = tableEnv.fromDataStream(stream, $("f1"), $("f0")) ;
    
    //将f1字段命名为myInt, f0命名为myLong
    Table table = tableEnv. fromDataStream (stream, $("f1") .as ("myInt"),  $("f0").as("myLong")) ;
    ```
  
  - POJO 类型
  
    Flink也支持多种数据类型组合成的“复合类型”，最典型的就是简单Java对象(POJO类型)。由于POJO中已经定义好了可读性强的字段名，这种类型的数据流转换成Table就显得无比顺畅了。
    将POJO类型的DataStream转换成Table,如果不指定字段名称,就会直接使用原始POJO类型中的字段名称。POJO中的字段同样可以被重新排序、提却和重命名，这在之前的例子中已经有过体现。
    ```java
    StreamTableEnvironment tableEnv = . . . ;
    DataStream<Event> stream = . . . ;
    Table table = tableEnv.fromDataStream (stream) ;
    Table table = tableEnv.fromDataStream (stream, $ ("user")) ;
    Table table = tableEnv.fromDataStream(stream, $("user").as ("myUser"), $("url").as("myUrl"));
    ```
  
  - Row类型
  
    - Table中数据的基本组织形式
    - Row类型是一种复合类型，长度固定，无法直接推断出每个字段的类型，所以在使用时必须指明具体的类型信息;
    - 我们在创建table时调用`CREATE`语句就会将所有字段名称和类型指定，这在Flink中被称为表的"模式结构"(Schema)
    - Row类型还附加了一个属性`RowKind`，用于表示当前行在更新操作中的类型。这样，Row就可以用来表示更新日志流(changelog stream)中的数据；
  
    ```java
    DataStream<Row> datastream = env.fromElements(
    Row.ofKird (RowKind. INSERT,"Alice", 12) ,
    Row.ofKind (RowKind. INSERT, "Bob", 5) ,
    Row.ofKind (RowKind. UPDATE BEFORE, "Alice", 12) ,
    Row.ofKind (RowKind. UPDATE AFTER, "Alice", 100));
    
    // 将更新日志流转换为表
    Table table = tableEnv.fromChangelogStream(datastream) ;
    ```

##### 流处理中的表

|                | 关系型表/SQL               | 流处理                                       |
| -------------- | -------------------------- | -------------------------------------------- |
| 处理的数据对象 | 字段元组的有界集合         | 字段元组的无限序列                           |
| 查询Query      | 可以访问到完整的数据输入   | 无法访问到所有数据，必须"持续"等待流式输入   |
| 查询终止条件   | 生成固定大小的结果集后终止 | 永不停止，根据持续收到的数据不断更新查询结果 |

所以感觉SQL更适合针对批处理，与流处理有天然隔阂。下面可以深入探讨流处理中标概念。

##### 动态表和持续查询

- **动态表 Dynamic Tables**

  一般用来做批处理的，面向的是固定的数据集，可以认为是 "静态表";

  动态表完全不同，里面的数据会随时间变化。

  其实动态表的概念，我们在传统的关系型数据库中已经有所接触。数据库中的表，其实是一系列INSERT、UPDATE和DELETE语句执行的结果;在关系型数据库中，我们一般把它称为更新日志流(changelog stream)。如果我们保存了表在某一时刻的快照(snapshot)， 那么接下来只要读取更新日志流，就可以得到表之后的变化过程和最终结果了。在很多高级关系型数据库(比如Oracle、 DB2)中都有“物化视图”(Materialized Views)的概念，可以用来缓存SQL查询的结果;它的更新其实就是不停地处理更新日志流的过程。

- **持续查询 Continuous Query**

  动态表的查询永不停止，随着数据量的到来持续执行;

  <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221221070550904.png" alt="image-20221221070550904" style="zoom:50%;" />
  
- **更新(Update)查询**

  ```java
  Table urlCountTable = tableEnv.sqlQuery("SELECT user,COUNT(url) as cnt FROM EventTable GROUP BY user")
  ```
  
  当原始表不停插入新的数据时，查询得到的`urlCountTable`持续进行更改。由于count数量可能会叠加增长，因此这里的更改操作可以使简单的插入`Insert`、也可以是对之前数据的更新`Update`。换句话说，用来定义结果表的更新日志`changelog`流中，包含了INSERT 和 UPDATE两种操作。这种持续查询被称为更新查询(Update Query)。更新查询得到的结果表如果想要转换成 DataStream，必须调用`toChangelogStream()`方法。
  
- **追加(Append)查询**

  ```java
  Table aliceVisitTable = tableEnv.sqlQuery("SELECT url,user FROM EventTable WHERE user='Cary'");
  ```

  这样的持续查询，被称为追加查询(Append Query)，它定义的结果表的更新日志(changelog)流中只有INSERT操作。追加查询得到的结果表，转换成DataStream调用方法没有限制，直接用toDataStream()，也可以像更新查询一样调用`toChangelogStream()`。

  这样看来，我们似乎可以总结一个规律:只要用到了聚合，在之前的结果上有叠加，就会产生更新操作，就是一个更新查询。但事实上，**更新查询的判断标准是结果表中的数据是否会有UPDATE操作，如果聚合的结果不再改变，那么同样也不是更新查询**

  什么时候聚合的结果会保持不变呢?一个典型的例子就是窗口聚合。
  我们考虑开一个滚动窗口,统计每一小时内所有用户的点击次数，并在结果表中增加一个endT字段，表示当前统计窗口的结束时间。这时结果表的字段定义如下:

  ```json
  [
  	user: VARCHAR, // 用户名
  	endT: TIMESTAMP, // 窗口结束时间
  	cnt: BIGINT //用户访问url次数
  ]
  ```

  如图11-5 所示，与之前的分组聚合一样，当原始动态表不停地插入新的数据时，查询得到的结果result会持续地进行更改。比如时间戳在12:00:00到12:59:59之间的有四条数据，其中Alice三次点击、Bob 一次点击;所以当水位线达到13:00:00时窗口关闭，输出到结果表中的就是新增两条数据`[Alice, 13:00:00, 3]`和`[Bob, 13:00:00, 1]`。同理,当下一小时的窗口关闭时,也会将统计结果追加到result表后面，而不会更新之前的数据。

  <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221222064019913.png" alt="image-20221222064019913" style="zoom:50%;" />

- **查询限制**

  在实际应用中，有些持续查询会因为计算代价太高而受到限制。

  - **状态大小**

    用持续查询做流处理，往往会运行至少几周到几个月;所以持续查询处理的数据总量可能非常大。例如我们之前举的更新查询的例子，需要记录每个用户访问url的次数。如果随着时间的推移用户数越来越大，那么要维护的状态也将逐渐增长，最终可能会耗尽存储空间导致查询失败。
    ```sql
    SELECT user, COUNT (url)  FROM clicks GROUP BY user;
    ```

  - **更新计算**

    对于有些查询来说，更新计算的复杂度可能很高。每来一条新的数据，更新结果的时候可能需要全部重新计算,并且对很多已经输出的行进行更新。一个典型的例子就是`RANK()`函数，它会基于一组数据计算当前值的排名。例如下面的SQL查询，会根据用户最后一次点击的时间为每个用户计算一个排名。当我们收到一个新的数据，用户的最后一次点击时间( lastAction)就会更新，进而所有用户必须重新排序计算一个新的排名。 当一个用户的排名发生改变时，被他超过的那些用户的排名也会改变;这样的更新操作无疑代价巨大，而且还会随着用户的增多越来越严重。
    ```sql
    SELECT user, RANK() OVER (ORDER BY lastAction)
    FROM ( SELECT user, MAX(ts) AS lastAction FROM EventTable GROUP BY user );
    ```
    这样的查询操作，就不太适合作为连续查询在流处理中执行。这里RANK()的使用要配合一个OVER子句，这是所谓的“开窗聚合”，我们会在11.5节展开介绍。

##### 将动态表转换成流

- **仅追加流(Append-only)**

  仅通过插入(insert)更改来修改的动态表，可以直接转换为 仅追加 流。这个流中发出的数据，其实就是动态表中增加的一行

- **撤回流(Retract)**

  撤回流是包含两类消息的流，添加(add) 消息和撤回(retract) 消息。

  **具体的编码规则是: INSERT 插入操作编码为add消息; DELETE 删除操作编码为retract消息;而UPDATE更新操作则编码为被更改行的retract消息，和更新后行(新行)的add消息。这样，我们可以通过编码后的消息指明所有的增删改操作，一个动态表就可以转换为撤回流了**

  可以看到，更新操作对于撤回流来说，对应着两个消息:之前数据的撤回(删除)和新数据的插入。

- **更新插入(Upsert)流**

  更新插入流中只包含两种类型的消息:更新插入(upsert) 消息和删除(delete) 消息。
  所谓的“upsert'其实是“update"和"*insert"的合成词，所以对于更新插入流来说，INSERT 插入操作和UPDATE更新操作,统一被编码为upsert消息;而DELETE删除操作则被编码为delete消息。

  既然更新插入流中不区分插入(insert) 和更新(update)， 那我们自然会想到一个问题:
  如果希望更新一行数据时，怎么保证最后做的操作不是插入呢?
  **这就需要动态表中必须有唯一的键 (key)。通过这个key进行查询，如果存在对应的数据就做更新(update),如果不存在就直接插入(insert)。 这是一个动态表可以转换为更新插入流的必要条件。当然，收到这条流中数据的外部系统，也需要知道这<font style="color:red">唯一的键(key)</font>， 这样才能正确地处理消息**

### 时间属性和窗口

所以所谓的时间属性( time attributes)， 其实就是每个表模式结构(schema) 的一部分。
它可以在创建表的DDL里直接定义为一个字段，也可以在DataStream转换成表时定义。一旦定义了时间属性，它就可以作为一一个普通字段引用，并且可以在基于时间的操作中使用。

时间属性的数据类型为`TIMESTAMP`,它的行为类似于常规时间戳，可以直接访问并且进行计算。

**按照时间语义不同，我们可以把时间属性的定义分成事件时间(event time)和处理时间(processing time)两种情况**

#### 事件时间

我们在实际应用中，最常用的就是事件时间。在事件时间语义下，允许表处理程序根据每个数据中包含的时间戳(也就是事件发生的时间)来生成结果。

**事件时间语义最大的用途就是处理乱序事件或者延迟事件的场景。我们通过设置水位线(watermark)来表示事件时间的进展，而水位线可以根据数据的最大时间戳设置一个延迟时间。这样即使在出现乱序的情况下，对数据的处理也可以获得正确的结果**

为了处理无序事件，并区分流中的迟到事件。Flink需要从事件数据中提取时间戳，并生成水位线，用来推进事件时间的进展。事件时间属性可以在创建表DDL中定义，也可以在数据流和表的转换中定义。

- **在创建表的DDL中定义**

  ```sql
  CREATE TABLE EventTable (
  user STRING,
  url STRING,
  ts TIMESTAMP (3) ,
  WATERMARK FOR ts AS ts - INTERVAL '5' SECOND ) // 5秒的水位线延迟
  WITH ( ... );
  ```

  这里我们把ts字段定义为事件时间属性，而且基于ts设置了5秒的水位线延迟。
  
  **<font style="color:red">如果原始的时间戳就是一个长整型的毫秒数，这时就需要另外顶一个字段来表示时间属性，类型定义为 TIMESTAMP_LTZ 会更方便</font>**
  
  ```sql
  CREATE TABLE events (
  	user STRING,
  	url STRING,
    ts BIGINT,
  	ts_ltz AS TO_TIMESTAMP_LTZ(ts,3),
    WATERMARK FOR ts_ltz AS time_ltz - INTERVAL '5' SECOND
  ) WITH ( ... );
  ```
  
  这里我们另外定义了一个字段`ts_ltz`, 是把长整型的ts转换为`TIMESTAMP_ LTZ`得到的;进而使用WATERMARK语句将它设为事件时间属性，并设置5秒的水位线延迟。
  
  ```java
          // 1. 在创建表的DDL找那个直接定义时间属性
          // 因为clicks.txt 文本中, 第三个字段ts是毫秒的LONG值,所以除以1000先转换成秒; FROM_UNIXTIME()将LONG转换为
          // 字符串格式;
          // select FROM_UNIXTIME(1) => 1970-01-01 08:00:01
          // TO_TIMESTAMP() 是将字符串转换为 timestamp?
          String createDDL = "CREATE TABLE  clickTable (" +
                  " user_name STRING, " +
                  " url STRING, " +
                  " ts BIGINT, " +
                  " et AS TO_TIMESTAMP( FROM_UNIXTIME(ts/1000) ), " +
                  " WATERMARK FOR et AS et - INTERVAL '1' SECOND " +
                  ")  WITH (" +
                  " 'connector' = 'filesystem'," +
                  " 'path' = 'input/clicks.txt'," +
                  " 'format' = 'csv' " + // 定义分隔符
                  ")";
  ```
  
  这里为啥不用 `TO_TIMESTAMP_LTZ()`函数呢？因为这个函数转换出的结果是 UTC时间，`1000`转换出来是负值？
  
- **在数据流转换为表时定义**

  事件时间属性也可以在将`DataStream`转换为表的时候来定义。
  
  我们调用`fromDataStream()`方法创建表时，可以追加参数来定义表中的字段结构;这时可以给某个字段加上`.rowtime()`后缀，就表示将当前字段指定为事件时间属性。**这个字段可以是数据中本不存在、额外追加上去的”逻辑字段“(如下面的水位线生成 和 ts)**，就像之前DDL中定义的第二种情况;也可以是本身固有的字段，那么这个字段就会被事件时间属性所覆盖，类型也会被转换为TIMESTAMP。
  
  不论那种方式，时间属性字段中保存的都是事件的时间戳(`TIMESTAMP`类型)。
  需要注意的是: **这种方式只负责指定时间属性，而时间戳的提取和水位线的生成应该之前就在DataStream上定义好了**。
  **由于DataStream中没有时区概念，因此Flink会将事件时间属性解析成不带时区的TIMESTAMP类型，所有的时间值都被当作UTC标准时间**
  在代码中的定义方式如下:
  ```java
  //方法一: 流中数据类型为二元组Tuple2, 包含两个字段;需要自定义提取时间戳并生成水位线
  DataStream<Tuple2<String, string>> stream = inputstream. assignTimestampsAndWatermarks(.. .);
  
  //声明一个额外的逻辑字段作为事件时间属性
  Table table = tEnv.fromDataStream(stream,$("user"),$("url"),$("ts").rowtime());
  
  //方法二: 流中数据类型为三元组Tuple3,最后一个字段就是事件时间戳
  DataStream<Tuple3<string, String, Long>> stream =inputStream. assignTimest ampsAndWatermarks(...);
  
  //不再声明额外字段，直接用最后一个字段作为事件时间属性
  Table table = tEnv.fromDataStream(stream, $("user"), $("url"),$("ts").rowtime());
  ```
  
  示例:
  
  ```java
  import static org.apache.flink.table.api.Expressions.*;
          
  // 2. 在 stream 转换成 Table时定义时间属性
          SingleOutputStreamOperator<Event> stream = env.addSource(new ClickLogs())
                  .assignTimestampsAndWatermarks(WatermarkStrategy.<Event>forBoundedOutOfOrderness(Duration.ZERO)
                          .withTimestampAssigner(new SerializableTimestampAssigner<Event>() {
                              @Override
                              public long extractTimestamp(Event element, long recordTimestamp) {
                                  return element.timestamp;
                              }
                          }));
          Table clickTable = tableEnv.fromDataStream(stream, $("user"), $("url"), $("timestamp").as("ts"),
                  $("et").rowtime());
          clickTable.printSchema();
    // 输出
  (
    `user` STRING,
    `url` STRING,
    `ts` BIGINT,
    `et` TIMESTAMP(3) *ROWTIME*
  )
  ```

#### 处理时间

相比之下处理时间就比较简单了，它就是我们的系统时间，使用时不需要提取时间戳(timestamp)和生成水位线(watermark)。因此在定义处理时间属性时，必须要额外声明一个字段，专门用来保存当前的处理时间。

类似地，处理时间属性的定义也有两种方式:创建表DDL中定义，或者在数据流转换成表时定义。

- **创建表的DDL中定义**

  在创建表的DDL(CREATE TABLE语句)中，可以增加一个额外的字段，通过调用系统内置的`PROCTIME()`函数来指定当前的处理时间属性，返回的类型是`TIMESTAMP_LTZ`

  ```sql
  CREATE TABLE EventTable(
  user STRING,
  url STRING,
  ts AS PROCTIME() // AS就是以后面的计算结果来做原本不存在列的值
  ) WITH (
  ...
  );
  ```

- **在数据流转换为表时定义**

  由于处理时间是系统时间，原始数据中并没有这个字段，所以处理时间属性一定不能定义在一个已有字段上，只能定义在表结构所有字段的最后，作为额外的逻辑字段出现:

  ```java
  DataStream<Tuple2<String,String>> stream = ...;
  
  // 声明一个额外的字段作为处理时间属性字段
  Table table = tEnv.fromDataStream(stream, $("user"), $("url"),$("ts").proctime());
  ```

#### 窗口(Window)

有了时间属性，接下来就可以定义窗口进行计算了。我们知道，窗口可以将无界流切割成大小有限的“桶”(bucket)来做计算,通过截取有限数据集来处理无限的流数据。在DataStreamAPI中提供了对不同类型的窗口进行定义和处理的接口，而在Table API和SQL中，类似的功能也都可以实现。

- **分组窗口(Group window,老版本)**

  在Flink 1.12之前的版本中，Table API和SQL提供了一组“分组窗口”(Group Window)函数，常用的时间窗口如滚动窗口、滑动窗口、会话窗口都有对应的实现;具体在SQL中就是调用`TUMBLE()`、`HOP()`、`SESSION()`, 传入时间属性字段、窗口大小等参数就可以了。
  以滚动窗口为例: ` TUMBLE (ts,INTERVAL '1' HOUR)`

  这里的ts是定义好的时间属性字段，窗口大小用“时间间隔`INTERVAL`来定义。
  在进行窗口计算时，分组窗口是将窗口本身当作一个字段对数据进行分组的，可以对组内的数据进行聚合。基本使用方式如下:

  ```sql
  Table result = tableEnv.sqlQuery (
  	"SELECT " +
  	"user, " +
  	"TUMBLE_END(ts, INTERVAL '1' HOUR) as endT, " + "ÇOUNT (url) AS cnt " +
  	" FROM EventTable " +
  	"GROUP BY " +   //使用窗口和用户名进行分组
  	"user, " +
  	"TUMBLE(ts, INTERVAL '1' HOUR)" // 定义一小时滚动窗口
  );
  ```

  这里定义了1小时的滚动窗口，将窗口和用户user一起作为分组的字段。用聚合函数`COUNT()`对分组数据的个数进行聚合统计，并将结果字段命名为`cnt`。**用`TUMBLE_END`函数获取滚动窗口的结束时间，重命名为`endT`提取出来**

  分组窗口功能比较有限，只支持窗口聚合，所以已经处于弃用(deprecated)状态。

- **窗口表值函数(Windowing TVFs,新版本)**

  从1.13 版本开始，Flink开始使用窗口表值函数( Windowing table-valued functions,Windowing TVFs)来定义窗口。窗口表值函数是Flink定义的多态表函数(PTF)， 可以将表进行扩展后返回。表函数(table function) 可以看作是返回一个表的函数，后续会介绍。

  **目前Flink提供了以下几个TVF:**

  - 滚动窗口(Tumbling Windows)
  - 滑动窗口(Hop Windows, 跳跃窗口)
  - 累积窗口(Cumulate Windows)
  - 会话窗口(Session Windows, 目前尚未完全支持)

  窗口表值函数可以完全替代传统的分组窗口函数。窗口TVF更符合SQL标准，性能得到了优化，拥有更强大的功能;可以支持基于窗口的复杂计算,例如窗口Top-N.窗口联结(window join)等等。当然，目前窗口TVF的功能还不完善，会话窗口和很多高级功能还不支持，不过正在快速地更新完善。可以预见在未来的版本中，窗口TVF将越来越强大，将会是窗口处理的唯一入口。

  在窗口TVF的返回值中，除去原始表中的所有列，还增加了用来描述窗口的额外3个列: “窗口起始点”( `window_ start`)、 “窗口结束点" (`window_ end`)、 “窗口时间”(`window_ time`)。起始点和结束点比较好理解，这里的“窗口时间”指的是窗口中的时间属性，它的值等于`window_ end- 1ms`，所以相当于是窗口中能够包含数据的最大时间戳。

  - 滚动窗口(TUMBLE)

    ```sql
    TUMBLE (TABLE EventTable, DESCRIPTOR(ts) ，INTERVAL '1' HOUR)
    ```
    这里基于时间字段ts,对表EventTable中的数据开了大小为1小时的滚动窗口。窗口会将表中的每一行数据，按照它们ts的值分配到一个指定的窗口中。

  - 滑动窗口(HOP )

    ```sql
    HOP (TABLE EventTable, DESCRIPTOR(ts)，INTERVAL '5' MINUTES, INTERVAL '1' HOURS) )
    ```
    这里我们基于时间属性ts,在表EventTable.上创建了大小为1小时的滑动窗口，每5分钟滑动一次。需要注意的是，紧跟在时间属性字段后面的第三个参数是步长(slide)， 第四个参数才是窗口大小(size)。

  - 累积窗口 (CUMUCATE)

    滚动窗口和滑动窗口，都可以用来计算大多数周期性的同级指标。不过在实际应用中还会遇到这样一类需求: **我们的统计周期可能较长，因此希望中间每隔一段时间就输出一次当前的统计值；与滑动窗口不同，在一个统计周期内，我们会多次输出统计值，他们应该是不断叠加累积的。**

    例如，我们按天来统计网站的PV (Page View， 页面浏览量)，如果用1天的滚动窗口，那需要到每天24点才会计算一次，输出频率太低;如果用滑动窗口，计算频率可以更高，但统计的就变成了“过去24小时的PV，比如是昨天6点到今天6点的PV"，而不是我们想要的今天的PV。所以我们真正希望的是，还是按照自然日统计每天的PV，不过需要每隔1小时就输出一次当天到目前为止的PV值。这种特殊的窗口就叫作"累积窗口"(Cumulate Window)。

    累积窗口 前面我们是通过 状态 + ProcessFunction() + 触发器完成的。

    累积创建的两个核心参数: 最大窗口长度(max window size)  和 累积步长(step)。最大窗口长度就是我们所说的 "同级周期"，最终目的就是同级这段时间内的数据。

    ```
    CUMULATE (TABLE EventTable, DESCRIPTOR(ts) , INTERVAL '1' HOURS, INTERVAL '1' DAYS) )
    ```

    **这里我们基于时间属性ts,在表EventTable上定义了一个统计周期为1天、累积步长为1小时的累积窗口。注意第三个参数为步长step，第四个参数则是最大窗口长度**

### 聚合(Aggregation)查询

在SQL中，一个很常见的功能就是对某一列的多条数据做一个合并统计，得到一个或多个结果值;比如求和、最大最小值、平均值等等，这种操作叫作聚合(Aggregation) 查询。

Flink中的SQL是流处理与标准SQL结合的产物，**所以聚合查询也可以分成两种:流处理中特有的聚合(主要指窗口聚合)，以及SQL原生的聚合查询方式**

#### 分组聚合

常见的聚合函数`SUM()`、`MAX()`、`MIN()`、`AVG()`以及`COUNT()`

另外，在持续查询的过程中，由于用于分组的key可能会不断增加，因此计算结果所需要维护的状态也会持续增长。为了防止状态无限增长耗尽资源，FlinkTableAPI和SQL可以在表环境中配置状态的生存时间(TTL):
```java
TableEnvi ronment tableEnv = . . .
//获取表环境的配置
TableConfig tableConfig = tableEnv.getConfig();
//配置状态保持时间
tableConfig. setIdleStateRetention (Duration. ofMinutes(60)) ;
```
或者也可以直接设置配置项tablexec.state.ttl:
```java
TableEnvi ronment tableEnv = ...;
Configuration configuration = tableEnv . getConfig () . getConfiguration();
configuration. setstring ("table.exec.state.ttl", "60 min");
```

这两种方式是等效的。需要注意的是，配置TTL可能导致统计结果不准确。

#### 窗口聚合

- 与分组聚合类似，窗口聚合也需要调用`SUM()`、`MAX()`、`MIN()`、`COUNT`一类聚合函数；

- 通过`GROUP BY`来指定分组字段，在Flink 1.12之前，是直接将窗口自身作为分组key放在`GROUP BY`之后的，所以也叫 "分组窗口聚合"，见上面;

- 而1.13版本后，开始使用了"窗口表值函数"(Windowing TVF)，窗口本身返回的是一个表，所以窗口会出现在FROM后面,GROUP BY后来则是窗口新增的字段`window_start`和`windwo_end`

  ```sql
  Table result = tableEnv. sqlQuery (
  	"SELECT " +
  	"user, window_end AS endT, COUNT (url) AS cnt  " +
  	" FROM TABLE( TUMBLE( TABLE EventTable, DESCRIPTOR(ts), INTERVAL '1' HOUR) )  " +
  	"GROUP BY user, window_start, window_end"
  );
  ```

  这里以ts作为时间属性字段、基于 EventTable定义了一小时的滚动窗口，希望统计出每个小时每个用户点击url的次数。用来分组的是user字段 和 表示窗口的`window_start`和`window_end`字段。而`TUMBLE()`是表值函数，所以得到的是一个表`Table`，我们的聚合查询就是在这个Table中进行的。

  示例 滚动窗口:

  ```java
  /* clicks.txt */
  Mary,./home,1000
  Alice,./cart,2000
  Bob,./prod?id=100,3000
  Bob,./cart,4000
  Bob,. /home,5000
  Mary,. /home,6000
  Bob,./cart,7000
  Bob,. /home,8000
  Bob,./prod?id=10,9000
  Alice,./cart,12000
  Alice,./home,13000
  Bob,. /home,18000
  Bob,. /home,15000
  
  /* TableTimeAndWindow类 */
          String createDDL = "CREATE TABLE clickTable (" +
                  " user_name STRING, " +
                  " url STRING, " +
                  " ts BIGINT, " +
                  " et AS TO_TIMESTAMP( FROM_UNIXTIME(ts/1000) ), " +
                  " WATERMARK FOR et AS et - INTERVAL '1' SECOND " +
                  ") WITH (" +
                  " 'connector' = 'filesystem'," +
                  " 'path' = 'input/clicks.txt'," +
                  " 'format' = 'csv' " + // 定义分隔符
                  ")";
          tableEnv.executeSql(createDDL);
  
          // 3. 窗口聚合
          // 3.1 滚动窗口
          Table tumbleWindowRetTable = tableEnv.sqlQuery("select user_name, COUNT(1) as cnt, window_end as endT " +
                  " from TABLE( TUMBLE( TABLE clickTable, DESCRIPTOR(et), INTERVAL '10' SECOND)) " +
                  " GROUP BY user_name,window_start,window_end");
  
          tableEnv.toDataStream(tumbleWindowRetTable).print("tumble window:");
          env.execute();
          
    // 输出:
  tumble window:> +I[Mary, 2, 1970-01-01T00:00:10]
  tumble window:> +I[Alice, 1, 1970-01-01T00:00:10]
  tumble window:> +I[Bob, 6, 1970-01-01T00:00:10]
  tumble window:> +I[Alice, 2, 1970-01-01T00:00:20]
  tumble window:> +I[Bob, 2, 1970-01-01T00:00:20]
  ```

  示例 滑动窗口:

  ```java
          // 3.2 滚动窗口,窗口长度为10秒,滑动步长为5秒
          Table hopWindowRetTable = tableEnv.sqlQuery("select user_name, COUNT(1) as cnt, window_end as endT " +
                  " from TABLE( HOP( TABLE clickTable, DESCRIPTOR(et), INTERVAL '5' SECOND, INTERVAL '10' SECOND)) " +
                  " GROUP BY user_name,window_start,window_end");
  
          tableEnv.toDataStream(hopWindowRetTable).print("hop window:");
          
          
  // 输出:
  hop window:> +I[Mary, 1, 1970-01-01T00:00:05]
  hop window:> +I[Alice, 1, 1970-01-01T00:00:05]
  hop window:> +I[Bob, 2, 1970-01-01T00:00:05]
  hop window:> +I[Alice, 1, 1970-01-01T00:00:10]
  hop window:> +I[Mary, 2, 1970-01-01T00:00:10]
  hop window:> +I[Bob, 6, 1970-01-01T00:00:10]
  hop window:> +I[Mary, 1, 1970-01-01T00:00:15]
  hop window:> +I[Alice, 2, 1970-01-01T00:00:15]
  hop window:> +I[Bob, 4, 1970-01-01T00:00:15]
  hop window:> +I[Alice, 2, 1970-01-01T00:00:20]
  hop window:> +I[Bob, 2, 1970-01-01T00:00:20]
  hop window:> +I[Bob, 2, 1970-01-01T00:00:25]
  ```

  示例 累积窗口:

  ```java
          // 3.3 累积窗口,最大窗口值30秒,每隔多5秒输出一次结果
          Table cumulateWindowRetTable = tableEnv.sqlQuery("select user_name, COUNT(1) as cnt, window_end as endT " +
                  " from TABLE( CUMULATE( TABLE clickTable, DESCRIPTOR(et), INTERVAL '5' SECOND, INTERVAL '30' SECOND)) "
                  +
                  " GROUP BY user_name,window_start,window_end");
  
          tableEnv.toDataStream(cumulateWindowRetTable).print("cumulate window:");
   
  // 输出:
  cumulate window:> +I[Mary, 1, 1970-01-01T00:00:05]
  cumulate window:> +I[Alice, 1, 1970-01-01T00:00:05]
  cumulate window:> +I[Bob, 2, 1970-01-01T00:00:05]
  cumulate window:> +I[Alice, 1, 1970-01-01T00:00:10]
  cumulate window:> +I[Mary, 2, 1970-01-01T00:00:10]
  cumulate window:> +I[Bob, 6, 1970-01-01T00:00:10]
  cumulate window:> +I[Mary, 2, 1970-01-01T00:00:15]
  cumulate window:> +I[Alice, 3, 1970-01-01T00:00:15]
  cumulate window:> +I[Bob, 6, 1970-01-01T00:00:15]
  cumulate window:> +I[Alice, 3, 1970-01-01T00:00:20]
  cumulate window:> +I[Mary, 2, 1970-01-01T00:00:20]
  cumulate window:> +I[Bob, 8, 1970-01-01T00:00:20]
  cumulate window:> +I[Mary, 2, 1970-01-01T00:00:25]
  cumulate window:> +I[Bob, 8, 1970-01-01T00:00:25]
  cumulate window:> +I[Alice, 3, 1970-01-01T00:00:25]
  cumulate window:> +I[Bob, 8, 1970-01-01T00:00:30]
  cumulate window:> +I[Alice, 3, 1970-01-01T00:00:30]
  cumulate window:> +I[Mary, 2, 1970-01-01T00:00:30]
  ```

#### 开窗(Over)聚合

在标准SQL中还有另外一 类比较特殊的聚合方式，可以针对每一行计算一个聚合值。比如说，我们可以以每一行数据为基准， **计算<font style="color:red">它之前1小时内所有数据的平均值;也可以计算它之前10个数的平均值</font>, 就好像是在每一行上打开了一扇窗户、收集数据进行统计一样，这就是所谓的“开窗函数”**

开窗函数的聚合与之前两种聚合有本质的不同:分组聚合、窗口TVF聚合都是“多对一”的关系，将数据分组之后每组只会得到一个聚合结果;而**开窗函数是对每行都要做一次开窗聚合，因此聚合之后表中的行数不会有任何减少**，是一个“多对多”的关系。

与标准SQL中一致，Flirk SQL中的开窗函数也是通过OVER子句来实现的，所以有时开窗聚合也叫作“OVER聚合”(Over Aggregation)。基本语法如下:

```sql
SELECT
	<聚合函数> OVER (
	[PARTITION BY <字段1>[，<字段2>，...]]
	ORDER BY <时间属性字段>
	<开窗范围>),
	...
FROM ...
```

这里OVER关键字前面是一个聚合函数，它会应用在后面OVER定义的窗口上。在OVER子句中主要由以下几部分:

- **PARTITION BY(可选)**: 用来指定分区的键(key)，类似于`GROUP BY`分组，可选的；

- **ORDER BY:** OVER窗口是基于当前行扩展出的一段数据范围，选择的标准可以基于时间也可以基于数据量。不论那种定义，数据都应该是以某种顺序排列好的;而表中的数据本身是无序的。**所以在OVER子句中必须用ORDER BY明确地指出数据基于那个字段排序。在Flink的流处理中, 目前只支持按照时间属性的升序排列，所以这里ORDER BY后面的字段必须是定义好的时间属性**

- **开窗范围:** 由`BETWEEN <下界> AND <上界>`来定义，也就是下界到上界的范围。目前支持的上界只能是`CURRENT ROW`，也就是定义一个 从之前某一行到当前行的范围，所以一般形式是

  ```sql
  BETWEEN ... PRECEDING AND CURRENT ROW
  ```

  开窗范围是时间范围(RANGE intervals) 还是 行间隔(ROW intervals)

  - **范围间隔**

    范围间隔以RANGE为前缀，就是基于`ORDER BY`指定的时间字段去选取一个范围，一般就是当前行时间戳之前的一段时间。例如开窗范围选择当前行之前1小时的数据:

    ```sql
    RANGE BETWEEN INTERVAL '1' HOUR PRECEDING AND CURRENT ROW
    ```

  - **行间隔**: 当前行之前的5行数据。最终聚合会包括当前行，所以一共是6条数据

    ```sql
    ROWS BETWEEN 5 PRECEDING AND CURRENT ROW
    ```

  下面是一个具体示例:

  ```java
          // 4. 开窗聚合(Over)
          Table overWindowRetTable = tableEnv.sqlQuery("select user_name, avg(ts) " +
                  " OVER( PARTITION BY user_name ORDER BY et ROWS BETWEEN  3 PRECEDING AND CURRENT ROW) AS avg_ts " +
                  " FROM clickTable");
  
          tableEnv.toDataStream(overWindowRetTable).print("over window:");
          
  // 输出:
  over window:> +I[Mary, 1000]
  over window:> +I[Alice, 2000]
  over window:> +I[Bob, 3000]
  over window:> +I[Bob, 3500]
  over window:> +I[Bob, 4000]
  over window:> +I[Mary, 3500]
  over window:> +I[Bob, 4750]
  over window:> +I[Bob, 6000]
  over window:> +I[Bob, 7250]
  over window:> +I[Alice, 7000]
  over window:> +I[Alice, 9000]
  over window:> +I[Bob, 9750]
  over window:> +I[Bob, 12500]
  ```

  也可以在WINDOW子句中在SELECT外部单独定义一个OVER窗口:

  ```sql
  SELECT user,ts, COUNT(url) OVER w AS cnt, MAX(CHAR_LENGTH(url)) OVER w as max_url
  FROM EventTable
  WINDOW w AS (PARTITION BY user ORDER BY ts ROWS BETWEEN  2 PRECEDING AND CURRENT ROW)
  ```
  

#### TOP N 应用示例

- **普通的Top N**

  在Flink SQL中，是通过OVER聚合和一个条件筛选来实现TopN的。具体来说，是通过将一个特殊的聚合函数`ROW_NUMBER()`应用到OVER窗口上，统计出每一行排序后的行号，作为一个字段提取出来;然后再用WHERE子句筛选行号小于等于N的那些行返回。

  基本语法如下:

  ```sql
  SELECT ...
  FROM (
  	SELECT ..., ROW_NUMBER() OVER ( [PARTITION BY <字段1>[,<字段1>...]] ORDER BY <排序字段1> [asc/desc][,排序字段2 asc/desc,...] )
    AS row_num
    FROM ...
  ) WHERE row_num <= N [AND 其他条件];
  ```

  上面的写法是固定的，固定的嵌套查询格式。其中`row_num`是内部子查询聚合的结果，不能在内部作为筛选条件，只能放在外层的WHERE 子语句中。

  下面是统计每个用户的访问事件中，按照字符串长度排序的前两个:

  ```java
          Table topNRetTable = tableEnv.sqlQuery("select user_name,url,ts,row_num " +
                  "FROM (" +
                  " SELECT *, ROW_NUMBER() OVER (PARTITION BY user_name ORDER BY CHAR_LENGTH(url) desc) AS row_num " +
                  " FROM clickTable)" +
                  " WHERE row_num <=2");
          tableEnv.toChangelogStream(topNRetTable).print("topN:");
   // 输出:
   topN:> +I[Mary, ./home, 1000, 1]
  topN:> +I[Alice, ./cart, 2000, 1]
  topN:> +I[Bob, ./prod?id=100, 3000, 1]
  topN:> +I[Bob, ./cart, 4000, 2]
  topN:> -U[Bob, ./cart, 4000, 2]
  topN:> +U[Bob, . /home, 5000, 2]
  topN:> -U[Mary, ./home, 1000, 1]
  topN:> +U[Mary, . /home, 6000, 1]
  topN:> +I[Mary, ./home, 1000, 2]
  topN:> -U[Bob, . /home, 5000, 2]
  topN:> +U[Bob, ./prod?id=10, 9000, 2]
  topN:> +I[Alice, ./cart, 12000, 2]
  ```

  统计访问次数前2的user:

  - 第一个子select，获得每个 user_name 和 访问次数;
  - 第二个子select, 通过 `ROW_NUMBER()`对每个用户的 访问次数进行排序
  - 最外层的select，做 limit N操作

  ```java
          // 选取浏览页面最多的两个用户
          Table topNUserTable = tableEnv.sqlQuery("select user_name,cnt,row_num " +
                  "FROM (" +
                  "SELECT *,ROW_NUMBER() OVER ( ORDER BY cnt DESC) AS row_num " +
                  " FROM ( SELECT user_name, COUNT(url) as cnt FROM clickTable GROUP BY user_name) " +
                  ") WHERE row_num<=2");
  				// 因为结果随着来的数据不断变化, 直接打印到控制台得用 toChangelogStream
          tableEnv.toChangelogStream(topNUserTable).print("topN user:");
          // 输出:
   topN user:> +I[Mary, 1, 1]
  topN user:> +I[Alice, 1, 2]
  topN user:> -U[Mary, 1, 1]
  topN user:> +U[Bob, 2, 1]
  topN user:> -U[Alice, 1, 2]
  topN user:> +U[Mary, 1, 2]
  topN user:> -U[Bob, 2, 1]
  topN user:> +U[Bob, 3, 1]
  topN user:> -U[Mary, 1, 2]
  topN user:> +U[Mary, 2, 2]
  topN user:> -U[Bob, 3, 1]
  topN user:> +U[Bob, 4, 1]
  topN user:> -U[Bob, 4, 1]
  topN user:> +U[Bob, 5, 1]
  topN user:> -U[Bob, 5, 1]
  topN user:> +U[Bob, 6, 1]
  topN user:> -U[Mary, 2, 2]
  topN user:> +U[Alice, 3, 2]
  topN user:> -U[Bob, 6, 1]
  topN user:> +U[Bob, 7, 1]
  topN user:> -U[Bob, 7, 1]
  topN user:> +U[Bob, 8, 1]
  ```

- **窗口TOP N**

  除了直接对数据进行TopN的选取，我们也可以针对窗口来做TopN。

  例如电商行业，实际应用中往往有这样的需求:统计一段时间内的热门商品。这就需要先开窗口，在窗口中统计每个商品的点击量;然后将统计数据收集起来，按窗口进行分组，并按点击量大小降序排序，选取前N个作为结果返回。

  我们已经知道，TopN聚合本质上是一个表聚合函数，这和窗口表值函数(TVF) 有天然的联系。
  尽管如此，想要基于窗口TVF实现一个通用的Top N聚合函数还是比较麻烦的，目前FlinkSQL尚不支持。不过我们同样可以借鉴之前的思路,使用OVER窗口统计行号来实现。具体来说，**可以先做一个窗口聚合，将窗口信息`window_ start`、`window_ end`连同每个商品的点击量一并返回，这样就得到了聚合的结果表，包含了窗口信息、商品和统计的点击量。接下来就可以像一般的Top N那样定义OVER窗口了，按窗口分组，按点击量排序，用`ROW_ NUMBER()`统计行号并筛选前N行就可以得到结果。所以窗口Top N的实现就是窗口聚合与OVER聚合的结合使用**

  ```sql
          String subQuery = "SELECT user_name,COUNT(url) AS cnt, window_start,window_end " +
                  " FROM TABLE( " +
                  " TUMBLE(TABLE clickTable, DESCRIPTOR(et), INTERVAL '10' SECOND)" +
                  " ) " +
                  " GROUP BY user_name,window_start,window_end";
  
          // 窗口top N,统计一段时间内的活跃用户
          Table topNUserByWindow = tableEnv.sqlQuery("select user_name,cnt,row_num,window_end " +
                  "FROM (" +
                  "SELECT *,ROW_NUMBER() OVER ( PARTITION BY window_start,window_end ORDER BY cnt DESC) AS row_num " + // partition中必须是window_start,window_end
                  " FROM ( " + subQuery + " ) " +
                  ") WHERE row_num<=2");
          // 一个窗口得到的结果是固定的所有直接用 toDataStream(),无需使用toChangelogStream
          tableEnv.toDataStream(topNUserByWindow).print("window topN user:");
          
  // 输出:
  window topN user:> +I[Bob, 6, 1, 1970-01-01T00:00:10]
  window topN user:> +I[Mary, 2, 2, 1970-01-01T00:00:10]
  window topN user:> +I[Alice, 2, 1, 1970-01-01T00:00:20]
  window topN user:> +I[Bob, 2, 2, 1970-01-01T00:00:20]
  ```

### **Join 查询**

#### 常规join查询

常规联结( Regular Join)是SQL中原生定义的Join方式，是最通用的一类联结操作。它的具体语法与标准SQL的联结完全相同，通过关键字JOIN来联结两个表，后面用关键字ON来指明联结条件。

**按照习惯，我们一般以“左侧"和“右侧”来区分联结操作的两个表。在两个动态表的联结中，任何一侧表的插入(INSERT) 或更改(UPDATE)操作都会让
联结的结果表发生改变**。例如，如果左侧有新数据到来，那么它会与右侧表中所有之前的数据进行联结合并，右侧表之后到来的新数据也会与这条数据连接合并。所以，常规联结查询一般是更新(Update)查询。

**与标准SQL一致，Flink SQL的常规联结也可以分为内联结(INNER JOIN)和外联结(OUTER JOIN)，区别在于结果中是否包含不符合联结条件的行。目前仅支持“等值条件”作为联结条件，也就是关键字ON后面必须是判断两表中字段相等的逻辑表达式**

- **等值内连接 (INNER Equi-JOIN)**

  ```sql
  SELECT * FROM Order INNER JOIN Product ON Order.product_id = Product.id;
  ```

- **等值外连接(OUTER Equi-JOIN)**

  外连接除了返回符合连接条件的笛卡尔积外，还会降某一侧表中找不到任何匹配的行业单独返回。Flink SQL支持左外(LEFT JOIN)、右外(RIGHT JOIN)和全外(FULL OUTER JOIN)，分别表示会将左侧表、右侧表以及双侧表中没有任何匹配的行返回。

  如 订单表中未必包含了商品表中所有ID，为了将那些没有任何订单的商品信息也查询出来，我们可以使用RIGHT JOIN。

  ```sql
  SELECT *
  FROM Order
  RIGHT JOIN Product ON Order.product_id = Product.id;
  ```

#### 间隔连接查询

我们曾经学习过DataStream API中的双流Join,包括窗口联结( window join)和间隔联结(intervaljoin)。

两条流的Join就对应着SQL中两个表的Join,这是流处理中特有的联结方式。目前Flink SQL还不支持窗口联结，而间隔联结则已经实现。
间隔联结(IntervalJoin)返回的，同样是符合约束条件的两条中数据的笛卡尔积。只不过这里的“约束条件"除了常规的联结条件外，还多了一个时间间隔的限制。具体语法有以下要点:

- **两表的连接**

  间隔联结不需要用JOIN关键字，直接在FROM后将要联结的两表列出来就可以，用逗号分隔。这与标准SQL中的语法一致，表示-一个“交叉联结”(Cross Join)，会返回两表中所有行的笛卡尔积。.

- **连接条件**

  联结条件用WHERE子句来定义，用一个等值表达式描述。交叉联结之后再用WHERE进行条件筛选，效果跟内联结`INNER JOIN .. ON ..`非常类似。

- **时间间隔限制**

  我们可以在WHERE子句中，联结条件后用AND追加一个时间间隔的限制条件;做法是提取左右两侧表中的时间字段，然后用一个表达式来指明两者需要满足的间隔限制。具体定义方式有下面三种，这里分别用ltime和rtime表示左右表中的时间字段:

  - `ltime = rtime`
  - `ltime >= rtime AND ltime < rtime + INTERVAL '10' MINUTE`
  - `ltime BETWEEN rtime - INTERVAL '10' SECOND AND rtime + INTERVAL '5' SECOND`

  判断两者相等，这是最强的时间约束，要求两表中数据的时间必须完全一致才 能匹配; 一般情况下，我们还是会放宽一些，给出一个间隔。间隔的定义可以用<，<=，>=， >这一类的关系不等式，也可以用`BETWEEN... AND...`这样的表达式。

  例如，我们现在除了订单表Order外，还有一个“发货表"Shipment,要求在收到订单后四个小时内发货。那么我们就可以用一个间隔联结查询,把所有订单与它对应的发货信息连接合并在一起返回。
  ```sql
  SELECT * FROM Order O,Shipment S
  WHERE o.id = s.order_id AND o.order_time BETWEEN s.ship_time - INTERVAL '4' HOUR AND s.ship_time; # 订单时间 在发货时间之前 4个小时
  ```

  **在流处理中，间隔连接查询只支持具有时间属性的"仅追加"(Append Only)**

  那对于有更新操作的表，又怎么办呢?除了间隔联结之外，Flink SQL还支持时间联结(`Temporal Join`)，这主要是针对“版本表" (versioned table)而言的。所谓版本表，就是记录了数据随着时间推移版本变化的表, I可以理解成一个“更新日志”(change log)，它就是具有时间属性、还会进行更新操作的表。当我们联结某个版本表时，并不是把当前的数据连接合并起来就行了，而是希望能够根据数据发生的时间，找到当时的“版本”;这种根据更新时间提取当时的值进行联结的操作，就叫作“时间联结”(Temporal Join)。这部分内容由于涉及版本表的定义，我们就不详细展开了，感兴趣的读者可以查阅官网资料。

### 函数

这里主要介绍 Flink SQL中的函数使用。

Flink SQL中函数分为两类，一类是 SQL中内置的系统函数，直接通过函数名调用，能够实现常用的转换操作，如`COUNT()`、`CHAR_LENGTH()`等；另一类是用户自定义的函数(UDF)，需要在表环境中注册才能使用。

#### 系统函数

- 标量系统函数(scalar Functions)

  所谓的“标量”，是指只有数值大小、没有方向的量;所以标量函数指的就是只对输入数据做转换操作、返回一个值的函数。这里的输入数据对应在表中，一般就是一行数据中1个或多个字段，因此这种操作有点像流处理转换算子中的map。另外，对于一些没有输入参数、直接可以得到唯一结果的函数，也属于标量函数。

  标量函数是最常见、也最简单的一类系统函数，数量非常庞大，很多在标准SQL中也有定义。所以我们这里只对一些常见类型列举部分函数，做一个简单概述，具体应用可以查看官网的完整函断列表。

  - 比较函数(Comparison Functions): `=`、`<>`、`IS NOT NULL`
  - 逻辑函数:`AND`、`OR`、`NOT`
  - 算术函数 
  - 时间函数
    - `DATE string`: 按格式`yyyy-MM-dd`解析字符串string, 返回类型为 SQL Date
    - `TIMESTAMP string`:按格式`yyyy-MM-dd HH:mm:ss[.SSS]`解析，返回类型为SQL timestamp
    - `CURRENT_TIME`:发货本地时区的当前时间，类型为 SQL time
    - `INTERVAL string range`: 返回一个时间间隔。string表示数值；RANGE 可以是 DAY、MINUTE、DAT TO HOUR 等单位，也可以是 YEAR TO MONTH 这样的复合单位，比如 "2年10个月"可以写成:`INTERVAL '2-10' YEAR TO MONTH`
  - ...

- 聚合函数

  - 以表中多行作为输入，提取字段进行聚合操作的函数，将唯一的聚合值作为结果返回；
  - `COUNT(*)`: 返回所有行的数量，统计个数
  - `SUM([ALL|DISTINCT|expression])` 对某个字段进行求和操作，默认情况下省略了关键字ALL。如果指定了DISTINCT，则会对数据进行去重，每个值只叠加一次;
  - `RANK()` 返回当前值在一组值中的排名
  - `ROW_NUMBER()`: 对一组值排序后，返回当前值的行号。与`RANK()`的功能类似；

#### 自定义函数

整体流程:

- (1) 注册函数

  注册函数时需要调用表环境的`createTemporarySystemFunction()`方法， 传入注册的函数名以及UDF类的Class对象:
  ```
  //注册函数
  tableEnv.createTemporarySystemFunction( "MyFunction", MyFunction.class);
  ```
  我们自定义的UDF类叫作MyFunction,它应该是上面四种UDF抽象类中某一个的具体实现;在环境中将它注册为名叫MyFunction的函数。

  这里`create TemporarySystemFunction()`方法的意思是创建了一个“临时系统函数”，所以MyFunction 函数名是全局的，可以当作系统函数来使用;我们也可以用`create TemporaryFunction()`方法，注册的函数就依赖于当前的数据库( database )和目录( catalog)了，所以这就不是系统函数，而是“目录函数”(catalog function)， 它的完整名称应该包括所属的database和catalog。

  一般情况下， 我们直接用create TemporarySystemFunction(方法将UDF注册为系统函数就可以了。

- (2) 使用Table API调用函数

  在 Table API中，需要使用`call()`方法来调用自定义函数:

  ```java
  tableEnv.from("MyTable").select(call("MyFunction", $("myField")));
  ```

  此外，在Table API中也可以不注册函数，直接调用"内联(inline)"的方式调用UDF:

  ```java
  tableEnv.from("MyTable").select(call(SubstringFunction.class,$("myField")));
  ```

- (3) 在SQL 中调用函数

  当我们将函数注册为系统函数后，在SQL中调用就与内置系统函数完全一样了:

  ```java
  tableEnv.sqlQuery("SELECT MyFunction(myField) FROM MyTable");
  ```

##### 标量函数(Scalar Functions)

**自定义标量函数可以把0个、1个或多个标量值转换成一个标量值，它对应的输入是一行数据中的字段，输出则是唯一 的值。所以从输入和输出表中行数据的对应关系看，标量函数是“一对一”的转换**

**想要实现自定义的标量函数，我们需要自定义一个类来继承抽象类`ScalarFunction`， 并实现叫作`eval()`的求值方法。标量函数的行为就取决于求值方法的定义，它必须是公有的(public),而且名字必须是eval。 求值方法eval可以重载多次，任何数据类型都可作为求值方法的参数和返回值类型**

这里需要特别说明的是，`ScalarFunction`抽象类中并没有定义`eval()`方法，所以我们不能直接在代码中重写(override);但Table API的框架底层又要求了求值方法必须名字为eval()。这是TableAPI和SQL目前还显得不够完善的地方，未来的版本应该会有所改进。

`ScalarFunction`以及其它所有的UDF接口，都在`org.aapache.flink.table. functions` 中。

```java
package com.luke.flink.datastreamdemo;

import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.table.api.Table;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
import org.apache.flink.table.functions.ScalarFunction;

public class UdfTest_ScalarFunction {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);

        StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);

        // 读取数据源
        DataStreamSource<Event> eventStream = env.addSource(new ClickLogs());

        String createDDL = "CREATE TABLE clickTable (" +
                " user_name STRING, " +
                " url STRING, " +
                " ts BIGINT, " +
                " et AS TO_TIMESTAMP( FROM_UNIXTIME(ts/1000) ), " +
                " WATERMARK FOR et AS et - INTERVAL '1' SECOND " +
                ") WITH (" +
                " 'connector' = 'filesystem'," +
                " 'path' = 'input/clicks.txt'," +
                " 'format' = 'csv' " + // 定义分隔符
                ")";
        tableEnv.executeSql(createDDL);

        // 2. 注册自定义标量函数
        tableEnv.createTemporaryFunction("MyHash", MyHashFunction.class);
        // 3. 调用UDF进行查询转换
        Table resultTable = tableEnv.sqlQuery("select user_name,MyHash(user_name) from clickTable");
        // 4. 转换成流打印输出
        tableEnv.toDataStream(resultTable).print();

        env.execute();
    }

    public static class MyHashFunction extends ScalarFunction {
        public int eval(String str) {
            return str.hashCode();
        }
    }
}
//输出:
+I[Mary, 2390779]
+I[Alice, 63350368]
+I[Bob, 66965]
+I[Bob, 66965]
+I[Bob, 66965]
+I[Mary, 2390779]
```

##### 表函数(Table Functions)

**跟标量函数一样， 表函数的输入参数也可以是0个、1个或多个标量值;不同的是，它可以返回任意多行数据。“多行数据”事实上就构成了一个表，所以“表函数”可以认为就是返回一个表的函数，这是一个“一对多"的转换关系。之前我们介绍过的窗口TVF，本质上就是表函数**

**类似地，要实现自定义的表函数，需要自定义类来继承抽象类`TableFunction`，内部必须要实现的也是一一个名为`eval`的求值方法。与标量函数不同的是，`TableFunction` 类本身是有一个泛型参数T的,这就是表函数返回数据的类型;而`eval()`方法没有返回类型,内部也没有return语句，是通过调用`collect()`方法来发送想要输出的行数据的。多么熟悉的感觉——回忆一下DataStream API中的FlatMapFunction和ProcessFunction，它们的flatMap和processElement方法也没有返回值，也是通过out.collect()来向下游发送数据的**

我们使用表函数，可以对一行数据得到一个表，这和Hive中的UDTF非常相似。**那对于原先输入的整张表来说，又该得到什么呢?一个简单的想法是，就让输入表中的每一行， 与它转换得到的表进行联结(join)， 然后再拼成一个完整的大表，这就相当于对原来的表进行了扩展**。在Hive的SQL语法中，提供了“侧向视图”( lateral view,也叫横向视图)的功能，可以将表中的一行数据拆分成多行; Flink SQL也有类似的功能，是用LATERAL TABLE语法来实现的。

在SQL中调用表函数，需要使用`LATERAL TABLE(<TableFunction> )`来生成扩展的“侧向表”，然后与原始表进行联结(Join)。 这里的Join操作可以是直接做交叉联结(cross join)，在FROM后用逗号分隔两个表就可以;也可以是以`ON TRUE`为条件的左联结(`LEFT JOIN`)。

下面是表函数的一个具体示例。我们实现了一个分隔字符串的函数`SplitFunction`， 可以将一个字符串转换成(字符串，长度)的二元组。

```java
//注意这里的类型标注，输出是Row类型，Row中包含两个字段: word和length。
@FunctionHint(output = @DataTypeHint ("ROW<word STRING, length INT>"))
public static class SplitFunction extends TableFunction<Row> {
	public void eval(String str) {
		for (String s : str.split(" ")) {
			//使用collect ()方法发送一行数据
			collect(Row.of(s, s.length())) ;
    }
  }
}
//注册函数
tableEnv.createTemporarySys temFunction ("SplitFunction", SplitFunction.class);

//在SQL里调用注册好的函数
// 1.交叉联结
tableEnv.sq1Query (
	"SELECT myField,word,length " +
	"FROM MyTable, LATERAL TABLE (SplitFunction (myField))") ;

// 2.带ON TRUE条件的左联结
tableEnv.sqlQuery (
	"SELECT myField, word, length "+
	"FROM MyTable "+
	"LEFT JOIN LATERAL TABLE (SplitFunction (myField)) ON TRUE") ;

//重命名侧向表中的字段
tableEnv.sq1Query (
	"SELECT myField, newWord, newLength "+
	"FROM MyTable”+
	"LEFT JOIN LATERAL TABLE (Spl itFunction (myField)) AS T (newWord, newLength) ON TRUE") ;
```

实际案例:

```java
        String createDDL = "CREATE TABLE clickTable (" +
                " user_name STRING, " +
                " url STRING, " +
                " ts BIGINT, " +
                " et AS TO_TIMESTAMP( FROM_UNIXTIME(ts/1000) ), " +
                " WATERMARK FOR et AS et - INTERVAL '1' SECOND " +
                ") WITH (" +
                " 'connector' = 'filesystem'," +
                " 'path' = 'input/clicks.txt'," +
                " 'format' = 'csv' " + // 定义分隔符
                ")";
        tableEnv.executeSql(createDDL);

        // 2. 注册自定义表函数
        tableEnv.createTemporaryFunction("MySplit", MySplitFunction.class);
        // 3. 调用UDF进行查询转换
        Table resultTable = tableEnv.sqlQuery("select user_name,url,word,length " +
                " from clickTable, LATERAL TABLE(MySplit(url)) AS T(word,length) ");
        // 4. 转换成流打印输出
        tableEnv.toDataStream(resultTable).print();

        env.execute();
    }

    // 实现自定义的表函数
    public static class MySplitFunction extends TableFunction<Tuple2<String, Integer>> {
        public void eval(String str) {
            String[] fields = str.split("\\?");
            for (String field : fields) {
                collect(Tuple2.of(field, field.length()));
            }
        }
    }

// 输出:
+I[Mary, ./home, ./home, 6]
+I[Alice, ./cart, ./cart, 6]
+I[Bob, ./prod?id=100, ./prod, 6]
+I[Bob, ./prod?id=100, id=100, 6]
+I[Bob, ./cart, ./cart, 6]
+I[Bob, . /home, . /home, 7]
+I[Mary, . /home, . /home, 7]
+I[Bob, ./cart, ./cart, 6]
+I[Bob, . /home, . /home, 7]
+I[Bob, ./prod?id=10, ./prod, 6]
+I[Bob, ./prod?id=10, id=10, 5]
+I[Alice, ./cart, ./cart, 6]
+I[Alice, ./home, ./home, 6]
+I[Bob, . /home, . /home, 7]
+I[Bob, . /home, . /home, 7]
```

##### 聚合函数(Aggregate Functions)

用户自定义聚合函数(User Defined AGGregate function，UDAGG)会把一行或多行数据(也就是一个表)聚合成-一个标量值。这是一个标准的“多对一”的转换。

聚合函数的概念我们之前已经接触过多次，如`SUM()`、`MAX()`、`MIN()`、`AVG()`、`COUNT()`都是常见的系统内置聚合函数。而如果有些需求无法直接调用系统函数解决，我们就必须自定义聚合函数来实现功能了。

自定义聚合函数需要继承抽象类`AggregateFunction`。 `AggregateFunction`有两个泛型参数`<T,ACC>`,`T`表示聚合输出的结果类型,`ACC`则表示聚合的中间状态类型。
Flink SQL中的聚合函数的工作原理如下:
- 首先，它需要创建一个累加器(accumulator)，用来存储聚合的中间结果。这与`DataStream API`中的`AggregateFunction`非常类似，累加器就可以看作是一个聚合状态。调用`createAccumulator()`方法可以创建一个空的累加器;
- 对于输入的每一行数据， 都会调用`accumulate()`方法来更新累加器，这是聚合的核心过程;
- 当所有的数据都处理完之后，通过调用`getValue()`方法来计算并返回最终的结果。

所以每个`AggregateFunction`都必须实现以下几个方法:

- `createAccumulator()`: 创建累加器的方法，返回类型为累加类型ACC;
- `accumulate()`: 进行聚合计算的核心方法，每来一行数据都会调用。第一个参数是确定的，就是当前的累加器，类型为ACC，表示当前聚合的中间状态。后面的参数则是聚合函数调用时传入的参数，可以有多个，类型也可以不同。该方法必须手动实现，不是`override`;
- `getValue()`: 得到最终返回结果的方法，输入参数是ACC类型的累加器，输出类型为T；
- 在遇到复杂类型时，Flink的类型推导可能会无法得到正确的结果。所以`AggregateFuncion`也可以专门对累加器和返回结果的类型进行声明，这里是通过`getAccumulatorType()`和`getResultType()`两个方法来指定；

除了上面的方法，还有几个方法是可选的。这些方法有些可以让查询更加高效，有些是在某些特定场景下必须要实现的。比如
- 如果是对会话窗口进行聚合，`merge()`方法就是必须要实现的，它会定义累加器的合并操作，而且这个方法对一些场景的优化也很有用;
- 如果聚合函数用在OVER窗口聚合中，就必须实现`retract()`方法， 保证数据可以进行撤回操作;
`resetAccumulator()`方法则是重置累加器，这在一些批处理场景中会比较有用。

AggregateFunction的所有方法都必须是公有的( public)，不能是静态的(static)， 而且名字必须跟上面写的完全一样。`createAccumulator`、`getValue`、`getResultType`以及`getAccumulatorType`这几个方法是在抽象类AggregateFunction 中定义的，可以override; 而其他则都是底层架构约定的方法。

示例:

```java
package com.luke.flink.datastreamdemo;

import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.table.api.Table;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
import org.apache.flink.table.functions.AggregateFunction;

public class UdfTest_AggregateFunction {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);

        // 读取数据源
        DataStreamSource<Event> eventStream = env.addSource(new ClickLogs());
        String createDDL = "CREATE TABLE clickTable (" +
                " user_name STRING, " +
                " url STRING, " +
                " ts BIGINT, " +
                " et AS TO_TIMESTAMP( FROM_UNIXTIME(ts/1000) ), " +
                " WATERMARK FOR et AS et - INTERVAL '1' SECOND " +
                ") WITH (" +
                " 'connector' = 'filesystem'," +
                " 'path' = 'input/clicks.txt'," +
                " 'format' = 'csv' " + // 定义分隔符
                ")";
        tableEnv.executeSql(createDDL);

        // 2. 注册自定义表函数
        tableEnv.createTemporaryFunction("WeightedAverage", WeightedAverage.class);
        // 3. 调用UDF进行查询转换
        Table resultTable = tableEnv
                .sqlQuery("select user_name, WeightedAverage(ts,1) as w_avg  from clickTable group by user_name");
        // 4. 每来一条数据后,对当前的加权平均是由影响的
        tableEnv.toChangelogStream(resultTable).print();
        env.execute();
    }

    // 单独定义一个累加器类型
    public static class WeightedAgvAccumulator {
        public long sum = 0;
        public int count = 0;
    }

    // 实现自定义的聚合函数,计算加权平均值
    public static class WeightedAverage extends AggregateFunction<Long, WeightedAgvAccumulator> {
        @Override
        public Long getValue(WeightedAgvAccumulator accumulator) {
            if (accumulator.count == 0) {
                return null;
            } else {
                return accumulator.sum / accumulator.count;
            }
        }

        @Override
        public WeightedAgvAccumulator createAccumulator() {
            return new WeightedAgvAccumulator();
        }

        // 累加计算的方法
        // iValue 这个示例中表示的是 timestamp, iWeight 表示次数
        public void accumulate(WeightedAgvAccumulator accumulator, Long iValue, Integer iWeight) {
            accumulator.sum += iValue * iWeight;
            accumulator.count += iWeight;
        }
    }
}
//结果
+I[Mary, 1000]
+I[Alice, 2000]
+I[Bob, 3000]
-U[Bob, 3000]
+U[Bob, 3500]
-U[Bob, 3500]
+U[Bob, 4000]
-U[Mary, 1000]
+U[Mary, 3500]
-U[Bob, 4000]
+U[Bob, 4750]
...
```

##### 表聚合函数 (Table Aggregate Functions)

用户自定义表聚合函数(UDTAGG)可以把一行或多行数据(也就是一个表)聚合成另一张表，结果表中可以有多行多列。很明显，这就像表函数和聚合函数的结合体，是一个“多对多”的转换。**可以有多行数据输入，输入后又可以有多行数据输出，所以是 多对多**

自定义表聚合函数需要继承抽象类`TableAggregateFunction`。`TableAggregateFunction` 的结构和原理与`AggregateFunction`非常类似，同样有两个泛型参数`<T,ACC>`,用一个ACC类型的累加器( accumulator)来存储聚合的中间结果。聚合函数中必须实现的三个方法，在`TableAggregateFunction`中也必须对应实现:

- `createAccumulator()`

  创建累加器的方法 与 `AggregateFunction`中用法相同;

- `accumulate()`: 与 `AggregateFunction`中用法相同;

- `emitValue()`: 

  所有输入行处理完成后，输出最终计算结果的方法。这个方法对应着`AggregateFunction`中的`getValue()`方法;区别在于`emitValue`没有输出类型，而输入参数有两个:第一个是`ACC`类型的累加器，第二个则是用于输出数据的“收集器”`out`,它的类型为`Collect<T>`. 所以很明显，表聚合函数输出数据不是直接`retturn`, 而是调用`out.collect()`方法， 调用多次就可以输出多行数据了;这一点与表函数非常相似。另外,`emitValue()`在抽象类中也没有定义,无法override,必须手动实现。

- **后面看吧！！！**

### SQL客户端

使用SQL客户端提供了一个命令行交互界面(CLI)，我们可以再里面非常容易地编写SQL进行查询，就像使用MySQL一样；整个Flink应用编写、提交的过程全变成了SQL，不需要写一行 Java/Scala代码。

**默认的启动模式是`embedded`，也就是说客户端是一个嵌入在本地的进程，这是目前唯一支持的模式。未来会支持连接到远程SQL客户端的模式**

```
$ cd /usr/local/flink-1.13.6
$ ./bin/sql-client.sh
Flink SQL>  SET 'execution.runtime-mode'='streaming'; /流处理
```

SQL客户端的执行结果模式,主要有`table`、`changelog`、`tableau`三种，默认为table模式

```sql
Flink SQL>  SET 'sql-client.execution.result-mode' = 'table';
```

table模式就是最普通的表处理模式，结果会以都好分隔每个字段；`changelog`则是更新日志模式，会在数据前佳航"+"(表示插入) 或 "-"(表示撤回)的前缀；而`tableau`则是经典的可视化表模式，结果会是一个虚线框的表格。

此外我们还可以做一些其他可选的配置，比如之前提到的空闲状态生存时间`TTL`:

```java
Flink SQL> SET 'table.exec.state.ttl' = '1000';
```

除了在命令行进行设置，我们也可以直接在SQL客户端的配置文件`sql-cli-defaults.yaml`中进行各种配置，甚至还可以在这个yaml文件里预定义表、函数和catalog。关于配置文件的更多用法，大家可以查阅官网的详细说明。

```sql
Flink SQL> CREATE TABLE clicksTable(
> user_name STRING,
> url STRING,
> `timestamp` BIGINT
> ) WITH (
> 'connector' = 'filesystem',
> 'path' = '/data/dbbak/clicks.txt',
> 'format' = 'csv'
> );
[INFO] Execute statement succeed.

Flink SQL> SET 'sql-client.execution.result-mode' = 'tableau';
[INFO] Session property has been set.

Flink SQL> select * from clicksTable;
+----+--------------------------------+--------------------------------+----------------------+
| op |                      user_name |                            url |            timestamp |
+----+--------------------------------+--------------------------------+----------------------+
| +I |                           Mary |                         ./home |                 1000 |
| +I |                          Alice |                         ./cart |                 2000 |
| +I |                            Bob |                  ./prod?id=100 |                 3000 |
| +I |                            Bob |                         ./cart |                 4000 |
| +I |                            Bob |                        . /home |                 5000 |
| +I |                           Mary |                        . /home |                 6000 |
| +I |                            Bob |                         ./cart |                 7000 |
| +I |                            Bob |                        . /home |                 8000 |
| +I |                            Bob |                   ./prod?id=10 |                 9000 |
| +I |                          Alice |                         ./cart |                12000 |
| +I |                          Alice |                         ./home |                13000 |
| +I |                            Bob |                        . /home |                18000 |
| +I |                            Bob |                        . /home |                15000 |
+----+--------------------------------+--------------------------------+----------------------+
Received a total of 13 rows

Flink SQL> set 'execution.runtime-mode'='batch';
[INFO] Session property has been set.

Flink SQL> select user_name,count(url) from clicksTable group by user_name;
+-----------+--------+
| user_name | EXPR$1 |
+-----------+--------+
|      Mary |      2 |
|     Alice |      3 |
|       Bob |      8 |
+-----------+--------+
3 rows in set
```

### 连接到外部系统

#### Kafka

Kafka的SQL连接器可以从Kafka的主题(topic) 读取数据转换成表，也可以将表数据写入Kafka的主题。换句话说，创建表的时候指定连接器为Kafka,则这个表既可以作为输入表，也可以作为输出表。

- **引入依赖**

  想要在Flink程序中使用Kafka连接器，需要引入如下依赖:

  ```xml
      <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-connector-kafka_${scala.binary.version}</artifactId>
        <version>${flink.version}</version>
      </dependency>
  ```

  这里我们引入的Flink和Kafka的连接器，与之前DataStream API中引入的连接器是一样的。如果想在SQL客户端里使用Kafka连接器，还需要下载对应的jar包放到lib目录下。

  另外，Flink为各种连接器提供了一系列的“表格式”(tableformats)，比如CSV、JSON、Avro、Parquet 等等。这些表格式定义了底层存储的二进制数据和表的列之间的转换方式，相当于表的序列化工具。对于Kafka而言，CSV、JSON、Avro等主要格式都是支持的。

  根据Kafka连接器中配置的格式，我们可能需要引入对应的依赖支持。以CSV为例:

  ```xml
      <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-csv</artifactId> 
        <version>1.13.0</version>
      </dependency>
  ```

  由于SQL客户端中已经内置了CSV、JSON的支持，因此使用时无需专门引入;而对于没有内置支持的格式(比如Avro)，则仍然要下载相应的jar包。

- **创建连接到kafka的表**

  ```sql
  Flink SQL> CREATE TABLE clicksTable(
  > user_name STRING,
  > url STRING,
  > `ts` TIMESTAMP(3) METADATA FROM 'timestamp' 
  > ) WITH (
  > 'connector' = 'kafka',
  > 'topic' = 'events',
  > 'properties.bootstrap.servers' = 'hadoop101:9092',
  > 'properties.group.id' = 'testGroup',
  > 'scan.startup.mode' = 'earliest-offset',
  > 'format' = 'csv'
  > );
  [INFO] Execute statement succeed.
  
  Flink SQL> select * from clicksTable;
  [ERROR] Could not execute SQL statement. Reason:
  java.lang.ClassNotFoundException: org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer
  ```
  
  **需要特别说明的是，在KafkaTable的字段中有一个ts，它的声明中用到了`METADATA FROM`，这是表示一个“元数据列”(metadata column)，它是由Kafka连接器的元数据“timestamp"生成的。这里的timestamp其实就是Kafka中数据自带的时间戳，我们把它直接作为元数据提取出来，转换成一个新的字段`ts`**
  
  上面的kafka 连接只能做INSERT 不能做 UPDATE
  
- **Upsert Kafka**

  正常情况下，Kafka作为保持数据顺序的消息队列，读取和写入都应该是流式的数据，对应在表中就是仅追加(append-only)模式。如果我们想要将有更新操作(比如分组聚合)的结果表写入Kafka，就会因为Kafka无法识别撤回(retract) 或更新插入(upsert) 消息而导致异常。
  
  为了解决这个问题，Flink 专门增加了一个“更新插入Kafka" (Upsert Kafka) 连接器。这个连接器支持以更新插入(UPSERT) 的方式向Kafka的topic 中读写数据。
  
  **具体来说，UpsertKafka连接器处理的是更新日志(changlog)流。如果作为TableSource 连接器会将读取到的topic中的数据(key,value),解释为对当前key的数据值的更新(UPDATE),也就是查找动态表中key对应的一行数据，将value更新为最新的值;因为是Upsert操作，所以如果没有key 对应的行，那么也会执行插入(INSERT) 操作。另外，如果遇到value为空(null)，连接器就把这条数据理解为对相应key那一行的删除(DELETE)操作**
  
  **如果作为TableSink， Upsert Kafka 连接器会将有更新操作的结果表，转换成更新日志(changelog)流。如果遇到插入(INSERT) 或者更新后(UPDATE_ AFTER) 的数据，对应的是一个添加(add) 消息，那么就直接正常写入Kafka主题;如果是删除(DELETE)或者更新前的数据，对应是一个撤回(retract)消息，那么就把value为空(null)的数据写入Kafka由于Flink是根据键(key)的值对数据进行分区的，这样就可以保证同一个key上的更新和删除**
  
  ```sql
  CREATE TABLE pv_per_region(
  user_region STRING,
  pv BIGINT,
  uv BIGINT,
  PRIMARY KEY (user_region) NOT ENFORCED
  ) WITH (
  'connector' = 'upsert-kafka',
  'topic' = 'pv_per_region',
  'properties.bootstrap.servers' = 'hadoop101:9092',
  'scan.startup.mode' = 'earliest-offset',
  'key.format' = 'avro',
  'value.format' = 'avro'
  );
  
  CREATE TABLE pageviews (
  user id BIGINT,
  page_id BIGINT,
  viewtime TIMESTAMP,
  user_region STRING,
  WATERMARK FOR viewtime AS viewtime - INTERVAL "2" SECOND
  ) WITH (
  'connector' = 'kafka',
  'topic' = 'pageviews',
  'properties.bootstrap.servers' = 'hadoop101:9092',
  'format' = 'json'
  );
  
  // 计算pv、uv并插入到upsert-kafka表中
  INSERT INTO pageviews_per_region
  SELECT
  user_region,
  COUNT (*),
  COUNT (DISTINCT user_id)
  FROM pageviews
  GROUP BY user_region;
  ```

#### 文件系统

```sql
CREATE TABLE clicksTable(
user_name STRING,
url STRING,
`timestamp` BIGINT
) WITH (
'connector' = 'filesystem',
'path' = '/data/dbbak/clicks.txt',
'format' = 'csv'
);
```

#### JDBC

关系型数据表本身就是SQL最初应用的地方，所以我们也会希望能直接向关系型数据库中读写表数据。Flink 提供的JDBC连接器可以通过JDBC驱动程序(driver) 向任意的关系型数据库读写数据，比如MySQL、PostgreSQL、 Derby 等。

**作为TableSink向数据库写入数据时，运行的模式取决于创建表的DDL是否定义了主键(primary key)。如果有主键，那么JDBC连接器就将以更新插入(Upsert) 模式运行，可以向外部数据库发送按照指定键(key) 的更新(UPDATE)和删除(DELETE) 操作;如果没有定义主键，那么就将在追加(Append)模式下运行，不支持更新和删除操作**

1. 引入依赖

   ```xml
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
   ```

2. 创建JDBC的表:

   ```sql
   # 创建一张连接到MySQL的表
   CREATE TABLE MyTable (
   id BIGINT,
   name STRING,
   age INT,
   status BOOLEAN,
   PRIMARY KEY(id) NOT ENFORCED 
   ) WITH (
   'connector'= 'jdbc',
   'url' = 'jdbc:mysql: //localhost: 3306/mydatabase' ,
   'table-name' = 'users' // MySQL中真实的表名
   );
   
   # 将另一张表T的数据写入到MyTable表中
   INSERT INTO MyTable
   SELECT id,name, age,status FROM T;
   ```

   **这里创建表的DDL中定义了主键，所以数据会以Upsert模式写入到MySQL表中;而到MySQL的连接，是通过WITH子句中的url 定义的。要注意写入MySQL中真正的表名称是users，而MyTable是注册在Flink表环境中的表**

#### Elasticsearch

具体依赖与Elasticsearch服务器版本有关，对于6.x版本引入依赖如下:

```java
    <dependency>
      <groupId>org.apache.flink</groupId>
      <artifactId>flink-connector-elasticsearch6_${scala.binary.version}</artifactId>
      <version>${flink.version}</version>
    </dependency>
```

对于Elasticsearch 7以上版本，引入依赖是:

```xml
    <dependency>
      <groupId>org.apache.flink</groupId>
      <artifactId>flink-connector-elasticsearch7_${scala.binary.version}</artifactId>
      <version>${flink.version}</version>
    </dependency>
```

创建连接到Elasticsearch的表:

```sql
CREATE TABLE MyTable (
user_id STRING,
user_name STRING,
uv BIGINT,
pv BIGINT,
PRIMARY KEY (user_id) NOT ENFORCED
) WITH (
'connector' = 'elasticsearch-7',
'hosts' = 'http://localhost:9200',
'index' = 'users'
);
```

这里定义了主键，所以会以更新插入(Upsert)模式向Elasticsearch写入数据。

#### HBase

**在流处理场景下，连接器作为TableSink向HBase写入数据时，采用的始终是更新插入(Upsert)模式。也就是说，HBase 要求连接器必须通过定义的主键(primary key)来发送更新日志(changelog)。所以在创建表的DDL中，我们必须要定义行键(rowkey) 字段，并将它声明为主键;如果没有用PRIMARY KEY子句声明主键，连接器会默认把rowkey作为主键**

- **引入依赖**

  想要在Flink程序中使用HBase连接器，需要引入对应的依赖。目前Flink只对HBase的1.4.x和2.2.x版本提供了连接器支持，而引入的依赖也应该与具体的HBase版本有关。对于1.4版本引入依赖如下:

  ```xml
  <dependency>
  	<groupId>org.apache.flink</groupId>
  	<artifactId>link-connector-hbase-1.4${scala.binary.version}</artifactId>
  	<version>${flink.version}</version>
  </dependency> 
  ```

  对于HBase2.2版本，引入的依赖则是:

  ```xml
  <dependency>
  	<groupId>org.apache.flink</groupId>
  	<artifactId>flink-connector-hbase-2.2${scala.binary.version}</artifactId>
  	<version>${flink.version} </version>
  </dependency>
  ```

- **创建连接到HBase的表**

  由于HBase并不是关系型数据库,因此转换为FlinkSQL中的表会稍有一些麻烦。

  在DDL创建出的HBase表中，所有的列族(column family)都必须声明为ROW类型，在表中占据一个字段;而每个family中的列(column qualifer)则对应着ROW里的嵌套字段。我们不需要将HBase中所有 family 和 qualifier都在Flink SQL的表中声明出来，只要吧那些在查询中用到的声明出来即可。

  除了所有ROW类型的字段(对应着HBase中的family)， 表中还应有一个原子类型的字段,它就会被识别为HBase的rowkey。在表中这个字段可以任意取名，不一定定非要叫rowkey。

  ```sql
  ---
  创建一-张连接到HBase的表
  CREATE TABLE MyTable (
  rowkey INT,
  familyl ROW<q1 INT>,
  family2 ROW<q2 STRING, q3 BIGINT>,
  family3 ROW<q4 DOUBLE, q5 BOOLEAN, q6 STRING>,
  PRIMARY KEY (rowkey) NOT ENFORCED
  ) WITH ( 
  'connector' = 'hbase-1.4',
  'table-name' = 'mytable',
  'zookeeper.quorum' = 'localhost:2181'
  );
  ```

#### Hive

Apache Hive 作为一个基于Hadoop的数据仓库基础框架，可以说已经成为了进行海量数据分析的核心组件。**Hive支持类SQL的查询语言,可以用来方便对数据进行处理和统计分析,而且基于HDFS的数据存储有非常好的可扩展性，是存储分析超大量数据集的唯一选择。Hive的主要缺点在于查询的延迟很高，几乎成了离线分析的代言人。而Flink的特点就是实时性强，所以FlinkSQL与Hive的结合势在必行**

Flink与Hive的集成比较特别。Flink 提供了“Hive目录”(HiveCatalog)功能，允许使用Hive的“元存储”(Metastore)来管理Flink的元数据。这带来的好处体现在两个方面:

- Metastore可以作为一个持久化的目录，因此使用HiveCatalog可以跨会话存储Flink特定的元数据。这样一来, 我们在HiveCatalog中执行创建Kafka表或者ElasticSearch 表，就可以把它们的元数据持久化存储在Hive的Metastore中;对于不同的作业会话就不需要重复创建了，直接在SQL查询中重用就可以;

- 使用HiveCatalog, Flink 可以作为读写Hive表的替代分析引擎。这样一来， 在Hive中进行批处理会更加高效;与此同时，也有了连续在Hive中读写数据、进行流处理的能力，这也使得“实时数仓”(real-time data warehouse)成为了可能;

**HiveCatalog被设计为“开箱即用”，与现有的Hive配置完全兼容，我们不需要做任何的修改与调整就可以直接使用。注意只有Blink的计划器(planner) 提供了Hive 集成的支持,所以需要在使用Flink SQL时选择Blink planner。'下面我们就来看以下与Hive集成的具体步骤**

- 引入依赖
  Hive各版本特性变化比较大，所以使用时需要注意版本的兼容性。目前Flink支持的Hive版本包括:

  -  Hive 1.x: 1.0.0~1.2.2;
  -  Hive 2.x: 2.0.0~2.2.0， 2.3.0~2.3.6
  - Hive 3.x: 3.0.0~3.1.2;I
    目前Flink与Hive 的集成程度在持续加强，支持的版本信息也会不停变化和调整，大家可以随着关注官网的更新信息。
    由于Hive是基于Hadoop的组件，因此我们首先需要提供Hadoop的相关支持，在环境变量中设置`HADOOP_CLASSPATH`

    ```shell
    export HADOOP_CLASSPATH= `hadoop classpath`
    ```

  - 引入依赖:

    ```xml
    <dependency>
    	<groupId>org.apache.flink</groupId>
    	<artifactId>flink-connector-hive${scala.binary.version}</artifactId>
    	<version>$(flink.version)</version>
    </dependency>
    
    <dependency>
    	<groupId>org.apache.hive</groupId>
    	<artifactId>hive-exec</artifactId>
    	<version>${hive.version}</version>
    </dependency>
    ```

- 连接到Hive

  在Flink中连接Hive，是通过在表环境中配置HiveCatalog来实现的。需要说明的是，配置HiveCatalog本身并不需要限定使用哪个planner, 不过对Hive表的读写操作只有Blink的planner才支持。所以一般我们需要将表环境的planner设置为Blink。

  ```java
  EnvironmentSettings settings = EnvironmentSettings.newInstance().useBlinkPlanner().build();
  TableEnvironment tableEnv = TableEnvironment.create(settings);
  string name = "myhive";
  String defaultDatabase = "mydạtabase";
  string hiveConfDir = "/opt/hive-conf";
  //创建一个HiveCatalog,并在表环境中注册
  HiveCatalog hive = new HiveCatalog(name,defaultDatabase,hiveConfDir);
  tableEnv.registerCatalog("myhive", hive);
  //使用HiveCatalog作为当前会话的catalog
  tableEnv.useCatalog("myhive");
  ```

  当然，我们也可以直接启动SQL客户端，用CREATE CATALOG语句直接创建HiveCatalog:
  ```sql
  Flink SQL> create catalog myhive with ('type' = 'hive', 'hive-conf-dir' = ' /opt/hive-conf') ;
  [ INFO] Execute statement succeed .
  
  Flink SQL> use catalog myhive;
  [ INFO] Execute statement succeed .
  ```

- SQL方言，就是在Flink中用Hive的SQL 语法

  我们知道, Hive内部提供了类SQL的查询语言,不过语法细节与标准SQL会有一些出入，相当于是SQL的一种“方言”(dialect)。为了提高与Hive集成时的兼容性，Flink SQL提供了一个非常有趣而强大的功能:可以使用方言来编写SQL语句。换句话说,我们可以直接在Flink中写HiveSQL来操作Hive表，这无疑给我们的读写处理带来了极大的方便。

  Flink目前支持两种SQL方言的配置: `default`和`hive`。所谓的default就是Flink SQL默认的SQL语法了。我们需要先切换到hive方言，然后才能使用Hive SQL的语法。具体设置可以分为SQL和Table API两种方式。
  - SQL中设置
  我们可以通过配置table.sql- dialect属性来设置SQL方言:
  ```
  set table.sql-dialect=hive;
  ```
  当然，我们可以在代码中执行上面的SET语句，也可以直接启动SQL客户端来运行。如果使用SQL客户端，我们还可以在配置文件sql-cli-defaults.yaml中通过“configuration"模块来设置:

  ```yaml
  execution:
  	planner: blink
  	type: batch
  	result-mode: table
  configuration:
  table.sql-dialect: hive
  ```

- 读写Hive表

  有了SQL方言的设置，我们就可以很方便的在Flink中创建Hive表并进行读写操作了。Flink支持以批处理和流处理模式向Hive中读写数据。在批处理模式下，Flink 会在执行查询语句时对Hive表进行一次性读取，在作业完成时将结果数据向Hive表进行一次性写入; 而在流处理模式下，Flink 会持续监控Hive表，在新数据可用时增量读取，也可以持续写入新数据并增量式地让它们可见。

  更灵活的是，我们可以随时切换SQL方言，从其它数据源(例如Kafka)读取数据、经转换后再写入Hive。下面是以纯SQL形式编写的一个示例,我们可以启动SQL客户端来运行:

  ```sql
  # 设置SQL方言为hive,创建Hive表
  SET table. sql-dialect=hive;
  CREATE TABLE hive_table (
  user_id STRING,
  order_amount DOUBLE
  ) PARTITIONED BY (dt STRING, hr STRING) STORED AS parquet TBLPROPERTIES (
  'partition.time-extractor.timestamp-pattern'='$dt $hr:00:00',
  'sink.partition-commit.trigger '='partition-time',
  'sink.partition-commit.delay'='1 h',
  'sink.partition-commit.policy.kind'= 'metastore, success-file'
  );
  
  
  # 没置SQL方言カdefault,創建Kafka表I
  SET table.sql-dialect=default;
  CREATE TABLE kafka_table ( 
  user_id STRING,
  order_amount DOUBLE,
  log_ts TIMESTAMP(3), 
  WATERMARK FOR log_ts AS log_ts - INTERVAL '5' SECOND -定义水位线
  WITH (...) ;
    
  # 将Kafka中读取的数据经转换后写入Hive 
  INSERT INTO TABLE hive table
  SELECT user_id, order_amount,DATE_FORMAT(log_ts, 'yyyy-MM-dd'),DATE_FORMAT(log_ts,'HH')
  FROM kafka table;
  ```

  这里我们创建Hive表时设置了通过分区时间来触发提交的策略。将Kafka中读取的数据经转换后写入Hive，这是一个流处理的Flink SQL程序。
