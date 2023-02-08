##  状态编程

每个任务进行计算处理时候，可以基于当前数据直接转换得到输出结果；也可以依赖一些其他数据，这些由一个任务维护，并且可以用计算输出结果的所有数据，就叫做这个任务的状态。

#### 有状态算子
在Flink中，算子任务可以分为无状态和有状态两种情况。
无状态的算子任务只需要观察每个独立事件，根据当前输入的数据直接转换输出结果。例如可以将一个字符串类型的数据拆分开作为元组输出;也可以对数据做一些计算，比如每个代表数量的字段加1。
我们之前讲到的基本转换算子，如map、filter、 flatMap,计算时不依赖其他数据，就都属于无状态的算子。
比如sum、window等就是有状态的算子。

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221212000951566.png" alt="image-20221212000951566" style="zoom:50%;" />

#### 状态的分类

1. **托管zhuangt(Managed State) 和 原始状态(Raw State)**

   Flink的状态有两种:托管状态(Managed State)和原始状态(Raw State)。
   **托管状态就是由Flink统一管理的，状态的存储访问、故障恢复和重组等一系列问题都由Flink实现，我们只要调接口就可以;而原始状态则是自定义的，相当于就是开辟了一块内存，需要我们自己管理，实现状态的序列化和故障恢复**

   **具体来讲，托管状态是由Flink 的运行时(Runtime) 来托管的;在配置容错机制后，状态会自动持久化内存，并在发生故障时自动恢复。当应用发生横向扩展时，状态也会自动地重组分配到所有的子任务实例上**

   **对于具体的状态内容，Flink 也提供了值状态( ValueState)、列表状态(ListState)、 映射状态(MapState)、 聚合状态(AggregateState) 等多种结构，内部支持各种数据类型。聚合、窗口等算子中内置的状态，就都是托管状态;我们也可以在富函数类(RichFunction)中通过上下文来自定义状态，这些也都是托管状态**

   而对比之下，原始状态就全部需要自定义了。Flink 不会对状态进行任何自动操作，也不知道状态的具体数据类型，只会把它当作最原始的字节(Byte) 数组来存储。我们需要花费大量的精力来处理状态的管理和维护。

   所以只有在遇到托管状态无法实现的特殊需求时，我们才会考虑使用原始状态;一般情况;下不推荐使用。绝大多数应用场景，我们都可以用Flink提供的算子或者自定义托管状态来实现需求。

2. **算子状态(Operator State) 和 按键分区状态(Keyed State)**

   接下来我们的重点就是托管状态( Managed State)。

   **我们知道在Flink中，一个算子任务会按照并行度分为多个并行子任务执行，而不同的子任务会占据不同的任务槽(taskslot)。由于不同的slot在计算资源上是物理隔离的，所以Flink能管理的状态在并行任务间是无法共享的，每个状态只能针对当前子任务的实例有效**

   **而很多有状态的操作(比如聚合、窗口)都是要先做keyBy 进行按键分区的。按键分区之后，任务所进行的所有计算都应该只针对当前key有效,所以状态也应该按照key彼此隔离。在这种情况下，状态的访问方式又会有所不同**
   基于这样的想法，我们又可以将托管状态分为两类:算子状态和按键分区状态。

   - **算子状态(Operator State)**
     状态作用范围限定为当前的算子任务实例，也就是只对当前并行子任务实例有效。这就意味着对于一个并行子任务，占据了一个“分区”，它所处理的所有数据都会访问到相同的状态，状态对于同一任务而言是共享的。

     <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221212215240989.png" alt="image-20221212215240989" style="zoom:50%;" />

     **算子状态可以用在所有算子上，各个算子的算子状态独立变化、互不干扰，使用的时候其实就跟一个本地变量没什么区别**——因为本地变量的作用域也是当前任务实例。在使用时，我们还需要进一步实现CheckpointedFunction接口。

     **一个算子可能处理 多个数据，那么该算子上的算子状态 就是对所有这些数据可见、可用的**

   - **<font style="color:red">按键分区状态(Keyed State)</font>**

     状态是根据输入流中定义的key来维护和访问的，所以只能定义KeyedStream中，也就是keyBy之后才能使用。

     <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221212215956532.png" alt="image-20221212215956532" style="zoom:50%;" />**按键分区状态应用非常广泛。之前讲到的聚合算子必须在keyBy 之后才能使用，就是因为聚合的结果是以Keyed State 的形式保存的。另外，也可以通过富函数类(Rich Function)来自定义Keyed State，所以只要提供了富函数类接口的算子，也都可以使用Keyed State**

     所以即使是map、filter 这样无状态的基本转换算子，我们也可以通过富函数类给它们“追加”Keyed State，或者实现CheckpointedFunction接口来定义Operator State;从这个角度讲,Flink中所有的算子都可以是有状态的，不愧是“有状态的流处理”。

#### <font style="color:red">按键分区状态(Keyed State)</font>

- 按键分区(keyBy)之后，具有相同key的所有数据，都会分配到同一个并行子任务中;
- 如果当前任务定义了状态，Flink就会在当前并行子任务实例中，为每个key维护一个状态的实例;
- 于是当前任务就会为分配来的所有数据，按照key维护和处理这些状态;
- 一个并行子任务可能会处理多个key的数据，在底层，Keyed State类似于一个分布式的映射(map)数据结构;
- 当一条数据到来时，任务就会自动将状态中的访问范围限定为当前数据的key，并从map中读取出其对应的状态值。此时相同key的所有数据都会访问到相同的状态，不同key的状态之间是彼此隔离的；
- **另外，在应用的并行度改变时，状态也需要随之进行重组。不同key对应的Keyed State可以进一步组成所谓的键组(key groups)，每一组都对应着一个并行子任务**
- **键组是Flink重新分配Keyed State的单元，键组的数量就等于定义的最大并行度。当算子并行度发生改变时，KeyedState就会按照当前的并行度重新平均分配，保证运行时各个子任务的负载相同**
- **需要注意，使用Keyed State必须基于KeyedStream。没有进行keyBy分区的DataStream,即使转换算子实现了对应的富函数类，也不能通过运行时，上下文访问Keyed State**

##### 所支持的结构类型

1. **值状态(ValueState<T>)**

   访问和更新状态:

   - `T value()`: 获取当前状态的值
   - `update(T value)`: 对状态进行更新，传入的参数value就是要复写的状态值

   在具体使用时，为了让运行时上下文清楚到底是哪个状态，我们还需要创建一个"状态描述器"(StateDescriptor)来提供状态的基本信息。如源码中，`ValueState`的状态描述器构造方法如下:

   ```java
   public ValueStateDescriptor(String name, Class<T> typeClass) {
   	super(name,typeClass,null);
   }
   ```

   有了这个描述器，运行时环境就可以获取到状态的控制句柄(handler)了。

2. **列表状态(ListState)**

   将需要保存的数据，以列表(List) 的形式组织起来。在`ListState<T>`接口中同样有一个类型参数T，表示列表中数据的类型。
   ListState 也提供了一系列的方法来操作状态，使用方式与一般的List非常相似。

   - `Iterable<T> get()`:获取当前的列表状态，返回的是一个可迭代类型Iterable<T>;
   - `update(List<T> values)`:传入一个列表values，直接对状态进行覆盖;
   - `add(T value)`:在状态列表中添加一-个元素value;
   - `addAll(List<T> values)`:向列表中添加多个元素，以列表values 形式传入。

   类似地，ListState 的状态描述器就叫作ListStateDescriptor， 用法跟ValueStateDescriptor完全一致。

3. **映射状态(MapState)**

   把一些键值对(key-value) 作为状态整体保存起来，可以认为就是一组key-value映射的列表。对应的MapState<UK, UV>接口中，就会有UK、UV两个泛型，分别表示保存的key和value的类型。同样，MapState 提供了操作映射状态的方法，与Map的使用非常类似。

   - `UV get(UK key)`:传入一个key作为参数，查询对应的value值;
   - put(UK key, UV value):传入一个键值对， 更新key对应的value值;
   - `putAll(Map<UK, UV> map)`:将传入的映射map中所有的键值对，全部添加到映射状
   态中;
   - `remove(UK key)`:将指定key对应的键值对删除;
   - `boolean contains(UK key)`:判断是否存在指定的key,返回一个boolean值。

   另外，MapState 也提供了获取整个映射相关信息的方法:
   - `Iterable<Map.Entry<UK, UV>> entries()`: 获取映射状态中所有的键值对; .
   - `Iterable<UK> keys()`: 获取映射状态中所有的键(key),返回一个可迭代Iterable类型;
   - `Iterable<UV> values()`: 获取映射状态中所有的值(value)， 返回一个可迭代Iterable类型;
   - `boolean isEmpty()`: 判断映射是否为空，返回一个boolean值;

4. **归约状态ReducingState**

   类似于值状态(Value),不过需要对添加进来的所有数据进行归约，将归约聚合之后的值作为状态保存下来。

   `ReducintState<T>`这个接口调用的方法类似于`ListState`， 只不过它保存的只是一个聚合值，所以调用`.add()`方法时，不是在状态列表里添加元素，而是直接把新数据和之前的状态进行归约，并用得到的结果更新状态。

   归约逻辑的定义，是在归约状态描述器(ReducingStateDescriptor)中，通过传入一个归约函数(`ReduceFunction`)来实现的。
   这里的归约函数，就是我们之前介绍reduce聚合算子时讲到的ReduceFunction，所以状态类型跟输入的数据类型是一样的。

   ```
   public ReducingStateDescriptor (
   String name, ReduceFunction<T> reduceFunction, Class<T> typeClass) {. . .}
   ```

   这里的描述器有三个参数，其中第二个参数就是定义了归约聚合逻辑的`ReduceFunction`,另外两个参数则是状态的名称和类型。

5. **聚合状态(AggregatingState)**

   与归约状态非常类似，聚合状态也是一个值，用来保存添加进来的所有数据的聚合结果。

   与`ReducingState`不同的是，它的聚合逻辑是由在描述器中传入一个更加一般化的聚合函数(`AggregateFunction`)来定义的;这也就是之前我们讲过的`AggregateFunction`,里面通过一个累加器(`Accumulator`) 来表示状态，所以聚合的状态类型可以跟添加进来的数据类型完全不同，使用更加灵活。

   同样地，`AggregatingState()`接口调用方法也与`ReducingState`相同，调用`.add)`方法添加元素时，会直接使用指定的`AggregateFunction`进行聚合并更新状态。

简单使用示例:

````java
package com.luke.flink.datastreamdemo;

import java.time.Duration;

import org.apache.flink.api.common.eventtime.SerializableTimestampAssigner;
import org.apache.flink.api.common.eventtime.WatermarkStrategy;
import org.apache.flink.api.common.functions.AggregateFunction;
import org.apache.flink.api.common.functions.ReduceFunction;
import org.apache.flink.api.common.functions.RichFlatMapFunction;
import org.apache.flink.api.common.state.AggregatingState;
import org.apache.flink.api.common.state.AggregatingStateDescriptor;
import org.apache.flink.api.common.state.ListState;
import org.apache.flink.api.common.state.ListStateDescriptor;
import org.apache.flink.api.common.state.MapState;
import org.apache.flink.api.common.state.MapStateDescriptor;
import org.apache.flink.api.common.state.ReducingState;
import org.apache.flink.api.common.state.ReducingStateDescriptor;
import org.apache.flink.api.common.state.ValueState;
import org.apache.flink.api.common.state.ValueStateDescriptor;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.util.Collector;

public class StateTest {
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

        stream.keyBy(data -> data.user)
                .flatMap(new MyFlatFunc())
                .print();

        env.execute();
    }

    public static class MyFlatFunc extends RichFlatMapFunction<Event, String> {
        ValueState<Event> myValueState;
        ListState<Event> myListState;
        MapState<String, Long> myMapState;
        ReducingState<Event> myReducingState;
        AggregatingState<Event, String> myAggregatingState;

        @Override
        public void open(Configuration parameters) throws Exception {
            myValueState = getRuntimeContext().getState(new ValueStateDescriptor<Event>("val-state", Event.class));
            myListState = getRuntimeContext().getListState(new ListStateDescriptor<Event>("list-state", Event.class));
            myMapState = getRuntimeContext()
                    .getMapState(new MapStateDescriptor<String, Long>("map-state", String.class, Long.class));
            myReducingState = getRuntimeContext()
                    .getReducingState(new ReducingStateDescriptor<Event>("reduce-state", new ReduceFunction<Event>() {
                        @Override
                        public Event reduce(Event value1, Event value2) throws Exception {
                            return new Event(value1.user, value1.url, value2.timestamp); // 简单的用value2的timestamp来更新
                        }
                    }, Event.class));
            myAggregatingState = getRuntimeContext()
                    .getAggregatingState(
                            new AggregatingStateDescriptor<Event, Long, String>("aggregate-state",
                                    new AggregateFunction<Event, Long, String>() {
                                        @Override
                                        public Long createAccumulator() {
                                            return 0L;
                                        }

                                        @Override
                                        public Long add(Event value, Long accumulator) {
                                            return accumulator + 1;
                                        }

                                        @Override
                                        public String getResult(Long accumulator) {
                                            return "count: " + accumulator;
                                        }

                                        @Override
                                        public Long merge(Long a, Long b) {
                                            return null;
                                        }

                                    },
                                    Long.class));

        }

        @Override
        public void flatMap(Event value, Collector<String> out) throws Exception {
            // 访问和更新状态
            // System.out.println(myValueState.value());
            myValueState.update(value);

            myListState.add(value);

            myMapState.put(value.user, myMapState.get(value.user) == null ? 1 : myMapState.get(value.user));
            System.out.println("my map value:" + value.user + " " + myMapState.get(value.user));

            myAggregatingState.add(value);
            System.out.println("my agg state: " + myAggregatingState.get());

            myReducingState.add(value);
            System.out.println("reducing state:" + myReducingState.get());
        }
    }
}
// 输出
my map value:Mary 1
my agg state: count: 1
reducing state:Event [user=Mary, url=./prod?id=100, timestamp=2022-12-12 22:18:25.71]
my map value:Bob 1
my agg state: count: 1
reducing state:Event [user=Bob, url=./home, timestamp=2022-12-12 22:18:26.715]
my map value:Bob 1
my agg state: count: 2
reducing state:Event [user=Bob, url=./home, timestamp=2022-12-12 22:18:27.716]
my map value:Bob 1
my agg state: count: 3
reducing state:Event [user=Bob, url=./home, timestamp=2022-12-12 22:18:28.717]
my map value:Cary 1
my agg state: count: 1
reducing state:Event [user=Cary, url=. /home, timestamp=2022-12-12 22:18:29.718]
my map value:Bob 1
my agg state: count: 4
````

##### `ValueState<T>`使用案例

- 前面的统计 wordCount的案例中，直接调用reduce方法，则每来一条数据都打印一次结果，输出结果太多了；
- 我们想要的是 隔断时间打印一个结果，知道相关进度
- 能否使用window来完成呢？**<font style="color:red">window只能统计 window这个窗口时间范围内的数据，而不是全量数据</font>**
- **<font style="color:red">所以我们通过 周期定时器 来完成</font>**

```java
package com.luke.flink.datastreamdemo;

import java.time.Duration;

import org.apache.flink.api.common.eventtime.SerializableTimestampAssigner;
import org.apache.flink.api.common.eventtime.WatermarkStrategy;
import org.apache.flink.api.common.state.ValueState;
import org.apache.flink.api.common.state.ValueStateDescriptor;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.KeyedProcessFunction;
import org.apache.flink.util.Collector;

public class StatePeriodicPvExample {
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

        // 统计每个用户的pv,也就是每个用户从最开始访问了多少页面
        stream.keyBy(data -> data.user)
                .process(new PeriodicPvEvent()).print();

        env.execute();
    }

    public static class PeriodicPvEvent extends KeyedProcessFunction<String, Event, String> {
        // 定义状态,保存当前的pv统计值
        ValueState<Long> countState;
        ValueState<Long> timerState;

        @Override
        public void open(Configuration parameters) throws Exception {
            countState = getRuntimeContext().getState(new ValueStateDescriptor<Long>("count", Long.class));
            timerState = getRuntimeContext().getState(new ValueStateDescriptor<Long>("timer", Long.class));
        }

        @Override
        public void processElement(Event value, KeyedProcessFunction<String, Event, String>.Context ctx,
                Collector<String> out) throws Exception {
            // 每来一条数据,更新对应的count值
            Long count = countState.value();
            countState.update(count == null ? 1 : count + 1);

            // 如果没注册过定时器,则注册
            if (timerState.value() == null) {
                ctx.timerService().registerEventTimeTimer(value.timestamp + 10 * 1000L);
                timerState.update(value.timestamp + 10 * 1000L);
            }
        }

        @Override
        public void onTimer(long timestamp, KeyedProcessFunction<String, Event, String>.OnTimerContext ctx,
                Collector<String> out) throws Exception {
            out.collect(ctx.getCurrentKey() + " pv count: " + countState.value());
            // 清空状态
            timerState.clear();
            // 不能清空 countState,否则就和 window结果一样了
        }
    }
}
//结果
Alice pv count: 3
Bob pv count: 2
Cary pv count: 5
Mary pv count: 2
Alice pv count: 5
Bob pv count: 6
Cary pv count: 9
Mary pv count: 5
Bob pv count: 8
Alice pv count: 7
Mary pv count: 11
```

#### ListState 使用案例

使用ListState来实现`SELECT * FROM A INNER JOIN B WHERE A.id=B.id`

```java
public class StateTwoStreamJoinExample {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);

        SingleOutputStreamOperator<Tuple3<String, String, Long>> stream1 = env.fromElements(
                Tuple3.of("a", "stream-1", 1000L),
                Tuple3.of("b", "stream-1", 2000L),
                Tuple3.of("a", "stream-1", 3000L))
                .assignTimestampsAndWatermarks(WatermarkStrategy
                        .<Tuple3<String, String, Long>>forBoundedOutOfOrderness(Duration.ZERO)
                        .withTimestampAssigner(new SerializableTimestampAssigner<Tuple3<String, String, Long>>() {
                            @Override
                            public long extractTimestamp(Tuple3<String, String, Long> element, long recordTimestamp) {
                                return element.f2;
                            }
                        }));

        SingleOutputStreamOperator<Tuple3<String, String, Long>> stream2 = env.fromElements(
                Tuple3.of("a", "stream-2", 3000L),
                Tuple3.of("b", "stream-3", 4000L),
                Tuple3.of("a", "stream-1", 6000L))
                .assignTimestampsAndWatermarks(WatermarkStrategy
                        .<Tuple3<String, String, Long>>forBoundedOutOfOrderness(Duration.ZERO)
                        .withTimestampAssigner(new SerializableTimestampAssigner<Tuple3<String, String, Long>>() {
                            @Override
                            public long extractTimestamp(Tuple3<String, String, Long> element, long recordTimestamp) {
                                return element.f2;
                            }
                        }));
        // 自定义列表状态进行 全外连接
        stream1.keyBy(data -> data.f0)
                .connect(stream2.keyBy(data -> data.f0))
                .process(new CoProcessFunction<Tuple3<String, String, Long>, Tuple3<String, String, Long>, String>() {
                    ListState<Tuple2<String, Long>> list01;
                    ListState<Tuple2<String, Long>> list02;

                    public void open(org.apache.flink.configuration.Configuration parameters) throws Exception {
                        list01 = getRuntimeContext().getListState(new ListStateDescriptor<Tuple2<String, Long>>(
                                "stream1-list", Types.TUPLE(Types.STRING, Types.LONG)));
                        list02 = getRuntimeContext().getListState(new ListStateDescriptor<Tuple2<String, Long>>(
                                "stream2-list", Types.TUPLE(Types.STRING, Types.LONG)));
                    };

                    @Override
                    public void processElement1(Tuple3<String, String, Long> left,
                            CoProcessFunction<Tuple3<String, String, Long>, Tuple3<String, String, Long>, String>.Context ctx,
                            Collector<String> out) throws Exception {
                        // 获取另一条中的所有数据,配对输出
                        for (Tuple2<String, Long> right : list02.get()) {
                            out.collect(left + "=>" + right);
                        }
                        list01.add(Tuple2.of(left.f0, left.f2));
                    }

                    @Override
                    public void processElement2(Tuple3<String, String, Long> right,
                            CoProcessFunction<Tuple3<String, String, Long>, Tuple3<String, String, Long>, String>.Context ctx,
                            Collector<String> out) throws Exception {
                        for (Tuple2<String, Long> left : list01.get()) {
                            out.collect(left + "=>" + right);
                        }
                        list02.add(Tuple2.of(right.f0, right.f2));
                    }
                }).print();

        env.execute();
    }
}
// 结果输出
(a,1000)=>(a,stream-2,3000)
(b,2000)=>(b,stream-3,4000)
(a,stream-1,3000)=>(a,3000)
(a,1000)=>(a,stream-1,6000)
(a,3000)=>(a,stream-1,6000)
```

#### MapState 使用案例

使用mapState模拟 窗口操作，如下是 滚动窗口。
首先能否用 直接用 ListState 将某段时间内的数据全保存下来，然后设置定时器，定时器到了 将数据整体做一个计算？这个方法适合于 没有 延迟数据的情况；
使用MapState，key是窗口开始时间，value是这个窗口中的多个值 或者一个值，比如 count、sum。

```java
package com.luke.flink.datastreamdemo;

import java.sql.Timestamp;
import java.time.Duration;

import org.apache.flink.api.common.eventtime.SerializableTimestampAssigner;
import org.apache.flink.api.common.eventtime.WatermarkStrategy;
import org.apache.flink.api.common.state.MapState;
import org.apache.flink.api.common.state.MapStateDescriptor;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.KeyedProcessFunction;
import org.apache.flink.util.Collector;

public class StateFakeWindowExample {
    public static void main(String[] args) throws Exception {
        // 统计每个url，10秒时间内被访问次数
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);

        SingleOutputStreamOperator<Event> stream = env.addSource(new ClickLogs())
                .assignTimestampsAndWatermarks(WatermarkStrategy.<Event>forBoundedOutOfOrderness(Duration.ZERO)
                        .withTimestampAssigner(new SerializableTimestampAssigner<Event>() {
                            @Override
                            public long extractTimestamp(Event element, long recordTimestamp) {
                                return element.timestamp;
                            }
                        }));

        stream.print("input");
        stream.keyBy(data -> data.url)
                .process(new FakeWindowResult(10000L))
                .print();

        env.execute();
    }

    public static class FakeWindowResult extends KeyedProcessFunction<String, Event, String> {
        private Long windowSize;

        // 定义一个MapState, 用来保存每个窗口统计的count值
        MapState<Long, Long> windowUrlCount;

        public FakeWindowResult(Long windowSize) {
            this.windowSize = windowSize;
        }

        @Override
        public void open(Configuration parameters) throws Exception {
            windowUrlCount = getRuntimeContext()
                    .getMapState(new MapStateDescriptor<Long, Long>("window-count", Long.class, Long.class));
        }

        @Override
        public void processElement(Event value, KeyedProcessFunction<String, Event, String>.Context ctx,
                Collector<String> out) throws Exception {

            // 每来一条数据,根据时间戳判断属于哪个窗口(窗口分配器)
            Long windowStart = value.timestamp / windowSize * windowSize; // 如果window是10秒,这样就是整10秒的时间
            Long windowEnd = windowStart + windowSize;

            // 注册end-1的定时器
            ctx.timerService().registerEventTimeTimer(windowEnd - 1);

            // 更新状态,进行增量聚合
            if (windowUrlCount.contains(windowStart)) {
                Long count = windowUrlCount.get(windowStart);
                windowUrlCount.put(windowStart, count + 1L);
            } else {
                windowUrlCount.put(windowStart, 1L);
            }
        }

        // 定时器触发时输出结果
        @Override
        public void onTimer(long timestamp, KeyedProcessFunction<String, Event, String>.OnTimerContext ctx,
                Collector<String> out) throws Exception {
            Long windowEnd = timestamp + 1;
            Long windowStart = windowEnd - windowSize;

            Long count = windowUrlCount.get(windowStart);

            out.collect("窗口" + new Timestamp(windowStart) + "~" + new Timestamp(windowEnd)
                    + " url:" + ctx.getCurrentKey() + "  count:" + count);

            // 模拟窗口的关闭,清除 mapState中key-value
            windowUrlCount.remove(windowStart);
        }
    }
}

//结果输出
input> Event [user=Mary, url=./cart, timestamp=2022-12-13 14:37:09.835]
input> Event [user=Bob, url=./prod?id=10, timestamp=2022-12-13 14:37:10.839]
窗口2022-12-13 14:37:00.0~2022-12-13 14:37:10.0 url:./cart  count:1 //整数10秒才输出
input> Event [user=Alice, url=./cart, timestamp=2022-12-13 14:37:11.839]
input> Event [user=Mary, url=. /home, timestamp=2022-12-13 14:37:12.84]
input> Event [user=Cary, url=./cart, timestamp=2022-12-13 14:37:13.841]
input> Event [user=Bob, url=./prod?id=100, timestamp=2022-12-13 14:37:14.844]
input> Event [user=Mary, url=./prod?id=10, timestamp=2022-12-13 14:37:15.849]
input> Event [user=Alice, url=./prod?id=100, timestamp=2022-12-13 14:37:16.851]
input> Event [user=Alice, url=. /home, timestamp=2022-12-13 14:37:17.853]
input> Event [user=Alice, url=./prod?id=10, timestamp=2022-12-13 14:37:18.855]
input> Event [user=Mary, url=./cart, timestamp=2022-12-13 14:37:19.857]
input> Event [user=Bob, url=./prod?id=10, timestamp=2022-12-13 14:37:20.858]
窗口2022-12-13 14:37:10.0~2022-12-13 14:37:20.0 url:./prod?id=10  count:3
窗口2022-12-13 14:37:10.0~2022-12-13 14:37:20.0 url:./cart  count:3
窗口2022-12-13 14:37:10.0~2022-12-13 14:37:20.0 url:./prod?id=100  count:2
窗口2022-12-13 14:37:10.0~2022-12-13 14:37:20.0 url:. /home  count:2
```

#### ReducingState 使用案例

```java
package com.luke.flink.datastreamdemo;

import java.time.Duration;

import org.apache.flink.api.common.eventtime.SerializableTimestampAssigner;
import org.apache.flink.api.common.eventtime.WatermarkStrategy;
import org.apache.flink.api.common.functions.AggregateFunction;
import org.apache.flink.api.common.functions.RichFlatMapFunction;
import org.apache.flink.api.common.state.AggregatingState;
import org.apache.flink.api.common.state.AggregatingStateDescriptor;
import org.apache.flink.api.common.state.ValueState;
import org.apache.flink.api.common.state.ValueStateDescriptor;
import org.apache.flink.api.common.typeinfo.Types;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.util.Collector;

public class StateAverageTimestampExample {
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

        stream.print("input");

        // 自定义实现平均时间戳的统计
        stream.keyBy(data -> data.user)
                .flatMap(new AvgTsResult(5L))
                .print();

        env.execute();
    }

    // 实现自定义的RichFlatmapFunction
    public static class AvgTsResult extends RichFlatMapFunction<Event, String> {
        // 定义一个聚合状态,用于保存平均时间戳
        AggregatingState<Event, Long> avgTsAggState;
        // 每次到count次数时就开始输出平均时间戳
        private Long count;
        // 保存用户访问次数
        ValueState<Long> userTimesState;

        public AvgTsResult(Long count) {
            this.count = count;
        }

        @Override
        public void open(Configuration parameters) throws Exception {
            avgTsAggState = getRuntimeContext()
                    .getAggregatingState(new AggregatingStateDescriptor<Event, Tuple2<Long, Long>, Long>(
                            "avg-ts-state", new AggregateFunction<Event, Tuple2<Long, Long>, Long>() {

                                @Override
                                public Tuple2<Long, Long> createAccumulator() {
                                    return Tuple2.of(0L, 0L);
                                }

                                @Override
                                public Tuple2<Long, Long> add(Event value, Tuple2<Long, Long> accumulator) {
                                    return Tuple2.of(accumulator.f0 + value.timestamp, accumulator.f1 + 1L);
                                }

                                @Override
                                public Long getResult(Tuple2<Long, Long> accumulator) {
                                    return accumulator.f0 / accumulator.f1;
                                }

                                @Override
                                public Tuple2<Long, Long> merge(Tuple2<Long, Long> a, Tuple2<Long, Long> b) {
                                    return null;
                                }
                            }, Types.TUPLE(Types.LONG, Types.LONG)));
            userTimesState = getRuntimeContext().getState(new ValueStateDescriptor<Long>("user-times", Types.LONG));
        }

        @Override
        public void flatMap(Event value, Collector<String> out) throws Exception {
            // 每来一条数据, count +1
            Long currCount = userTimesState.value();
            if (currCount == null) {
                currCount = 1L;
            } else {
                currCount++;
            }

            // 更新状态
            userTimesState.update(currCount);
            avgTsAggState.add(value);

            // 如果达到count次数就输出结果
            if (currCount.equals(count)) {
                out.collect(value.user + "过去" + count + "次平均时间戳为" + avgTsAggState.get());
                // 清理状态
                userTimesState.clear();
                // 如果下面这一行不注释,则是每5秒输出 单个user的所有历史依赖的平均访问时间,
                // 因为 avgTsAggState中的 Tuple2<Long, Long> 是持续累加不清空的
                avgTsAggState.clear();
            }
        }
    }
}

//输出:
input> Event [user=Bob, url=./cart, timestamp=2022-12-13 15:11:08.474]
input> Event [user=Bob, url=./prod?id=100, timestamp=2022-12-13 15:11:09.478]
input> Event [user=Mary, url=./prod?id=100, timestamp=2022-12-13 15:11:10.483]
input> Event [user=Bob, url=./home, timestamp=2022-12-13 15:11:11.485]
input> Event [user=Alice, url=. /home, timestamp=2022-12-13 15:11:12.486]
input> Event [user=Alice, url=. /home, timestamp=2022-12-13 15:11:13.488]
input> Event [user=Mary, url=./prod?id=100, timestamp=2022-12-13 15:11:14.493]
input> Event [user=Cary, url=./cart, timestamp=2022-12-13 15:11:15.495]
input> Event [user=Bob, url=./home, timestamp=2022-12-13 15:11:16.503]
input> Event [user=Cary, url=./cart, timestamp=2022-12-13 15:11:17.504]
input> Event [user=Cary, url=./cart, timestamp=2022-12-13 15:11:18.505]
input> Event [user=Bob, url=. /home, timestamp=2022-12-13 15:11:19.506]
Bob过去5次平均时间戳为1670944273089
```

#### 状态生存时间TTL

**我们在创建状态时，可以给状态附加一个属性，也就是状态的 "失效时间"。状态创建的时候，设置失效时间=当前时间+ TTL;之后如果有对状态的访问和修改，我们可以再对失效时间进行更新;当设置的清除条件被触发时(比如，状态被访问的时候，或者每隔一段时间扫描一次失效状态)，就可以判断状态是否失效、从而进行清除了。**

配置状态的TTL时，需要创建一个`StateTtlConfig` 配置对象，然后调用状态描述器的`.enableTime'ToLive()`方法启动TTL功能。

```java
    public static class MyFlatFunc extends RichFlatMapFunction<Event, String> {
        ValueStateDescriptor<Event> valStateDescpter = new ValueStateDescriptor<Event>("val-state", Event.class);
        ValueState<Event> myValueState;
        
        @Override
        public void open(Configuration parameters) throws Exception {
            myValueState = getRuntimeContext().getState(valStateDescpter);
            
            // 配置状态的TTL, 基于处理时间
            StateTtlConfig ttlConfig = StateTtlConfig.newBuilder(Time.hours(1))
                    // 什么时候更新 状态的ttl 时间, 默认 读是不更新ttl的,只有写时候更新
                    .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)
                    // 配置状态的可见性: 对于一个已经过期还没clear的数据,是继续拿来用; 还是立马清理?
                    // 默认是 NeverReturnExpired
                    .setStateVisibility(StateTtlConfig.StateVisibility.ReturnExpiredIfNotCleanedUp)
                    .build();
            valStateDescpter.enableTimeToLive(ttlConfig);
        }
    }
```

#### 算子状态 Operator State

算子状态(OperatorState)就是一个算子并行实例上定义的状态，作用范围被限定为当前算子任务。算子状态跟数据的key无关,所以不同key的数据只要被分发到同一个并行子任务，就会访问到同一个Operator State。

**算子状态的实际应用场景不如Keyed State多, 一般用在Source或Sink等与外部系统连接的算子上，或者完全没有key定义的场景。比如Flink的Kafka连接器中，就用到了算子状态**

在我们给Source 算子设置并行度后，Kafka 消费者的每一个并行实例，都会为对应的主题(topic) 分区维护一个偏移量， 作为算子状态保存起来。这在保证Flink应用“精确一次”( exactly-once) 状态一 致性时非常有用。

**当算子的并行度发生变化时，算子状态也支持在并行的算子任务实例之间做重组分配。根据状态的类型不同，重组分配的方案也会不同**

##### 状态类型

主要由三种: `ListState`、`UnionListState`、`BroadcastState`

1. **列表状态 ListState**

   与Keyed State中的ListState一样，将状态表示为一组数据的列表。

   与Keyed State中的列表状态的区别是:在算子状态的上下文中，不会按键(key)分别处理状态，所以每一个并行子任务，上只会保留一个“列表”(list)，也就是当前并行子任务，上所有状态项的集合。列表中的状态项就是可以重新分配的最细粒度，彼此之间完全独立。
   当算子并行度进行缩放调整时，**算子的列表状态中的所有元素项会被统一收集起来，相当于把多个分区的列表合并成了一一个“大列表”，然后再均匀地分配给所有并行任务。这种“均匀分配”的具体方法就是“轮询”(round-robin)，与之前介绍的rebanlance数据传输方式类似**
   是通过逐一“发牌"的方式将状态项平均分配的。这种方式也叫作“平均分割重组”(even-split redistribution)。

   算子状态中不会存在“键组”(key group)这样的结构，所以为了方便重组分配，就把它直接定义成了“列表”(list)。这也就解释了，为什么算子状态中没有最简单的ValueState。

2. **联合列表状态 UnionListState**

   与ListState类似，联合列表状态也会将状态表示为一个列表。它与常规列表状态的区别在于，算子并行度进行缩放调整时对于状态的分配方式不同。

   **UnionListState的重点就在于“联合”(union)。在并行度调整时，常规列表状态是轮询分配状态项，而联合列表状态的算子则会直接广播状态的完整列表。这样，并行度缩放之后的并行子任务就获取到了联合后完整的“大列表”，可以自行选择要使用的状态项和要丢弃的状态项。这种分配也叫作“联合重组”(union redistribution)。如果列表中状态项数量太多，为资源和效率考虑一般不建议使用联合重组的方式**

3. **广播状态 BroadcastState**

   有时我们希望算子并行子任务都保持同一份“全局”状态,用来做统一的配置和规则设定。

   **这时所有分区的所有数据都会访问到同一个状态，状态就像被“广播”到所有分区一样，这种特殊的算子状态，就叫作广播状态( BroadcastState)。
   因为广播状态在每个并行子任务.上的实例都一样，所以在并行度调整的时候就比较简单，只要复制一份到新的并行任务就可以实现扩展;而对于并行度缩小的情况，可以将多余的并行子任务连同状态直接砍掉一-因为状态 都是复制出来的，并不会丢失**

   在底层，广播状态是以类似映射结构(map) 的键值对(key-value) 来保存的，必须基于一个“广播流" (BroadcastStream)来创建。关于广播流，后面介绍。

##### 代码实现

数据分区发生变化，带来的问题是: 怎么保证原先的状态跟故障恢复后数据的对应关系呢?

- 对于KeyedState这个问题很好解决:状态都是跟key相关的，而相同key的数据不管发往哪个分区，总是会全部进入一个分区的;
  于是只要将状态也按照key的哈希值计算出对应的分区，进行重组分配就可以了。
  恢复状态后继续处理数据，就总能按照key找到对应之前的状态，就保证了结果的一致性。所以Flink对Keyed State进行了非常完善的包装，我们不需实现任何接口就可以直接使用。
- 而对于Operator State来说就会有所不同。因为不存在key,所有数据发往哪个分区是不可预测的;也就是说，当发生故障重启之后，我们不能保证某个数据跟之前一样，进入到同一个并行子任务、访问同一个状态。所以Flink无法直接判断该怎样保存和恢复状态，而是提供了接口，让我们根据业务需求自行设计状态的快照保存( snapshot)和恢复(restore) 逻辑。

1. **CheckpointedFunction接口**

   在Flink中，对状态进行持久化保存的快照机制叫作“检查点" (Checkpoint)。 于是使用算子状态时，就需要对检查点的相关操作进行定义，实现一个CheckpointedFunction接口。

   CheckpointedFunction接口在源码中定义如下:

   ```java
   public interface CheckpointedFunction {
   	//保存状态快照到检查点时，调用这个方法
   	void snapshotState (FunctionSnapshotContext context) throws Exception
   	//初始化状态时调用这个方法，也会在恢复状态时调用
   	void initiali zeState (FunctionInitializationContext context) throws Exception;
   }
   ```

   每次应用保存检查点做快照时，都会调用`.snapshotState()`方法，将状态进行外部持久化。而在算子任务进行初始化时，会调用`.initializeState()`方法。
   这又有两种情况:一种是整个应用第一次运行，这时状态会被初始化为一个默认值(default value);另一种是应用重启时，从检查点(checkpoint)或者保存点(savepoint) 中读取之前状态的快照，并赋给本地状态。所以，**接口中的`.snapshotState()`方法定义了检查点的快照保存逻辑，而`.initializeState()`方法不仅定义了初始化逻辑，也定义了恢复逻辑**

   这里需要注意, `CheckpointedFunction`接口中的两个方法，分别传入了一个上下文(context)作为参数。

   不同的是，`.snapshotState()`方法 拿到的是快照的上下文`FunctionSnapshotContext`,它可以提供检查点的相关信息，**不过无法获取状态句柄**;

   而`.initializeState()`方法拿到的是`FunctionInitializationContext`，这是函数类进行初始化时的上下文，是真正的“运行时上下文”。

   `FunctionInitializationContext`中提供了“算子状态存储”(OperatorStateStore) 和“按键分区状态存储"(KeyedStateStore),在这两个存储对象中可以非常方便地获取当前任务实例中的Operator State和Keyed State。例如:

   ```java
   ListstateDescriptor<string> descriptor = new ListstateDescriptor<>("buf fered-elements",Types.of (String));
   
   Liststate<string> checkpointedstate = context.getOperatorStạteStore().getListstate (descriptor);
   ```

**示例代码**

下面例子中自定义的SinkFuntion会在`CheckpointedFunction`中进行数据缓存，然后统一发送给下游。这个例子演示了列表状态的平均分割重组(event-split redistribution)

```java
package com.luke.flink.datastreamdemo;

import java.time.Duration;
import java.util.ArrayList;
import java.util.List;

import org.apache.flink.api.common.eventtime.SerializableTimestampAssigner;
import org.apache.flink.api.common.eventtime.WatermarkStrategy;
import org.apache.flink.api.common.state.ListState;
import org.apache.flink.api.common.state.ListStateDescriptor;
import org.apache.flink.runtime.state.FunctionInitializationContext;
import org.apache.flink.runtime.state.FunctionSnapshotContext;
import org.apache.flink.streaming.api.checkpoint.CheckpointedFunction;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.sink.SinkFunction;

public class StateBufferingSinkExample {
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

        // stream.print("input:");

        stream.addSink(new BufferingSink(10));

        env.execute();
    }

    // 自定义实现SinkFunction
    public static class BufferingSink implements SinkFunction<Event>, CheckpointedFunction {
        // 定义当前类的属性,批量
        private final int threshold;
        private List<Event> bufferedElements;

        public BufferingSink(int threshold) {
            this.threshold = threshold;
            this.bufferedElements = new ArrayList<>();
        }

        // 定义一个算子状态, 和 bufferedElements 保存的是同样的数据
        // 只有在需要做checkpoint时才将 bufferedElements中数据保存到 checkpointedState 中
        private ListState<Event> checkpointedState;

        @Override
        public void invoke(Event value, Context context) throws Exception {
            bufferedElements.add(value);
            // 判断如果达到阈值,则批量写入
            if (bufferedElements.size() == threshold) {
                // 用控制台打印,模拟写入外部系统
                for (Event ele : bufferedElements) {
                    System.out.println(ele);
                }
                System.out.println("=====输出完毕=====");
                bufferedElements.clear();
            }
        }

        @Override
        public void snapshotState(FunctionSnapshotContext context) throws Exception {
            checkpointedState.clear();
            // 对状态进行持久化, 复制 bufferedElements 到 checkpointedState
            for (Event ele : bufferedElements) {
                checkpointedState.add(ele);
            }
        }

        @Override
        public void initializeState(FunctionInitializationContext context) throws Exception {
            // 定义算子
            ListStateDescriptor<Event> descripter = new ListStateDescriptor<>("buffered-elements", Event.class);
            checkpointedState = context.getOperatorStateStore().getListState(descripter);

            // 如果从故障恢复,需要将 checkpointedState 中数据恢复到 bufferedElements 中
            if (context.isRestored()) {
                for (Event ele : checkpointedState.get()) {
                    bufferedElements.add(ele);
                }
            }
        }

    }
}
```

#### 广播状态 Broadcast State

状态广播出去，所有并行子任务的状态都是相同的；并行度调整时只要直接复制即可。

**基本用法**

让所有并行子任务都持有同一状态，也就意味着一旦状态发生变化，所有子任务上的实例都要更新。

什么时候会用到 广播状态呢？

最普遍的情况是"动态配置" 或 "动态规则"。

**一个简单的想法是，定期扫描配置文件，发现改变就立即更新。但这样就需要另外启动一个扫描进程，如果扫描周期太长，配置更新不及时就会导致结果错误;如果扫描周期太短，又会耗费大量资源做无用功。解决的办法,还是流处理的“事件驱动”思路一一我们可以<font style="color:red">将这动态的配置数据看作一条流，将这条流和本身要处理的数据流进行连接(connect)， 就可以实时地更新配置进行计算了</font>**

**<font style="color:red">由于配置或者规则数据是全局有效的，我们需要把它广播给所有的并行子任务。而子任务需要把它作为一个算子状态保存起来，以保证故障恢复后处理结果是一致的</font>。这时的状态，就是一个典型的广播状态。我们知道，广播状态与其他算子状态的列表(list) 结构不同，底层是以键值对(key-value) 形式描述的，所以其实就是一个映射状态( MapState)**

代码上，可以直接调用`DataStream`的`.broadcast()`方法，传入一个"映射状态描述器" (MapStateDescripter)说明状态的名称和类型，就可以得到一个 广播流 (BroadcastStream)。进而将要处理的数据流和这条广播流进行连接(connect)，就会得到"广播连接流"(BroadcastConnectedStream)。

````java
// 状态描述器中key类型为string, 区分不同状态值而给定的key名称
MapStateDescriptor<string,Rule> ruleStateDescriptor = new MapStateDescriptor<>(...);
// ruleStream 中保存的就是 数据流stream的处理规则
// 通过 ruleStream.broadcast() 得到广播流
BroadcastStream<Rule> ruleBroadcastStream = ruleStream.broadcast(rulestateDescriptor);

DataStream<string> output = stream.connect(ruleBroadcaststream).process(new BroadcastProcessFunction<>() l. ..)) ;
````

对于广播流调用`.process()`方法，可以传入"广播处理函数" KeyedBroadcastProcessFunction 或者 BroadcastProcessFuntion来进行处理计算。广播处理函数中有两个方法`.processElement()`和`.processBroadcastElement()`
```java
// 处理正常的数据流,第一个参数value急速当前来的流数据
public  abstract void processElement (IN1 value,ReadOnlyContext ctx,Collector<ouT> out) throws Exception;

//处理广播流，第一个参数value就是广播流中的规则或者配置数据。
public abstract void processBroadcastElement (IN2 value, Context ctx, Collector<ouT> out) throws Exception;
```

第二个参数都一个ctx，都可以通过调用 .getBroadcastState()方法获取到当前的广播状态。
**区别在于.processElement()方法里的上下文是只读的Readonly，因此获取到的广播状态也只能读取不能更改**
**而.processBroadcastElement()中的 Context 没有限制，可以根据当前广播流中数据更新状态**

```java
Rule rule= ctx.getBroadcastState(new MapStateDescriptor<>("rules",Types.String,Types.Pojo(Rule.class))).get("my rule");
```

通过调用`ctx.getBroadcastState()`方法，传入一个`MapStateDescriptor`，就可以得到当前叫做`rules`的广播状态，调用它的`.get()`方法，就可以取出其中的`my rule`对应的值进行计算处理。

**示例代码**

电商应用中，往往需要判断用户先后发生的行为的"组合模式"，比如 "登录-下单" 或者 "登录-支付"，检测出这些连续行为进行统一，就可以了解平台的运用状状况以及用户的行为习惯。

```java
package com.luke.flink.datastreamdemo;

import org.apache.flink.api.common.state.BroadcastState;
import org.apache.flink.api.common.state.MapStateDescriptor;
import org.apache.flink.api.common.state.ReadOnlyBroadcastState;
import org.apache.flink.api.common.state.ValueState;
import org.apache.flink.api.common.state.ValueStateDescriptor;
import org.apache.flink.api.common.typeinfo.Types;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.datastream.BroadcastStream;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.co.KeyedBroadcastProcessFunction;
import org.apache.flink.util.Collector;

public class StateBehaviorPatternDetectExample {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);

        // 用户的行为数据流
        DataStreamSource<Action> actionStream = env.fromElements(
                new Action("Alice", "login"),
                new Action("Alice", "pay"),
                new Action("Bob", "login"),
                new Action("Bob", "order"));

        // 行为模式流，用于构建广播流
        DataStreamSource<Pattern> patternStream = env.fromElements(
                new Pattern("login", "pay"),
                new Pattern("login", "order"));

        // 定义广播状态描述器
        MapStateDescriptor<Void, Pattern> descriptor = new MapStateDescriptor<Void, Pattern>("pattern", Types.VOID,
                Types.POJO(Pattern.class));

        BroadcastStream<Pattern> broadcastStream = patternStream.broadcast(descriptor);

        // connect两条流进行处理
        actionStream.keyBy(data -> data.userId)
                .connect(broadcastStream)
                .process(new PatternDetector())
                .print();

        env.execute();
    }

    // 定义用户行为和模式的POJO类
    public static class Action {
        public String userId;
        public String action;

        public Action(String userId, String action) {
            this.userId = userId;
            this.action = action;
        }

        public Action() {
        }

        @Override
        public String toString() {
            return "Action [userId=" + userId + ", action=" + action + "]";
        }
    }

    // 行为模式
    public static class Pattern {
        public String action1;
        public String action2;

        @Override
        public String toString() {
            return "Pattern [action1=" + action1 + ", action2=" + action2 + "]";
        }

        public Pattern(String action1, String action2) {
            this.action1 = action1;
            this.action2 = action2;
        }

        public Pattern() {
        }
    }

    // 实现自定义的 keyedBroadcastProcessFunction
    // 第一个参数是 key类型, userId是string
    // 第二个参数是 第一条流的输入类型 Action
    // 第三个参数是 第二条流的输入类型 Pattern
    // 第四个参数是 输出类型 Tuple2<String,Pattern>
    public static class PatternDetector
            extends KeyedBroadcastProcessFunction<String, Action, Pattern, Tuple2<String, Pattern>> {

        // 定义一个 keydState, 保存上一次用户的行为
        ValueState<String> prevActionState;

        @Override
        public void open(Configuration parameters) throws Exception {
            prevActionState = getRuntimeContext()
                    .getState(new ValueStateDescriptor<String>("last-state", String.class));
        }

        @Override
        public void processElement(Action value,
                KeyedBroadcastProcessFunction<String, Action, Pattern, Tuple2<String, Pattern>>.ReadOnlyContext ctx,
                Collector<Tuple2<String, Pattern>> out) throws Exception {
            // 从广播状态中获取匹配模式
            // new MapStateDescriptor 这里写法和 main函数中是一样的
            ReadOnlyBroadcastState<Void, Pattern> readonlyPatternState = ctx
                    .getBroadcastState(new MapStateDescriptor<Void, Pattern>("pattern", Types.VOID,
                            Types.POJO(Pattern.class)));
            Pattern pattern = readonlyPatternState.get(null);

            // 获取用户上一次行为
            String prevAction = prevActionState.value();

            // 判断是否匹配
            if (pattern != null && prevAction != null) {
                if (pattern.action1.equals(prevAction) && pattern.action2.equals(value.action)) {
                    out.collect(new Tuple2<String, Pattern>(ctx.getCurrentKey(), pattern));
                }
            }
            // value.action中保存的是用户本次行为,更新状态
            prevActionState.update(value.action);
        }

        @Override
        public void processBroadcastElement(Pattern value,
                KeyedBroadcastProcessFunction<String, Action, Pattern, Tuple2<String, Pattern>>.Context ctx,
                Collector<Tuple2<String, Pattern>> out) throws Exception {
            // 从上下文中获取广播状态, 并用当前数据更新状态
            // new MapStateDescriptor 这里写法和 main函数中是一样的
            BroadcastState<Void, Pattern> patternState = ctx
                    .getBroadcastState(new MapStateDescriptor<Void, Pattern>("pattern", Types.VOID,
                            Types.POJO(Pattern.class)));

            // 更新当前的广播状态
            patternState.put(null, value);
        }

    }
}
//输出
(Alice,Pattern [action1=login, action2=pay])
(Bob,Pattern [action1=login, action2=order])
```

### 状态持久化 和 状态后端

**在Flink 的状态管理机制中，很重要的一个功能就是对状态进行持久化(persistence) 保存，这样就可以在发生故障后进行重启恢复。Flink 对状态进行持久化的方式，就是将当前所有分布式状态进行“快照”保存，写入一个“检查点”(checkpoint)或者保存点(savepoint)保存到外部存储系统中。具体的存储介质，一般是分布式文件系统( distributed file system)**

#### 检查点 checkpoint

**有状态流应用中的检查点(checkpoint)，其实就是所有任务的状态在某个时间点的一个快照(一份拷贝)**。
简单来讲，就是一次“存盘”，让我们之前处理数据的进度不要丢掉。在一个流应用程序运行时，Flink会定期保存检查点，在检查点中会记录每个算子的id和状态;如果发生故障，Flink 就会用最近一次成功保存的检查点来恢复应用的状态，重新启动处理流程, 就如同“读档”一样。

如果保存检查点之后又处理了一些数据，然后发生了故障，那么重启恢复状态之后这些数据带来的状态改变会丢失。为了让最终处理结果正确，我们还需要让源(Source)算子重新读取这些数据，再次处理一遍。 这就需要流的数据源具有“数据重放”的能力，一个典型的例子就是Kafka，我们可以通过保存消费数据的偏移量、故障重启后重新提交来实现数据的重放。
这是对“至少一次”(at least once)状态一致性的保证， 如果希望实现“精确一次"( exactly once)的一致性，还需要数据写入外部系统时的相关保证。关于这部分内容后续讨论。

默认情况下，检查点是被禁用的，需要在代码中手动开启。直接调用执行环境的 `.enableCheckpointing()`方法开启可以开启检查点。

```java
env.enableCheckpointing(1000L); //参数是毫秒
```

##### 状态后端 State Backends

检查点的保存离不开JobManager和TaskManager，以及外部存储系统的协调。在应用进行检查点保存时，首先会由JobManager 向所有TaskManager 发出触发检查点的命令;TaskManger收到之后，将当前任务的所有状态进行快照保存，持久化到远程的存储介质中; 完成之后向JobManager 返回确认信息。这个过程是分布式的，当JobManger 收到所有TaskManager的返回信息后，就会确认当前检查点成功保存，如图9-5 所示。而这一切工作的协调，就需要一个“专职人员”来完成。

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221214221628840.png" alt="image-20221214221628840" style="zoom:50%;" />

在Flink中，状态的存储、访问以及维护，都是由一个可插拔的组件决定的，这个组件就叫作状态后端(state backend)。状态后端主要负责两件事:一是本地的状态管理，二是将检查点(checkpoint)写入远程的持久化存储。

**状态后端的分类**

状态后端是一个“开箱即用”的组件，可以在不改变应用程序逻辑的情况下独立配置。

Flink中提供了两类不同的状态后端，一种是 “哈希表状态后端”(`HashMapStateBackend`)，另一种是 “内嵌RocksDB状态后端”(`EmbeddedRocksDBStateBackend`)。如果没有特别配置,系统默认的状态后端是`HashMapStateBackend`

- **哈希表状态后端 HashMapStateBackend**
  - 把状态存放在内存里;
  - 具体实现上，哈希表状态后端在内部会直接把状态当作对象(objects), 保存在Taskmanager的JVM堆(heap), 上;
  - 普通的状态， 以及窗口中收集的数据和触发器(triggers)， 都会以键值对(key-value) 的形式存储起来，所以底层是一个哈希表(HashMap)， 这种状态后端也因此得名。
  - 对于检查点的保存，一般是放在持久化的分布式文件系统( file system)中，也可以通过配置“检查点存储”(CheckpointStorage)来另外指定;
  - HashMapStateBackend是将本地状态全部放入内存的，这样可以获得最快的读写速度，使计算性能达到最佳;代价则是内存的占用。它适用于具有大状态、长窗口、大键值状态的作业，对所有高可用性设置也是有效的。
- **内嵌RocksDB状态后端 EmbeddedRocksDBStateBackend**
  - RocksDB是一种内嵌的key-value 存储介质，可以把数据持久化到本地硬盘。**配置 EmbeddedRocksDBStateBackend后，会将处理中的数据全部放入RocksDB数据库中，RocksDB 默认存储在TaskManager的本地数据目录里**
  - 与HashMapStateBackend 直接在堆内存中存储对象不同，这种方式下状态主要是放在RocksDB中的。数据被存储为序列化的字节数组(Byte Arrays),读写操作需要序列化/反序列化，因此状态的访问性能要差一些。 另外，因为做了序列化，key 的比较也会按照字节进行，而不是直接调用`.hashCode()`和`.equals()`方法。
  - 对于检查点，同样会写入到远程的持久化文件系统中。
  - **EmbeddedRocksDBStateBackend始终执行的是异步快照，也就是不会因为保存检查点而阻塞数据的处理;而且它还提供了<font style="color:red">增量式保存检查点</font>的机制，这在很多情况下可以大大提升保存效率**
  - **由于它会把状态数据落盘，而且支持增量化的检查点，所以在状态非常大、窗口非常长、键/值状态很大的应用场景中是一个好选择，同样对所有高可用性设置有效**

**如何选择正确的状态后端呢?**

HashMap和RocksDB两种状态后端最大的区别，就在于本地状态存放在哪里:前者是内存，后者是RocksDB。 在实际应用中，选择那种状态后端，主要是需要根据业务需求在处理性能和应用的扩展性上做-一个选择。
HashMapStateBackend是内存计算，读写速度非常快;但是，状态的大小会受到集群可用内存的限制，如果应用的状态随着时间不停地增长，就会耗尽内存资源。

而RocksDB是硬盘存储，所以可以根据可用的磁盘空间进行扩展，而且是唯一支持增量检查点的状态后端，所以它非常适合于超级海量状态的存储。不过由于每个状态的读写都需要做序列化/反序列化,r而且可能需要直接从磁盘读取数据，这就会导致性能的降低，平均读写性能要比HashMapStateBackend慢一个数量级。

**状态后端的配置**

不做配置时，应用程序使用的默认状态后端是 集群配置文件`flink-conf.yaml`中指定的，配置键名是`state.backend`。该配置对集群上运行的所有作业都有效，我们可以通过修改值来改变默认的状态后端。

我们也可以在代码中为当前作业单独配置状态后端，这个配置会覆盖集群配置文件的默认值。

如果不想用hashmap 也不想用 rocksdb，想自定义，那就需要定义一个实现了状态后端工厂类`StateBackendFactory`的类。

- 配置集群默认状态后端

  ```yaml
  state.backend: hashmap  or rocksdb
  state.checkpoints.dir: hdfs://namenode:40010/flink/checkpoints
  ```

- 为每一个作业单独配置状态后端

  ```java
          env.setStateBackend(new HashMapStateBackend());
      
      //添加依赖
      <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-statebackend-rocksdb_2.11</artifactId>
        <version>1.13.0</version>
      </dependency>
          env.setStateBackend(new EmbeddedRocksDBStateBackend());
  ```

  
