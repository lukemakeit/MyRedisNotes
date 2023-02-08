## Transformation 转换算子

##### 映射 map: 一一映射，消费一个元素就产生一个元素

```java
package com.luke.flink.datastreamdemo;

import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class TransformMapTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);

        // 从元素中读取数据
        DataStreamSource<Event> stream = env.fromElements(
                new Event("Mary", "./home", 1000L),
                new Event("Alice", "./cart", 2000L),
                new Event("Blob", "./prod?id=100", 3000L));
        // 进行转换操作,提取user字段
        // 1. 使用自定义类,实现 MapFunction接口
        SingleOutputStreamOperator<String> ret1 = stream.map(new MyMapper());
        // 2. 使用匿名类实现MapFunction接口
        SingleOutputStreamOperator<String> ret2 = stream.map(new MapFunction<Event, String>() {
            @Override
            public String map(Event value) throws Exception {
                return value.user;
            }
        });
        // 3. 传入lambda 表达式
        // 类似前面的 StreamWordCount中的 flatMap 代码,如果很难推断出返回值,则后面还需要用 returns 来定义类型
        // 不过现在返回类型是确定的
        SingleOutputStreamOperator<String> ret3 = stream.map(data -> data.user);
        ret3.print();
        env.execute();
    }

    // 自定义MapFunction
    public static class MyMapper implements MapFunction<Event, String> {
        @Override
        public String map(Event value) throws Exception {
            return value.user;
        }

    }
}
```

##### 过滤(filter)

```java

        // 1. 传入一个实现了 FilterFunction类的对象
        SingleOutputStreamOperator<Event> filterRet = stream.filter(new MyFilter());

        // 2. 传入匿名类实现FilterFunction接口
        SingleOutputStreamOperator<Event> filterRet2 = stream.filter(new FilterFunction<Event>() {
            @Override
            public boolean filter(Event value) throws Exception {
                return value.user.equals("Mary");
            }
        });
        // 3. 传入lambda表达式
        SingleOutputStreamOperator<Event> filterRet3 = stream.filter(data -> data.user.equals("Mary"));

        filterRet3.print();

    public static class MyFilter implements FilterFunction<Event> {
        @Override
        public boolean filter(Event value) throws Exception {
            return value.user.equals("Mary");
        }
    }
```

##### 扁平映射 (flatMap)

**主要是将数据流中的整体(一般是集合类型)拆分成一个个的个体使用，消费一个元素，可以产生0到多个元素**
**flatMap是 "扁平化"(flatten) 和 "映射"(map) 两步操作的结合，也就是按照某种规则对数据进行打散拆分，再对拆分后的元素做转换处理**
例如前面的WordCount程序第一步分词操作一样，就用到了flatMap:

```java
lineDataStream.flatMap((String line, Collector<Tuple2<String, Long>> out) -> {
                    // 分词
                    String[] words = line.split("\\s+");
                    // 转换成二元组输出
                    for (String word : words) {
                        out.collect(Tuple2.of(word, 1L)); // 必须是 1L
                    }
                }).returns(new TypeHint<Tuple2<String, Long>>() {}); // 定义返回类型?
```

示例:
```java

        // 1. 传入一个实现了 FlatMapFunction 类的对象
        SingleOutputStreamOperator<String> flatRet = stream.flatMap(new MyFlatMap());

        // 2. 传入lamba 表达式
        SingleOutputStreamOperator<String> flatRet2 = stream.flatMap((Event value, Collector<String> out) -> {
            out.collect(value.user);
            out.collect(value.url);
            out.collect(value.timestamp.toString());
        }).returns(new TypeHint<String>() {});

        flatRet2.print();
        
    // 自定义类实现 FlatMapFunction
    public static class MyFlatMap implements FlatMapFunction<Event, String> {
        @Override
        public void flatMap(Event value, Collector<String> out) throws Exception {
            out.collect(value.user);
            out.collect(value.url);
            out.collect(value.timestamp.toString());
        }
    }
```

在map算子中，Flink可以从函数签名 `OUT map(IN value)`的实现中自动提取出结果的类型信息;
<font style="color:red">但是在flatmap中，它的函数签名`void flatMap(IN value,Collector<OUT> out)`被Java编译器编译成`void flatMap(IN value,Collector out)`，也就是说`Collector`的泛型被擦除了。此时Flink无法推断出输出的类型。我们就需要用 `returns(new TypeHint<String>(){})`等方式告诉Flink输出类型</font>

### 聚合算子(Aggregation) 对应 reduce阶段

##### 按键分区 keyBy

```java
package com.luke.flink.datastreamdemo;
public class TransformSimpleAggTest {
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

        // 按键分组之后,提取当前用户最近一次访问数据
        // max 的输入输出类型是不变的
        // 每来一条数据, 都尝试更新一下 max,然后打印输出
        stream.keyBy(new KeySelector<Event, String>() {
            @Override
            public String getKey(Event value) throws Exception {
                return value.user;
            }
        }).max("timestamp").print("max: "); // 用的是字段名

        // 按键分组之后,提取当前用户最近一次访问数据
        stream.keyBy(data -> data.user).maxBy("timestamp").print("maxBy: "); // 用的是字段名

        env.execute();
    }
}
输出:
max: > Event [user=Mary, url=./home, timestamp=1970-01-01 00:00:01.0]
max: > Event [user=Alice, url=./cart, timestamp=1970-01-01 00:00:02.0]
max: > Event [user=Blob, url=./prod?id=100, timestamp=1970-01-01 00:00:05.0] #Blob来了 5000L 所以更新 max
max: > Event [user=Blob, url=./prod?id=100, timestamp=1970-01-01 00:00:05.0] #Blob来了 300L,但是max不更新还是打印 5000L,后面类似
max: > Event [user=Alice, url=./cart, timestamp=1970-01-01 00:00:06.0]
max: > Event [user=Blob, url=./prod?id=100, timestamp=1970-01-01 00:00:05.0]
maxBy: > Event [user=Mary, url=./home, timestamp=1970-01-01 00:00:01.0]
maxBy: > Event [user=Alice, url=./cart, timestamp=1970-01-01 00:00:02.0]
maxBy: > Event [user=Blob, url=./prod?id=100, timestamp=1970-01-01 00:00:05.0]
maxBy: > Event [user=Blob, url=./prod?id=100, timestamp=1970-01-01 00:00:05.0]
maxBy: > Event [user=Alice, url=./pencil, timestamp=1970-01-01 00:00:06.0]
maxBy: > Event [user=Blob, url=./prod?id=100, timestamp=1970-01-01 00:00:05.0]
```

##### reduce 方法

```java
package com.luke.flink.datastreamdemo;
public class TransformReduceTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        // env.setRuntimeMode(RuntimeExecutionMode.BATCH); 如果是批处理方式,则最后只输出一条
        // 从元素中读取数据
        DataStreamSource<Event> stream = env.fromElements(
                new Event("Mary", "./home", 1000L),
                new Event("Alice", "./cart", 2000L),
                new Event("Blob", "./prod?id=100", 5000L),
                new Event("Blob", "./prod?id=101", 300L),
                new Event("Alice", "./pencil", 6000L),
                new Event("Blob", "./prod?id=100", 4000L));
        // 1. 统计每个用户的访问次数
        SingleOutputStreamOperator<Tuple2<String, Long>> clicksByUser = stream
                .map(new MapFunction<Event, Tuple2<String, Long>>() {
                    @Override
                    public Tuple2<String, Long> map(Event value) throws Exception {
                        return Tuple2.of(value.user, 1L);
                    }
                }).keyBy(data -> data.f0). // 按照 用户名 分区
                reduce(new ReduceFunction<Tuple2<String, Long>>() {
                    @Override
                    public Tuple2<String, Long> reduce(Tuple2<String, Long> value1, Tuple2<String, Long> value2)
                            throws Exception {
                        return Tuple2.of(value1.f0, value1.f1 + value2.f1); // 每个用户的访问次数 ((1+1)+1)+1 一直累加
                    }

                });
        // 2. 选取当前最活跃的用户
        // 必须先 keyBy 然后再调用 sum()、min()、max()等聚合方法
        // keyBy(data->"key") 意思是将 所有数据分配到同一个分区
        // reduce做的是统计 访问次数最大的那个
        SingleOutputStreamOperator<Tuple2<String, Long>> result = clicksByUser.keyBy(data -> "key")
                .reduce(new ReduceFunction<Tuple2<String, Long>>() {
                    @Override
                    public Tuple2<String, Long> reduce(Tuple2<String, Long> value1, Tuple2<String, Long> value2)
                            throws Exception {
                        return value1.f1 > value2.f1 ? value1 : value2;
                    }
                });
        result.print();
        env.execute();
    }
}

//输出,每来一条数据,flink处理后都比较而后打印输出
(Mary,1)
(Alice,1)
(Blob,1)
(Blob,2)
(Alice,2)
(Blob,3)
```

### 富函数类(Rich Function Classes)

Rich函数一定会比常规的函数类提供更多、更丰富的功能。与常规函数不同的是，富函数类可以获取运行环境的上下文，并拥有生命周期方法，可以实现更复杂的功能。
所有的Flink函数类都有Rich版本，富函数一般是以抽象类的形式出现的。例如`RichMapFunction`、`RichFilterFunction`、`RichReduceFunction`等。
与常规函数不同的是，富函数类可以获取运行环境的上下文，并拥有一些生命周期方法，所以可以实现更复杂的功能。
注: 生命周期的概念在编程中其实非常重要，到处都有体现。如，C语言，需手动管理内存的分配和回收，也就是手动管理内存的生命周期。分配内存而不回收，会造成内存泄漏，回收没分配过内存，会造成空指针异常。
Rich Function 有生命周期的概念。典型的生命周期方法有:

- open()方法，是Rich Function的初始化方法，也就是开启一个算子的生命周期。当一个算子的实际工作方法，如`map()`或`filter()`方法被调用前，`open()`会首先被调用。所以像文件IO的创建，数据库连接的创建，配置文件的读取等这样一次性的工作，都适合在open()方法中完成;
- close()方法，是生命周期中的最后一个调用的方法，类似于解构方法。一般用来做一些清理工作;

需要注意的是: <font style="color:red">这里的生命周期方法，对于一个并行子任务来说只调用一次；而对应的，实际工作方法，如 RichMapFunction中的map()，在每条数据来了之后触发一次调用</font>
```java
public class TransformRichFunctionTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(2);
        // 从元素中读取数据
        DataStreamSource<Event> stream = env.fromElements(
                new Event("Mary", "./home", 1000L),
                new Event("Alice", "./cart", 2000L),
                new Event("Blob", "./prod?id=100", 3000L));
        stream.map(new MyRichMapper()).print();
        env.execute();
    }
    public static class MyRichMapper extends RichMapFunction<Event, Integer> {
        @Override
        public void open(Configuration param) throws Exception {
            super.open(param);
            System.out.println("open called,context:" + getRuntimeContext().getIndexOfThisSubtask() + "号任务启动");
        }
        @Override
        public Integer map(Event value) throws Exception {
            return value.url.length();
        }
        @Override
        public void close() throws Exception {
            super.close();
            System.out.println("close生命周期被调用:" + getRuntimeContext().getIndexOfThisSubtask() + "号任务结束");
        }
    }
}

// 结果输出:
open called,context:1号任务启动
open called,context:0号任务启动
2> 6
1> 6
1> 13
close生命周期被调用:1号任务结束
close生命周期被调用:0号任务结束
```

### 物理分区(Physical Partitioning)

"分区"(partitionning)操作就是将数据进行重新分布，传递到不同的流分区去进行下一步处理。
前面的**keyBy**就是按照key的哈希值来进行重新分区，不过这种分区操作只能确保数据按照**key**分开，至于分布是否均匀、每个key的数据具体分到哪个区，是完全无法控制的——所以，可以将 `keyBy`理解为一种逻辑分区(logical partitionning)操作。

如果说keyBy是一种软分区，那真正的硬分区就是物理分区(physical partitioning)。也就是我们要真正控制的分区策略，精准地调配数据，告诉每个数据区了哪里。其实这种分区方式在一些情况下已经在发生了:**例如我们编写的程序可能对多个处理任务设置了不同的并行度,那么当数据执行的上下游任务并行度变化时，数据就不应该还在当前分区以直逋(forward)方式传输了一一因为如果并行度变小， 当前分区可能没有下游任务了;而如果并行度变大，所有数据还在原先的分区处理就会导致资源的浪费。所以这种情况下，系统会自动地将数据均匀地发往下游所有的并行任务，保证各个分区的负载均衡**

- **shuffle(): 随机分区，打散数据，比如上游的并行度是1，下游的并行度是4，那么调用shuffle()可以做打散**
  例如:

  ```java
  stream.shuffle().print().setParallelism(4);
  
  输出结果:
  1> Event [user=Alice, url=./cart, timestamp=1970-01-01 00:00:02.0]
  1> Event [user=Blob, url=./prod?id=100, timestamp=1970-01-01 00:00:05.0]
  3> Event [user=Mary, url=./home, timestamp=1970-01-01 00:00:01.0]
  3> Event [user=Blob, url=./prod?id=101, timestamp=1970-01-01 00:00:00.3]
  3> Event [user=Alice, url=./pencil, timestamp=1970-01-01 00:00:06.0]
  2> Event [user=Alice, url=./pencil, timestamp=1970-01-01 00:00:06.0]
  2> Event [user=Alice, url=./pencil, timestamp=1970-01-01 00:00:06.0]
  4> Event [user=Alice, url=./pencil, timestamp=1970-01-01 00:00:06.0]
  4> Event [user=Blob, url=./prod?id=100, timestamp=1970-01-01 00:00:04.0]
  ```

- **rebalance():轮询分区,roundRobin**

  ```java
  stream.rebalance().print().setParallelism(4);
  输出结果:
  1> Event [user=Blob, url=./prod?id=101, timestamp=1970-01-01 00:00:00.3]
  1> Event [user=Alice, url=./pencil, timestamp=1970-01-01 00:00:06.0]
  3> Event [user=Alice, url=./cart, timestamp=1970-01-01 00:00:02.0]
  3> Event [user=Alice, url=./pencil, timestamp=1970-01-01 00:00:06.0]
  2> Event [user=Mary, url=./home, timestamp=1970-01-01 00:00:01.0]
  2> Event [user=Alice, url=./pencil, timestamp=1970-01-01 00:00:06.0]
  2> Event [user=Blob, url=./prod?id=100, timestamp=1970-01-01 00:00:04.0]
  4> Event [user=Blob, url=./prod?id=100, timestamp=1970-01-01 00:00:05.0]
  4> Event [user=Alice, url=./pencil, timestamp=1970-01-01 00:00:06.0]
  ```

- **rescale():重缩放分区，并非朝下游所有分区发送，比如上一个原子2个并发(f0 f1)，下一个原子 4个并发(b0 b1 b2 b3)。那么就是(f0: b0 b1,f1:b2 b3)**

- **broadcast():广播,一个数据默认只分配下游一个子任务上，现在是广播到所有子任务上**

  ```java
  stream.broadcast().print().setParallelism(4);
  ```

- **global():全局分区, 所有数据分配到下游第一个并行子任务上。就是下游设置多少并行度都没用**

  ```java
  stream.global().print().setParallelism(4); //设置4没用
  ```

- **partitionCustom():自定义重分区**

  ```java
          // 3.自定义重分区
          env.fromElements(1, 2, 3, 4, 5, 6, 7, 8).partitionCustom(
                  new Partitioner<Integer>() {
                      @Override
                      public int partition(Integer key, int numPartitions) { // partition传入类型 应该和后面 getKey返回类型一样
                          return key % 2; // 按照奇偶性分区,返回的结果是下一分区的索引号
                      }
                  }, new KeySelector<Integer, Integer>() { // 第一个参数代表拿到的参数类型, 第二个参数是返回的类型
                      @Override
                      public Integer getKey(Integer value) throws Exception {
                          return value;
                      }
                  }).print().setParallelism(4);
  // 最后结果,只有第一 第二个分区输出了
  2> 1
  2> 3
  2> 5
  2> 7
  1> 2
  1> 4
  1> 6
  1> 8
  ```

  
