### 处理函数

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221204152433092.png" alt="image-20221204152433092" style="zoom:50%;" />

在更底层，我们可以不定义任何具体的算子(比如map, filter,或者 window), 而是提炼出一个统一的 "处理"(process)操作——它是左右转换算子的一个概括性表达，可以自定义处理逻辑，所以这一层接口就被叫作 "处理函数"(process function)。

- 我们之前学习的转换算子，一般只是针对某种具体操作来定义的，能够拿到的信息比较有限;

- 如果我们想要访问事件的时间戳，或者当前的水位线信息，都是完全做不到的;

- **跟时间相关的操作，目前我们只会用窗口来处理。而在很多应用需求中，要求我们对时间有更精细的控制，需要能够获取水位线，甚至要“把控时间”、定义什么时候做什么事，这就不是基本的时间窗口能够实现的了。这时就需要使用底层的处理函数(ProcessFunction)。
  处理函数提供了一一个“定时服务”(TimerService)，我们可以通过它访问流中的事件(event)、 时间戳( timestamp)、水位线( watermark)，甚至可以注册“定时事件”**

- **而且处理函数继承了AbstractRichFunction 抽象类，所以拥有富函数类的所有特性，同样可以访问状态(state) 和其他运行时信息。此外，处理函数还可以直接将数据输出到侧输出流(sideoutput)中。所以，处理函数是最为灵活的处理方法，可以实现各种自定义的业务逻辑;同时也是整个DataStreamAPI的底层基础**

- 处理函数的使用和基本的转换操作类似，只需要基于 DataStream调用`.process()`方法就可以了。方法需要传入一个`ProcessFunction`作为参数，用来定义处理逻辑

  ```java
  stream.process(new MyProcessFunction())
  ```

  其中ProcessFunction不是接口，而是一个抽象类，继承了`AbstractRichFunction`。所以所有的处理函数，都是富函数`RichFunction`,富函数可以调用的东西这里同样可以调用。

#### 处理函数的分类

Flink的处理函数是一个大家族，`ProcessFunction`只是其中一员。
DataStream在调用一些转换方法后，可能生成新的流类型:

- 调用 `.keyBy()`之后得到`KeyedStream`
- 再调用`.window()`之后得到`WindowStream`;

对于不同类型的流，其实都可以直接调用`.process()`方法进行自定义处理，这时传入的参数就都叫做处理函数。
他们尽管本质相同，都可以访问状态和时间信息的底层API，可彼此之间也会有所差异。

Flink 提供了8个不同的处理函数:

- **ProcessFunction**

  最基本的处理函数，基于`DataStream`直接调用`.process()`时作为参数传入;

- **<font style="color:red">KeyedProcessFunction</font>**

  调用`.keyBy()`之后的处理函数，基于`KeydStream`调用`.process()`时作为参数传入。

- **ProcessWindowFunction**

  调用`.window()`之后的处理函数，也是全窗口函数的代表。基于`WindowedStream`调用`.process()`时作为参数传入;

- **ProcessAllWindowFunction**

  同时是开窗之后的处理函数，基于`AllWindowedStream`调用`.process()`时作为参数传入。

- **CoProcessFunction**

  合并`connect`两条流之后的处理函数，基于`ConnectedStreams`调用`.process()`时作为参数传入。关于流的合并操作，后面介绍。

- **ProcesssJoinFunction**

  间接连接(interval join)两条流之后的处理函数，基于`IntervalJoined`调用`.process()`是作为参数传入。

- **BroadcastProcessFunction**

- **KeyedBroadcastProcessFunction**


#### ProcessFunction 解析

抽象类`ProcessFunction`继承了`AbstractRichFunction`。
有两个泛型类型参数: I 表是Input， 输入参数类型；O代表 Output，处理完成后输出数据类型。
两个方法: 一个是必须实现的抽象方法`processElement)`; 另一个是非抽象方法`.onTimer()`

```java
package com.luke.flink.datastreamdemo;
import java.time.Duration;
import org.apache.flink.api.common.eventtime.SerializableTimestampAssigner;
import org.apache.flink.api.common.eventtime.WatermarkStrategy;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.ProcessFunction;
import org.apache.flink.util.Collector;

public class ProcessFunctionTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        env.getConfig().setAutoWatermarkInterval(100); // 100毫秒发送watermark
        SingleOutputStreamOperator<Event> stream = env.addSource(new ClickLogs())
                // 乱序流的watermark生成
                .assignTimestampsAndWatermarks(WatermarkStrategy.<Event>forBoundedOutOfOrderness(Duration.ZERO)
                        .withTimestampAssigner(new SerializableTimestampAssigner<Event>() {
                            @Override
                            public long extractTimestamp(Event element, long recordTimestamp) {
                                return element.timestamp;
                            }
                        }));
        stream.process(new ProcessFunction<Event, String>() {
            @Override
            public void processElement(Event value, ProcessFunction<Event, String>.Context ctx, Collector<String> out)
                    throws Exception {
                // 简单的filter、map等操作都可以通过 ProcessFunction搞定
                if (value.user.equals("Mary")) {
                    out.collect(value.user + " clicks " + value.url);
                } else if (value.user.equals("Bob")) {
                    out.collect(value.user);
                    out.collect(value.user);
                }
                System.out.println("timestamp: " + ctx.timestamp()); //当前时间
                System.out.println("watermark: " + ctx.timerService().currentWatermark()); //水位线
            }

          	// onTimer()需要和 ctx.timerService().registerProcessingTimeTimer() 一起使用, 这里是用不了的
          	// 因为只有 KeyedProcessFunctiton 才能调用 timeService()
            @Override
            public void onTimer(long timestamp, ProcessFunction<Event, String>.OnTimerContext ctx,
                    Collector<String> out) throws Exception {
                super.onTimer(timestamp, ctx, out);
            }
        }).print();
        env.execute();
    }
}

// 输出
Event [user=Blob, url=. /home, timestamp=2022-12-04 22:46:28.585] // 这个是在 new Clicklogs()中,每生成一个Event打印的
timestamp: 1670193988585
watermark: -9223372036854775808 // 第一个event处理时, watermark还是 LONG.Min
  
Event [user=Cary, url=./home, timestamp=2022-12-04 22:46:29.586] // 这个是在 new Clicklogs()中,每生成一个Event打印的
timestamp: 1670193989586
watermark: 1670193988584 // 第二个event处理时,watermark是上一条数据的 timestamp -1ms
  
Event [user=Cary, url=./prod?id=100, timestamp=2022-12-04 22:46:30.588]
timestamp: 1670193990588
watermark: 1670193989585
  
Event [user=Mary, url=./home, timestamp=2022-12-04 22:46:31.589]
Mary clicks ./home //processElement()函数中遇到了 Mary
timestamp: 1670193991589
watermark: 1670193990587
```

ProcessFunction的缺点是 无法使用 `TimerService`设置定时器。

#### 按键分区处理函数 (KeyedProcessFunctiton)

只有在KeyedStream中才支持使用 TimerService设置定时器操作;
所以一般情况下，我们都是现做 keyBy分区后，再去定义处理操作，处理函数一般是`KeydProcessFunction`。

##### 定时器(Timer)和定时服务(TimerService)

- 定时器(timers)是处理函数中进行时间相关操作的主要机制;

- **在.onTimer()方法中可以实现定时处理的逻辑，而它能触发的前提，就是之前曾经注册过定时器、并且现在已经到了触发时间。注册定时器的功能，是通过上下文中提供的“定时服务”(TimerService)来实现的**

- 定时服务与当前运行的环境有关。前面已经介绍过，ProcessFunction 的上下文(Context)中提供了.timerService(方法，可以直接返回一个TimerService对象。

- **TimerService 是Flink关于时间和定时器的基础服务接口，包含以下六个方法**

  ```java
  //获取当前的处理时间
  long currentProcessingTime() ;
  
  //获取当前的水位线(事件时间)
  long currentWatermark() ;
  
  //注册处理时间定时器，当处理时间超过time时触发,然后调用.onTimer()方法
  void registerProcessingTimeTimer (long time) ;
  
  //注册事件时间定时器，当水位线超过time时触发,然后调用.onTimer()方法
  void registerEventTimeTimer (long time) ;
  
  //删除触发时间为time的处理时间定时器
  void deleteProcessingTimeTimer (long time) ;
  
  //删除触发时间为time的处理时间定时器
  void deleteEventT imeTimer (1ong time) ;
  ```

  6个方法可以分成两大类: 基于处理时间和基于事件时间。而对应的操作主要有三个:获取当前时间，注册定时器，以及删除定时器。

  需要注意，尽管处理函数中都可以直接访问TimerService，不过只有基于KeyedStream的处理函数，才能去调用注册和删除定时器的方法;未作按键分区的DataStream不支持定时器操作，只能获取当前时间。
  **TimerService会以键(key)和时间戳为标准，对定时器进行去重;也就是说对于每个key和时间戳，最多只有一个定时器，如果注册了多次，onTimer ()方法也将只被调用一次。这样一来，我们在代码中就方便了很多，可以肆无忌惮地对一个key注册定时器，而不用担心重复定义——因为一个时间戳上的定时器只会触发一次**

##### KeyedProcessFunction的使用

````java
package com.luke.flink.datastreamdemo;

import java.sql.Timestamp;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.KeyedProcessFunction;
import org.apache.flink.util.Collector;

public class ProcessingTimeTimerTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        DataStreamSource<Event> stream = env.addSource(new ClickLogs());
        stream.keyBy(data -> data.user)
                // 第一个参数是key类型,第二个是输入数据类型,第三个输出数据类型
                .process(new KeyedProcessFunction<String, Event, String>() {
                    @Override
                    public void processElement(Event value, KeyedProcessFunction<String, Event, String>.Context ctx,
                            Collector<String> out) throws Exception {
                        Long currTs = ctx.timerService().currentProcessingTime();
                        out.collect(ctx.getCurrentKey() + "数据到达,到达时间:" + new Timestamp(currTs));

                        // 注册一个10秒后的定时器
                        ctx.timerService().registerProcessingTimeTimer(currTs + 10 * 1000L);
                    }
                    @Override
                    public void onTimer(long timestamp,
                            KeyedProcessFunction<String, Event, String>.OnTimerContext ctx,
                            Collector<String> out) throws Exception {
                        out.collect(ctx.getCurrentKey() + "定时器触发,触发时间: " + new Timestamp(timestamp));
                    };
                }).print();

        env.execute();
    }
}

// 输出结果
Cary数据到达,到达时间:2022-12-05 22:15:56.866
Mary数据到达,到达时间:2022-12-05 22:15:57.876
Blob数据到达,到达时间:2022-12-05 22:15:58.888
Mary数据到达,到达时间:2022-12-05 22:15:59.899
Alice数据到达,到达时间:2022-12-05 22:16:00.812
Cary数据到达,到达时间:2022-12-05 22:16:01.82
Alice数据到达,到达时间:2022-12-05 22:16:02.828
Cary数据到达,到达时间:2022-12-05 22:16:03.833
Cary数据到达,到达时间:2022-12-05 22:16:04.844
Blob数据到达,到达时间:2022-12-05 22:16:05.846
Alice数据到达,到达时间:2022-12-05 22:16:06.854
Cary定时器触发,触发时间: 2022-12-05 22:16:06.866 //Cary第一条数据的定时器触发
Blob数据到达,到达时间:2022-12-05 22:16:07.862 
Mary定时器触发,触发时间: 2022-12-05 22:16:07.876 //Mary第一条数据的定时器触发
Alice数据到达,到达时间:2022-12-05 22:16:08.873
Blob定时器触发,触发时间: 2022-12-05 22:16:08.888
Alice数据到达,到达时间:2022-12-05 22:16:09.883
Mary定时器触发,触发时间: 2022-12-05 22:16:09.899 //Mary 10秒前数据的定时器触发
````

**根据EventTime**

````java
package com.luke.flink.datastreamdemo;

import java.sql.Timestamp;
import java.time.Duration;

import org.apache.flink.api.common.eventtime.SerializableTimestampAssigner;
import org.apache.flink.api.common.eventtime.WatermarkStrategy;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.KeyedProcessFunction;
import org.apache.flink.util.Collector;

public class ProcessingEventTimer {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        SingleOutputStreamOperator<Event> stream = env.addSource(new ClickLogs())
                // 乱序流的watermark生成
                .assignTimestampsAndWatermarks(WatermarkStrategy.<Event>forBoundedOutOfOrderness(Duration.ZERO)
                        .withTimestampAssigner(new SerializableTimestampAssigner<Event>() {
                            @Override
                            public long extractTimestamp(Event element, long recordTimestamp) {
                                return element.timestamp;
                            }
                        }));
        // 事件时间
        stream.keyBy(data -> data.user)
                // 第一个参数是key类型,第二个是输入数据类型,第三个输出数据类型
                .process(new KeyedProcessFunction<String, Event, String>() {
                    @Override
                    public void processElement(Event value, KeyedProcessFunction<String, Event, String>.Context ctx,
                            Collector<String> out) throws Exception {
                        Long currTs = ctx.timestamp();
                        out.collect(ctx.getCurrentKey() + "数据到达,时间戳:" + new Timestamp(currTs) + " 当前的waterMark:"
                                + ctx.timerService().currentWatermark());
                        // 注册一个10秒后的定时器
                        ctx.timerService().registerEventTimeTimer(currTs + 10 * 1000L);
                    }

                    @Override
                    public void onTimer(long timestamp,
                            KeyedProcessFunction<String, Event, String>.OnTimerContext ctx,
                            Collector<String> out) throws Exception {
                        out.collect(ctx.getCurrentKey() + "定时器触发,触发时间: " + new Timestamp(timestamp) + " 当前的waterMark:"
                                + ctx.timerService().currentWatermark());
                    };
                }).print();

        env.execute();
    }
}
// 输出
Cary数据到达,时间戳:2022-12-05 22:46:49.988 当前的waterMark:-9223372036854775808
Blob数据到达,时间戳:2022-12-05 22:46:50.991 当前的waterMark:1670280409987
Blob数据到达,时间戳:2022-12-05 22:46:51.992 当前的waterMark:1670280410990
Cary数据到达,时间戳:2022-12-05 22:46:52.993 当前的waterMark:1670280411991
Mary数据到达,时间戳:2022-12-05 22:46:53.994 当前的waterMark:1670280412992
Mary数据到达,时间戳:2022-12-05 22:46:54.994 当前的waterMark:1670280413993
Alice数据到达,时间戳:2022-12-05 22:46:55.995 当前的waterMark:1670280414993
Alice数据到达,时间戳:2022-12-05 22:46:56.996 当前的waterMark:1670280415994
Blob数据到达,时间戳:2022-12-05 22:46:57.996 当前的waterMark:1670280416995
Alice数据到达,时间戳:2022-12-05 22:46:58.997 当前的waterMark:1670280417995
Alice数据到达,时间戳:2022-12-05 22:46:59.998 当前的waterMark:1670280418996
Cary定时器触发,触发时间: 2022-12-05 22:46:59.988 当前的waterMark:1670280419997 // 只有waterMark到了，才会触发waterMark之前的Event
Alice数据到达,时间戳:2022-12-05 22:47:00.999 当前的waterMark:1670280419997
Blob定时器触发,触发时间: 2022-12-05 22:47:00.991 当前的waterMark:1670280420998
Mary数据到达,时间戳:2022-12-05 22:47:02.0 当前的waterMark:1670280420998
Blob定时器触发,触发时间: 2022-12-05 22:47:01.992 当前的waterMark:1670280421999
Mary数据到达,时间戳:2022-12-05 22:47:03.001 当前的waterMark:1670280421999
  
//最后的waterMark会是 Long.MAX, 这样方便触发所有的Event
````

##### 应用案例——TopN

需求: 统计最近10秒钟内最热门的url链接，且每5秒更新一次。
技术: 滑动窗口

**使用ProcessAllWindowFunction实现**

简单想法: 不区分url,将所有数据都收集起来，统一进行统计计算。所以无需keyBy, 直接通过datastream 开窗。

```java
package com.luke.flink.datastreamdemo;

import java.sql.Timestamp;
import java.time.Duration;
import java.util.ArrayList;
import java.util.Comparator;
import java.util.HashMap;

import org.apache.flink.api.common.eventtime.SerializableTimestampAssigner;
import org.apache.flink.api.common.eventtime.WatermarkStrategy;
import org.apache.flink.api.common.functions.AggregateFunction;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.windowing.ProcessAllWindowFunction;
import org.apache.flink.streaming.api.windowing.assigners.SlidingEventTimeWindows;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.streaming.api.windowing.windows.TimeWindow;
import org.apache.flink.util.Collector;

public class TopNExample_ProcessAllWindowFunction {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        env.getConfig().setAutoWatermarkInterval(100); // 和下面的createWatermarkGenerator有关系 ,100毫秒生成watermark
        SingleOutputStreamOperator<Event> stream = env.addSource(new ClickLogs())
                // 乱序流的watermark生成
                .assignTimestampsAndWatermarks(WatermarkStrategy.<Event>forBoundedOutOfOrderness(Duration.ZERO)
                        .withTimestampAssigner(new SerializableTimestampAssigner<Event>() {
                            @Override
                            public long extractTimestamp(Event element, long recordTimestamp) {
                                return element.timestamp;
                            }
                        }));

        // 直接开窗,收集所有数据排序
        stream.map(data -> data.url)
                .windowAll(SlidingEventTimeWindows.of(Time.seconds(10), Time.seconds(5)))
                .aggregate(new UrlHashMapCountAgg(), new UrlAllWindowResult())
                .print();

        env.execute();
    }

    // 实现自定义的增量聚合函数
    public static class UrlHashMapCountAgg
            implements AggregateFunction<String, HashMap<String, Long>, ArrayList<Tuple2<String, Long>>> {
        @Override
        public HashMap<String, Long> createAccumulator() {
            return new HashMap<String, Long>();
        }

        @Override
        public HashMap<String, Long> add(String value, HashMap<String, Long> accumulator) {
            Long cnt = 0L;
            if (accumulator.containsKey(value)) {
                cnt = accumulator.get(value);
            }
            cnt++;
            accumulator.put(value, cnt);
            return accumulator;
        }

        @Override
        public ArrayList<Tuple2<String, Long>> getResult(HashMap<String, Long> accumulator) {
            ArrayList<Tuple2<String, Long>> result = new ArrayList<>();
            for (String key : accumulator.keySet()) {
                result.add(Tuple2.of(key, accumulator.get(key)));
            }
            result.sort(new Comparator<Tuple2<String, Long>>() {
                @Override
                public int compare(Tuple2<String, Long> o1, Tuple2<String, Long> o2) {
                    if (o1.f1 > o2.f1) {
                        return -1;
                    } else if (o1.f1 == o2.f1) {
                        return 0;
                    } else {
                        return 1;
                    }
                }
            });
            return result;
        }

        @Override
        public HashMap<String, Long> merge(HashMap<String, Long> a, HashMap<String, Long> b) {
            return null;
        }
    }

    // 实现自定义全
  窗口函数,包装信息输出结果
    public static class UrlAllWindowResult
            extends ProcessAllWindowFunction<ArrayList<Tuple2<String, Long>>, String, TimeWindow> {
        @Override
        public void process(
                ProcessAllWindowFunction<ArrayList<Tuple2<String, Long>>, String, TimeWindow>.Context context,
                Iterable<ArrayList<Tuple2<String, Long>>> elements, Collector<String> out) throws Exception {

            ArrayList<Tuple2<String, Long>> ret01 = elements.iterator().next();
            StringBuilder result = new StringBuilder();
            result.append("----------------------\n");
            result.append("窗口结束时间: " + new Timestamp(context.window().getEnd()) + "\n");

            // 取list前2个,包装信息输出
            for (int i = 0; i < 2; i++) {
                Tuple2<String, Long> currTuple = ret01.get(i);
                String info = "No. " + (i + 1) + " " + "url:" + currTuple.f0 + "访问量: " + currTuple.f1 + "\n";
                result.append(info);
            }
            result.append("----------------------\n");
            out.collect(result.toString());
        }
    }
}

//输出结果:
Event [user=Cary, url=./prod?id=10, timestamp=2022-12-07 23:00:53.011]
Event [user=Cary, url=./home, timestamp=2022-12-07 23:00:54.014]
Event [user=Mary, url=./home, timestamp=2022-12-07 23:00:55.016]
----------------------
窗口结束时间: 2022-12-07 23:00:55.0
No. 1 url:./prod?id=10访问量: 1
No. 2 url:./home访问量: 1
----------------------

Event [user=Alice, url=./home, timestamp=2022-12-07 23:00:56.018]
Event [user=Alice, url=. /home, timestamp=2022-12-07 23:00:57.019]
Event [user=Blob, url=./cart, timestamp=2022-12-07 23:00:58.021]
Event [user=Mary, url=./home, timestamp=2022-12-07 23:00:59.023]
Event [user=Blob, url=. /home, timestamp=2022-12-07 23:01:00.024]
----------------------
窗口结束时间: 2022-12-07 23:01:00.0
No. 1 url:./home访问量: 4
No. 2 url:./cart访问量: 1
----------------------
```

#### 侧输出流(Side Output)

主流 之外可以有 支流，可以一条流产生多条流。
具体应用时, 只要在处理函数的`.processEelement()`或者`.onTimer()`方法中，调用上下文的`.output`方法就可以了。

```java
        OutputTag<String> outputTag = new OutputTag<String>("side") {
        };

        SingleOutputStreamOperator<Long> longStream = stream.process(new ProcessFunction<Event, Long>() {
            @Override
            public void processElement(Event value, ProcessFunction<Event, Long>.Context ctx, Collector<Long> out)
                    throws Exception {
                // timestamp 输出到主流中
                out.collect(value.timestamp);
                // url输出到侧输出流中
                ctx.output(outputTag, value.url);
            }
        });
        // 拿到侧输出流中的数据
        DataStream<String> sideOutput = longStream.getSideOutput(outputTag);
```

这里`output()`方法需要输入两个参数，一个是输出标签`OutputTag`，用于标识侧输出流，一般在`ProcessFunction`之外声明。第二个是要输出的数据。
如果想要获取这个侧输出流，可以基于处理之后的`DataStream`直接调用`.getSideOutput()`方法，传入对应的`OutputTag`，这个方式与窗口API中获取侧输出流完全一样。

```java
DataStream<String> sideOutput = longStream.getSideOutput(outputTag);
```

