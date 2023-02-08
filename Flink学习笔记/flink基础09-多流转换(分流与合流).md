### 分流 split

通过**outputTag**实现

```java
        OutputTag<Tuple3<String, String, Long>> maryTag = new OutputTag<Tuple3<String, String, Long>>("mary") {
        };
        OutputTag<Tuple3<String, String, Long>> bobTag = new OutputTag<Tuple3<String, String, Long>>("bob") {
        };

        SingleOutputStreamOperator<Event> processStream = stream.process(new ProcessFunction<Event, Event>() {
            @Override
            public void processElement(Event value, ProcessFunction<Event, Event>.Context ctx, Collector<Event> out)
                    throws Exception {
                if (value.user.equals("Mary")) {
                    ctx.output(maryTag, Tuple3.of(value.user, value.url, value.timestamp));
                } else if (value.user.equals("Bob")) {
                    ctx.output(bobTag, Tuple3.of(value.user, value.url, value.timestamp));
                } else {
                    out.collect(value);
                }
            }
        });
        processStream.print("eles");
        processStream.getSideOutput(maryTag).print("mary");
        processStream.getSideOutput(bobTag).print("bob");
        env.execute();

//输出结果
eles> Event [user=Alice, url=./cart, timestamp=2022-12-10 01:56:54.194]
mary> (Mary,./cart,1670637415195)
eles> Event [user=Alice, url=./home, timestamp=2022-12-10 01:56:56.197]
bob> (Bob,./prod?id=100,1670637420206)
bob> (Bob,./prod?id=10,1670637421207)
```

### 合流 Union

**union操作要求必须流中的数据类型相同，合并后的新流会包括所有流中的元素，数据类型不变**

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221210100254874.png" alt="image-20221210100254874" style="zoom:50%;" />

**在流处理中，上有任务处理完水位线、时钟改变之后，要把当前的水位线再次发出，广播给所有的下游子任务。而当一个任务接收到多个上游并行任务传递来的水位线时，应当以最小的那个作为当前任务的事件时钟**

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221129063809654.png" alt="image-20221129063809654" style="zoom:50%;" />

```java
        SingleOutputStreamOperator<Event> stream1 = env.socketTextStream("hadoop101", 7777)
                .map(new MapFunction<String, Event>() {
                    @Override
                    public Event map(String value) throws Exception {
                        String[] fields = value.split(",");
                        return new Event(fields[0].trim(), fields[1].trim(), Long.valueOf(fields[2].trim()));
                    }
                })
                .assignTimestampsAndWatermarks(WatermarkStrategy.<Event>forBoundedOutOfOrderness(Duration.ofSeconds(2)) // 延迟2秒
                        .withTimestampAssigner(new SerializableTimestampAssigner<Event>() {
                            @Override
                            public long extractTimestamp(Event element, long recordTimestamp) {
                                return element.timestamp;
                            }
                        }));

        stream1.print("stream1");

        SingleOutputStreamOperator<Event> stream2 = env.socketTextStream("hadoop101", 8888)
                .map(new MapFunction<String, Event>() {
                    @Override
                    public Event map(String value) throws Exception {
                        String[] fields = value.split(",");
                        return new Event(fields[0].trim(), fields[1].trim(), Long.valueOf(fields[2].trim()));
                    }
                })
                .assignTimestampsAndWatermarks(WatermarkStrategy.<Event>forBoundedOutOfOrderness(Duration.ofSeconds(5)) // 延迟5秒
                        .withTimestampAssigner(new SerializableTimestampAssigner<Event>() {
                            @Override
                            public long extractTimestamp(Event element, long recordTimestamp) {
                                return element.timestamp;
                            }
                        }));

        stream2.print("stream2");

        stream1.union(stream2).process(new ProcessFunction<Event, String>() {
            @Override
            public void processElement(Event value, ProcessFunction<Event, String>.Context ctx, Collector<String> out)
                    throws Exception {
                out.collect("水位线:" + ctx.timerService().currentWatermark());
            }
        }).print();

        env.execute();

//输出,下面 被减去2000L、5000L是上面的stream1 和stream2的水位线延迟,:
stream1> Event [user=Mary, url=./home, timestamp=1970-01-01 00:00:01.0] // 更新了stream1的水位线, 1000L-2000L-1 毫秒
水位线:-9223372036854775808  //但是stream2的水位线没被更新过,所以合流后的水位线和 stream2的相同
stream1> Event [user=Mary, url=./home, timestamp=1970-01-01 00:00:02.0]
水位线:-9223372036854775808
stream1> Event [user=Mary, url=./home, timestamp=1970-01-01 00:00:05.0] // 更新了stream1的水位线, 5000L-2000L-1 毫秒
水位线:-9223372036854775808
stream2> Event [user=Bob, url=./cart, timestamp=1970-01-01 00:00:04.0] // 更新了stream2的水位线, 4000L-5000L-1 毫秒
水位线:-9223372036854775808 //为啥stream2的水位线更新了,合流的水位线还是 最小值? 因为 ProcessFunction 执行完成后才会更新水位线。也就是现在水位线已经更新了，但是 要下一次才能打印
stream2> Event [user=Bob, url=./cart, timestamp=1970-01-01 00:00:04.0] // 更新了stream2的水位线, 4000L-5000L-1 毫秒
水位线:-1001 // 合流的水位线更新成了 stream2 的
stream1> Event [user=Mary, url=./home, timestamp=1970-01-01 00:00:05.0]
水位线:-1001
stream2> Event [user=Mary, url=./home, timestamp=1970-01-01 00:00:05.0] // 更新了stream2的水位线, 5000L-5000L-1 毫秒,要下次一次才体现在合流水位线上
水位线:-1001
stream2> Event [user=Mary, url=./home, timestamp=1970-01-01 00:00:05.0]
水位线:-1 // 合流水位线更新为 stream2的-1
```

### 连接 connect

因为只能处理相同的流类型，上面的union在实际应用中应用较少，Flink提供了另一种更方便的连接(connect)。

#### 连接流(ConnectedStreams)

最大优势，允许操作不同数据类型的流。

- connect 操作后，得到不是DataStream，而是一个ConnectedStreams;
- 想要通过 ConnectedStreams 得到 DataStream，得通过`co-process`操作。
- 每次只能操作两条流, 

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221210110406049.png" alt="image-20221210110406049" style="zoom:50%;" />

```java
public class StreamsConnectTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);

        DataStreamSource<Integer> stream1 = env.fromElements(1, 2, 3);
        DataStreamSource<Long> stream2 = env.fromElements(4l, 5L, 6L, 7L);

        stream1.connect(stream2)
                .map(new CoMapFunction<Integer, Long, String>() {
                    @Override
                    public String map1(Integer value) throws Exception {
                        return "Integer:" + value.toString();
                    }

                    @Override
                    public String map2(Long value) throws Exception {
                        return "Long:" + value.toString();
                    }
                }).print();

        env.execute();
    }
}
//输出:
Long:4
Integer:1
Long:5
Integer:2
Long:6
Integer:3
Long:7
```

**一个实施对账的例子**

```java
package com.luke.flink.datastreamdemo;
import java.time.Duration;
import org.apache.flink.api.common.eventtime.SerializableTimestampAssigner;
import org.apache.flink.api.common.eventtime.WatermarkStrategy;
import org.apache.flink.api.common.state.ValueState;
import org.apache.flink.api.common.state.ValueStateDescriptor;
import org.apache.flink.api.common.typeinfo.Types;
import org.apache.flink.api.java.tuple.Tuple3;
import org.apache.flink.api.java.tuple.Tuple4;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.co.CoProcessFunction;
import org.apache.flink.util.Collector;

public class StreamConnectBillCheck {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);

        // 来自app的支付日志
        SingleOutputStreamOperator<Tuple3<String, String, Long>> appStream = env.fromElements(
                Tuple3.of("order-1", "app", 1000L),
                Tuple3.of("order-2", "app", 2000L),
                Tuple3.of("order-3", "app", 3500L))
                .assignTimestampsAndWatermarks(WatermarkStrategy
                        .<Tuple3<String, String, Long>>forBoundedOutOfOrderness(Duration.ZERO)
                        .withTimestampAssigner(new SerializableTimestampAssigner<Tuple3<String, String, Long>>() {
                            @Override
                            public long extractTimestamp(Tuple3<String, String, Long> element, long recordTimestamp) {
                                return element.f2;
                            }
                        }));

        // 来自第三方支付平台的支付日志
        SingleOutputStreamOperator<Tuple4<String, String, String, Long>> thirdpartyStream = env.fromElements(
                Tuple4.of("order-1", "third-party", "success", 3000L),
                Tuple4.of("order-3", "third-party", "success", 4000L))
                .assignTimestampsAndWatermarks(WatermarkStrategy
                        .<Tuple4<String, String, String, Long>>forBoundedOutOfOrderness(Duration.ZERO)
                        .withTimestampAssigner(
                                new SerializableTimestampAssigner<Tuple4<String, String, String, Long>>() {
                                    @Override
                                    public long extractTimestamp(Tuple4<String, String, String, Long> element,
                                            long recordTimestamp) {
                                        return element.f3;
                                    }
                                }));

        // 检测同一支付单在两条流中是否匹配,不匹配则报警
        appStream.connect(thirdpartyStream)
                .keyBy(data -> data.f0, data -> data.f0) // 利用appStream.f0 和 thirdpartyStream.f0 相同的值进行链接
                .process(new OrderMatchResult())
                .print();

        env.execute();
    }

    // Tuple3是第一个输入类型、Tuple4是第二个输入类型、String是输出类型
    public static class OrderMatchResult
            extends CoProcessFunction<Tuple3<String, String, Long>, Tuple4<String, String, String, Long>, String> {
        // 定义状态变量,用于保存已到达的事件
        private ValueState<Tuple3<String, String, Long>> appEventState;
        private ValueState<Tuple4<String, String, String, Long>> thirdpartyEventState;

        @Override
        public void open(Configuration parameters) throws Exception {
            // 初始化 appEventState 和 thirdpartyEventState
            appEventState = getRuntimeContext().getState(
                    new ValueStateDescriptor<Tuple3<String, String, Long>>("app-Event",
                            Types.TUPLE(Types.STRING, Types.STRING, Types.LONG)));
            thirdpartyEventState = getRuntimeContext()
                    .getState(new ValueStateDescriptor<Tuple4<String, String, String, Long>>("thirdparty-Event",
                            Types.TUPLE(Types.STRING, Types.STRING, Types.STRING, Types.LONG)));
        }

        // processElement1 处理的是 appStream 中的数据
        @Override
        public void processElement1(Tuple3<String, String, Long> value,
                CoProcessFunction<Tuple3<String, String, Long>, Tuple4<String, String, String, Long>, String>.Context ctx,
                Collector<String> out) throws Exception {

            // 来的是app events,看另一条流中的事件是否来过
            if (thirdpartyEventState.value() != null) {
                // 来过, thirdpartyEventState中最多只保留一条数据
                out.collect("对账成功:" + value + " " + thirdpartyEventState.value());
                thirdpartyEventState.clear();
            } else {
                // 没来,则保存app event
                appEventState.update(value);
                // 注册一个5秒后的定时器,开始等待另一条流的事件
                ctx.timerService().registerEventTimeTimer(value.f2 + 5000L);
            }
        }

        // processElement2 处理的是 thirdpartyStream 中的数据
        @Override
        public void processElement2(Tuple4<String, String, String, Long> value,
                CoProcessFunction<Tuple3<String, String, Long>, Tuple4<String, String, String, Long>, String>.Context ctx,
                Collector<String> out) throws Exception {
            // 来的是thirdparty events,看另一条流中的事件是否来过
            if (appEventState.value() != null) {
                // 来过, thirdpartyEventState中最多只保留一条数据
                out.collect("对账成功:" + value + " " + appEventState.value());
                appEventState.clear();
            } else {
                // 没来,则保存app event
                thirdpartyEventState.update(value);
                // 注册一个5秒后的定时器,开始等待另一条流的事件
                ctx.timerService().registerEventTimeTimer(value.f3 + 5000L);
            }
        }

        @Override
        public void onTimer(long timestamp,
                CoProcessFunction<Tuple3<String, String, Long>, Tuple4<String, String, String, Long>, String>.OnTimerContext ctx,
                Collector<String> out) throws Exception {
            // 定时器触发,判断状态,如果某个状态不为空,说明另一条流中事件没来
            if (appEventState.value() != null) {
                out.collect("对账失败: " + appEventState.value() + " 第三方支付信息未到");
            }
            if (thirdpartyEventState.value() != null) {
                out.collect("对账失败: " + appEventState.value() + " app支付信息未到");
            }
            appEventState.clear();
            thirdpartyEventState.clear();
        }
    }
}

//输出
对账成功:(order-1,third-party,success,3000) (order-1,app,1000)
对账成功:(order-3,app,3500) (order-3,third-party,success,4000)
对账失败: (order-2,app,2000) 第三方支付信息未到
```

**BroadcastConnectedStream: 广播连接流**

用于动态定义配置项的场景，一条流是标准数据流，另一个配置流，广播给下游所有子任务。

### 基于时间的合流——双流Join

上面的案例可以知道，基于`keyBy`的两条流的`connect`操作，和关系型数据库中标的join操作非常接近，另外connect支持处理函数，可以使用自定义状态和TimerService灵活实现各种需求，实际上已经可以处理双流join的大多数场景。
**不过直接用connect因为还是太复杂，比如，如果我们希望统计固定时间内两条流数据的匹配情况，那就需要设置定时器、自定义触发器来实现——其实完全可以用窗口(window)来表示**

#### 窗口join(Window Join)

可以定义时间窗口，并将两条流中共享一个公共键(key)的数据放在窗口中进行配对处理。

- **window join的调用**

  ```java
  stream1.join(stream2)
  	.where(<keySelector>)
  	.equalTo(<keySelector>)
  	.window(<WindowAssigner>)
  	.apply(<JoinFunction>)
  ```

  上面代码中`.where()`的参数是键选择器(KeySelector), 用来指定第一条流中的key;而`.equalTo()`传入的KeySelector则指定了第二条流中的key，两者相同的元素，如果在同一窗口中，就可以匹配起来，并通过`joinFunction`进行处理了。
  `.window()`传入的就是窗口分配器，前面的滚动窗口、滑动窗口等均可以在这里使用。

  `.apply()`可看作实现了一个特殊的窗口函数。

```java
package com.luke.flink.datastreamdemo;
import java.time.Duration;

import org.apache.flink.api.common.eventtime.SerializableTimestampAssigner;
import org.apache.flink.api.common.eventtime.WatermarkStrategy;
import org.apache.flink.api.common.functions.JoinFunction;
import org.apache.flink.api.java.tuple.Tuple3;
import org.apache.flink.api.java.tuple.Tuple4;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.windowing.assigners.TumblingEventTimeWindows;
import org.apache.flink.streaming.api.windowing.time.Time;

public class StreamWindowJoin {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        // 来自app的支付日志
        SingleOutputStreamOperator<Tuple3<String, String, Long>> appStream = env.fromElements(
                Tuple3.of("order-1", "app", 1000L),
                Tuple3.of("order-2", "app", 2000L),
                Tuple3.of("order-3", "app", 3500L))
                .assignTimestampsAndWatermarks(WatermarkStrategy
                        .<Tuple3<String, String, Long>>forBoundedOutOfOrderness(Duration.ZERO)
                        .withTimestampAssigner(new SerializableTimestampAssigner<Tuple3<String, String, Long>>() {
                            @Override
                            public long extractTimestamp(Tuple3<String, String, Long> element, long recordTimestamp) {
                                return element.f2;
                            }
                        }));
        // 来自第三方支付平台的支付日志
        SingleOutputStreamOperator<Tuple4<String, String, String, Long>> thirdpartyStream = env.fromElements(
                Tuple4.of("order-1", "third-party", "todo", 500L),
                Tuple4.of("order-1", "third-party", "fail", 700L),
                Tuple4.of("order-1", "third-party", "success", 3000L),
                Tuple4.of("order-3", "third-party", "success", 4000L))
                .assignTimestampsAndWatermarks(WatermarkStrategy
                        .<Tuple4<String, String, String, Long>>forBoundedOutOfOrderness(Duration.ZERO)
                        .withTimestampAssigner(
                                new SerializableTimestampAssigner<Tuple4<String, String, String, Long>>() {
                                    @Override
                                    public long extractTimestamp(Tuple4<String, String, String, Long> element,
                                            long recordTimestamp) {
                                        return element.f3;
                                    }
                                }));

        appStream.join(thirdpartyStream)
                .where(data -> data.f0)
                .equalTo(data -> data.f0)
                .window(TumblingEventTimeWindows.of(Time.seconds(5)))
                .apply(new JoinFunction<Tuple3<String, String, Long>, Tuple4<String, String, String, Long>, String>() {
                    @Override
                    public String join(Tuple3<String, String, Long> first, Tuple4<String, String, String, Long> second)
                            throws Exception {

                        return first + " -> " + second;
                    }
                }).print();

        env.execute();
    }
}
// 结果
(order-1,app,1000) -> (order-1,third-party,todo,500)
(order-1,app,1000) -> (order-1,third-party,fail,700)
(order-1,app,1000) -> (order-1,third-party,success,3000)
(order-3,app,3500) -> (order-3,third-party,success,4000)
```

- **间隔join(Interval Join)**

  在某些场景中，我们要处理的时间间隔可能不是固定的。这时显然不应该用滚动窗口 或 滑动窗口来处理——因为匹配的两个数据可能刚好"卡在"窗口边缘两侧，于是窗口内就都没匹配了。
  为应对这样的需求，Flink提供了一种叫做"间隔join"(interval join)的合流操作。顾名思义，间隔join的思路就是: **针对一条流上的每个数据，开辟出其时间戳前后的一段时间间隔，看这个期间是否有来自另一条流的数据匹配**

  **原理:**

  间隔join具体的定义方式是,我们给定两个时间点,分别叫作间隔的“上界”( upperBound) 和“下界”(lowerBound);

  **于是对于一条流(不妨叫作A)中的任意一个数据元素a，就可以开辟一 段时间间隔: `[a.timestamp + lowerBound, a.timestamp + upperBound]`，即以a的时间戳为中心，下至下界点、上至，上界点的一个闭区间:**

  **我们就把这段时间作为可以匹配另一条流数据的“窗口”范围**

  **所以对于另一条流(不妨叫B)中的数据元素b,如果它的时间戳落在了这个区间范围内，a和b就可以成功配对，进而进行计算输出结果。所以匹配的条件为:`a.timestamp + lowerBound <= b.timestamp <= a.timestamp + upperBound`**

  注意: 做间隔join的两条流 A 和B，也必须是相同的key；且 lowerBound 小于等于 upperBound，两者可正可负。

  <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221211231937288.png" alt="image-20221211231937288" style="zoom:50%;" />

  调用方式:

  ```java
  stream1.
    .keyBy(<keySelector>)
  	.intervalJoin(stream2.keyBy(<keySelector>))
    .between(Time.milliseconds(2),Time.milliseconds(1))
  	.process(new ProcessJoinFunction<IN1,IN2,OUT>(){
      
    });
  ```

  示例, 为了方便分析用户行为，在用户下单后，将其浏览历史都筛选出来。所以我们有两条流，一条是下订单的流，一条最近10分钟浏览数据的流，也可能是稍后10分钟内继续浏览的数据流。

  ```java
  package com.luke.flink.datastreamdemo;
  
  import java.time.Duration;
  
  import org.apache.flink.api.common.eventtime.SerializableTimestampAssigner;
  import org.apache.flink.api.common.eventtime.WatermarkStrategy;
  import org.apache.flink.api.java.tuple.Tuple2;
  import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  import org.apache.flink.streaming.api.functions.co.ProcessJoinFunction;
  import org.apache.flink.streaming.api.windowing.time.Time;
  import org.apache.flink.util.Collector;
  
  public class StreamIntervalJoin {
      public static void main(String[] args) throws Exception {
          StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
          env.setParallelism(1);
          // 订单日志
          SingleOutputStreamOperator<Tuple2<String, Long>> orderStream = env.fromElements(
                  Tuple2.of("Mary", 1000L),
                  Tuple2.of("Alice", 20000L),
                  Tuple2.of("Blob", 3500L),
                  Tuple2.of("Alice", 4500L),
                  Tuple2.of("Blob", 5500L))
                  .assignTimestampsAndWatermarks(WatermarkStrategy
                          .<Tuple2<String, Long>>forBoundedOutOfOrderness(Duration.ZERO)
                          .withTimestampAssigner(new SerializableTimestampAssigner<Tuple2<String, Long>>() {
                              @Override
                              public long extractTimestamp(Tuple2<String, Long> element, long recordTimestamp) {
                                  return element.f1;
                              }
                          }));
  
          // 浏览记录日志
          SingleOutputStreamOperator<Event> clicksStream = env.fromElements(
                  new Event("Mary", "./home", 1000L),
                  new Event("Alice", "./cart", 2000L),
                  new Event("Blob", "./prod?id=100", 5000L),
                  new Event("Blob", "./prod?id=101", 300L),
                  new Event("Alice", "./pencil", 36000L),
                  new Event("Alice", "./pencil", 6000L),
                  new Event("Alice", "./pencil", 6000L),
                  new Event("Alice", "./pencil", 6000L),
                  new Event("Blob", "./prod?id=100", 4000L))
                  .assignTimestampsAndWatermarks(WatermarkStrategy
                          .<Event>forBoundedOutOfOrderness(Duration.ZERO)
                          .withTimestampAssigner(
                                  new SerializableTimestampAssigner<Event>() {
                                      @Override
                                      public long extractTimestamp(Event element,
                                              long recordTimestamp) {
                                          return element.timestamp;
                                      }
                                  }));
  
          orderStream.keyBy(data -> data.f0)
                  .intervalJoin(clicksStream.keyBy(data -> data.user))
                  .between(Time.seconds(-5), Time.seconds(10)) // 下订单之前5秒 到 之后10秒
                  .process(new ProcessJoinFunction<Tuple2<String, Long>, Event, String>() {
                      @Override
                      public void processElement(Tuple2<String, Long> left, Event right,
                              ProcessJoinFunction<Tuple2<String, Long>, Event, String>.Context ctx, Collector<String> out)
                              throws Exception {
                          out.collect(right + "=>" + left);
                      }
                  }).print();
          env.execute();
      }
  }
  
  //结果输出
  Event [user=Mary, url=./home, timestamp=1970-01-01 00:00:01.0]=>(Mary,1000)
  Event [user=Blob, url=./prod?id=100, timestamp=1970-01-01 00:00:05.0]=>(Blob,3500)
  Event [user=Blob, url=./prod?id=101, timestamp=1970-01-01 00:00:00.3]=>(Blob,3500)
  Event [user=Alice, url=./cart, timestamp=1970-01-01 00:00:02.0]=>(Alice,4500)
  Event [user=Blob, url=./prod?id=100, timestamp=1970-01-01 00:00:05.0]=>(Blob,5500)
  Event [user=Alice, url=./pencil, timestamp=1970-01-01 00:00:06.0]=>(Alice,4500)
  Event [user=Alice, url=./pencil, timestamp=1970-01-01 00:00:06.0]=>(Alice,4500)
  Event [user=Alice, url=./pencil, timestamp=1970-01-01 00:00:06.0]=>(Alice,4500)
  Event [user=Blob, url=./prod?id=100, timestamp=1970-01-01 00:00:04.0]=>(Blob,3500)
  Event [user=Blob, url=./prod?id=100, timestamp=1970-01-01 00:00:04.0]=>(Blob,5500)
  ```

- **窗口同组Join (Window CoGroup)**

  和window join非常类似，都是将两条流合并之后开窗处理匹配的元素，调用时只需将`.join()`换成`.coGroup()`即可。

  ```java
  stream1.coGroup(stream2)
  	.where(<keySelector>)
  	.equalTo(<keySelector>)
  	.window(<WindowAssigner>)
  	.apply(<CoGroupFunction>)
  ```

  与window join的区别在于，调用`.apply()`方法定义具体操作时，传入的是一个`CoGroupFunction`。这也是一个函数类接口。

  **<font style="color:red">通过CoGroup不仅可以实现 inner join，还可以实现 left join 和 right join等操作</font>**

  示例:

  ```java
  package com.luke.flink.datastreamdemo;
  
  import java.time.Duration;
  
  import org.apache.flink.api.common.eventtime.SerializableTimestampAssigner;
  import org.apache.flink.api.common.eventtime.WatermarkStrategy;
  import org.apache.flink.api.common.functions.CoGroupFunction;
  import org.apache.flink.api.java.tuple.Tuple3;
  import org.apache.flink.api.java.tuple.Tuple4;
  import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  import org.apache.flink.streaming.api.windowing.assigners.TumblingEventTimeWindows;
  import org.apache.flink.streaming.api.windowing.time.Time;
  import org.apache.flink.util.Collector;
  
  public class StreamCoGroup {
      public static void main(String[] args) throws Exception {
          StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
          env.setParallelism(1);
  
          // 来自app的支付日志
          SingleOutputStreamOperator<Tuple3<String, String, Long>> appStream = env.fromElements(
                  Tuple3.of("order-1", "app", 1000L),
                  Tuple3.of("order-2", "app", 2000L),
                  Tuple3.of("order-3", "app", 3500L))
                  .assignTimestampsAndWatermarks(WatermarkStrategy
                          .<Tuple3<String, String, Long>>forBoundedOutOfOrderness(Duration.ZERO)
                          .withTimestampAssigner(new SerializableTimestampAssigner<Tuple3<String, String, Long>>() {
                              @Override
                              public long extractTimestamp(Tuple3<String, String, Long> element, long recordTimestamp) {
                                  return element.f2;
                              }
                          }));
  
          // 来自第三方支付平台的支付日志
          SingleOutputStreamOperator<Tuple4<String, String, String, Long>> thirdpartyStream = env.fromElements(
                  Tuple4.of("order-1", "third-party", "todo", 500L),
                  Tuple4.of("order-1", "third-party", "fail", 700L),
                  Tuple4.of("order-1", "third-party", "success", 3000L),
                  Tuple4.of("order-3", "third-party", "success", 4000L))
                  .assignTimestampsAndWatermarks(WatermarkStrategy
                          .<Tuple4<String, String, String, Long>>forBoundedOutOfOrderness(Duration.ZERO)
                          .withTimestampAssigner(
                                  new SerializableTimestampAssigner<Tuple4<String, String, String, Long>>() {
                                      @Override
                                      public long extractTimestamp(Tuple4<String, String, String, Long> element,
                                              long recordTimestamp) {
                                          return element.f3;
                                      }
                                  }));
  
          appStream.coGroup(thirdpartyStream)
                  .where(data -> data.f0)
                  .equalTo(data -> data.f0)
                  .window(TumblingEventTimeWindows.of(Time.seconds(5)))
                  .apply(new CoGroupFunction<Tuple3<String, String, Long>, Tuple4<String, String, String, Long>, String>() {
                      @Override
                      public void coGroup(Iterable<Tuple3<String, String, Long>> first,
                              Iterable<Tuple4<String, String, String, Long>> second, Collector<String> out)
                              throws Exception {
                          out.collect(first + "=>" + second);
                      }
                  }).print();
  
          env.execute();
      }
  }
  
  //结果输出
  [(order-1,app,1000)]=>[(order-1,third-party,todo,500), (order-1,third-party,fail,700), (order-1,third-party,success,3000)]
  [(order-3,app,3500)]=>[(order-3,third-party,success,4000)]
  [(order-2,app,2000)]=>[]
  ```

  
