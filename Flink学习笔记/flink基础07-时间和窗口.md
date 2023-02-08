### Flink中的时间和窗口

在流数据处理应用中，一个很重要、很常见的操作就是窗口计算。所谓窗口就是划定一个事件范围，就是时间窗；对在范围内的数据进行处理，就是窗口计算。

#### 时间定义

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221127225522197.png" alt="image-20221127225522197" style="zoom:50%;" />

两个非常重要的时间点:

- 数据产生时间，称为 事件时间;
- 数据真正被处理的时间，称为 处理时间;

#### 水位线(Watermark)

Flink默认是事件时间。在窗口处理过程中，Flink可以基于数据的时间戳，自定义一个 "逻辑时钟"。这个时钟的时间不会自动流水；它的时间进展，就是靠着新到数据的时间戳来推动的。比如接收到 8:00的事件，时间就是 8:00，接收到 9:00的事件，时间就是 9：00.

**什么是水位线?**

在Flink中,用来衡量事件时间(Event Time )进展的标记,就被称作“水位线”( Watermark)。
具体实现上，水位线可以看作一条特殊的数据记录，它是插入到数据流中的一 个标记点, 主要内容就是一个时间戳，用来指示当前的事件时间。而它插入流中的位置，就应该是在某个数据到来之后;这样就可以从这个数据中提取时间戳，作为当前水位线的时间戳了。

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221127231923030.png" alt="image-20221127231923030" style="zoom:50%;" />

**1. 有序流中的水位线**

在理想状态下，数据应该按照它们生成的先后顺序、排好队进入流中;也就是说，它们处理的过程会保持原先的顺序不变，遵守先来后到的原则。这样的话我们从每个数据中提取时间戳，就可以保证总是从小到大增长的，从而插入的水位线也会不断增长、事件时钟不断向前推进。
实际应用中，如果当前数据量非常大，可能会有很多数据的时间戳是相同的，这时每来一条数据就提取时间戳、插入水位线就做了大量的无用功。而且即使时间戳不同，同时涌来的数据时间差会非常小(比如几毫秒)，往往对处理计算也没什么影响。**所以为了提高效率，一般会每隔一段时间生成一一个水位线，这个水位线的时间戳，就是当前最新数据的时间戳，如图6-6所示。所以这时的水位线，其实就是有序流中的一个周期性出现的时间标记**

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221128062510207.png" alt="image-20221128062510207" style="zoom:50%;" />

**2. 乱序流中的水位线**

在分布式系统中，数据在节点间传输，会因为网络传输延迟的不确定性，导致顺序发生改变，这就是所谓的“乱序数据”。
这里所说的“乱序”(out-of-order),是指数据的先后顺序不一-致，主要就是基于数据的产生时间而言的。如图6-7所示，一个7秒时产生的数据，生成时间自然要比9秒的数据早;但是经过数据缓存和传输之后,处理任务可能先收到了9秒的数据,之后7秒的数据才姗姗来迟。这时如果我们希望插入水位线，来指示当前的事件时间进展，又该怎么做呢?

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221128063247141.png" alt="image-20221128063247141" style="zoom:50%;" />

解决思路也很简单:**我们还是靠数据来驱动，每来一个数据就提取它的时间戳、插入一个水位线。不过现在的情况是数据乱序，所以插入新的水位线时，要先判断一下时间戳是否比之前的大，否则就不再生成新的水位线，如图6-8所示。也就是说，只有数据的时间戳比当前时钟大，才能推动时钟前进，这时才插入水位线**

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221128063524944.png" alt="image-20221128063524944" style="zoom:50%;" />

如果考虑到大量数据同时到来的处理效率，我们同样可以周期性地生成水位线。这时只需要保存一下之前所有数据中的最大时间戳，需要插入水位线时，就直接以它作为时间戳生成新的水位线，如图所示:

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221128063933602.png" alt="image-20221128063933602" style="zoom:50%;" />

但是这样做会带来一个非常大的问题:**我们无法正确处理“迟到”的数据。为了让窗口能够正确收集到迟到的数据，我们也可以等上一段时间，比如2秒;也就是用当前已有数据的最大时间戳减去2秒，就是要插入的水位线的时间戳，如图6-10所示。这样的话，9秒的数据到来之后，事件时钟不会直接推进到9秒，而是进展到了7秒;必须等到11秒的数据到来之后，事件时钟才会进展到9秒，这时迟到数据也都已收集齐，0~9秒的窗口就可以正确计算结果了。但是如果时间窗已经是 w(9)，现在再来1秒的数据，那么1秒的数据记录就会丢失。**

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221128064600048.png" alt="image-20221128064600048" style="zoom:50%;" />

**3. 水位线的特性**
现在我们可以知道，水位线就代表了当前的事件时间时钟，而且可以在数据的时间戳基础上加一些延迟来保证不丢数据，这一点对于乱序流的正确处理非常重要。
总结一下水位线的特性:

- **水位线 是插入到数据流中的一一个标记，可以认为是一个特殊的数据**
- **水位线主要的内容是一个时间戳，用来表示当前事件时间的进展**
- 水位线是基于数据的时间戳生成的
- **水位线的时间戳必须单调递增，以确保任务的事件时间时钟一直向前推进**
- **水位线可 以通过设置延迟，来保证正确处理乱序数据**
- 一个水位线Watermark(t),表示在当前流中事件时间已经达到了时间戳t,这代表t之前的所有数据都到齐了，之后流中不会出现时间戳t'≤t的数据
水位线是Flink流处理中保证结果正确性的核心机制，它往往会跟窗口一起配合，完成对乱序数据的正确处理。

水位线是Flink流处理中保证结果正确性的核心机制，它往往会跟窗口一起配合，完成对乱序数据的正确处理。

**4. 如何生成水位线**

水位线是用来保证窗口处理结果的正确性的，如果不能正确处理所有乱序数据，可以尝试调大延迟的时间。

- 生成水位线的总体原则

  **完美的水位线是“绝对正确”的，也就是一个水位线一旦出现， 就表示这个时间之前的数据已经全部到齐、之后再也不会出现了。不过如果要保证绝对正确，就必须等足够长的时间，这会带来更高的延迟。**
  如果我们希望计算结果能更加准确，那可以将水位线的延迟设置得更高一些,等待的时间越长，自然也就越不容易漏掉数据。不过这样做的代价是处理的实时性降低了，我们可能为极少数的迟到数据增加了很多不必要的延迟。
  如果我们希望处理得更快、实时性更强,那么可以将水位线延迟设得低一些。这种情况下，可能很多迟到数据会在水位线之后才到达，就会导致窗口遗漏数据，计算结果不准确。当然，如果我们对准确性完全不考虑、一味地追求处理速度，可以直接使用处理时间语义，这在理论上可以得到最低的延迟。
  **所以Flink中的水位线，其实是流处理中对低延迟和结果正确性的一个权衡机制，而且把控制的权力交给了程序员，我们可以在代码中定义水位线的生成策略。**

- 水位线生成策略(Watermark Stategies)

  在Flink的 DataStream API中，有一个单独用于生成水位线的方法: `.assignTimestampsAndWatermarks()`，它主要用来为流中的数据分配时间戳，并生成水位线来指示事件时间。

  ```java
  public SingleOutputStreamOperator<T> assignTimestampsAndWatermarks(WatermarkStrategy<T> watermarkStrategy)
  ```

  具体使用时，直接用 DataStream调用方法即可，与普通的transform方法完全一样。
  ```java
  DataStream<Event> stream = env.addSource(new ClickSource);
  DataStream<Event> withTimestampAndWatermarks = stream.assignTimestampsAndWatermarks(<watermark strategy>);
  ```

- **自定义水位线策略**

  **在WatermarkStrategy中，时间戳分配器TimestampAssigner都是大同小异的，指定字段提取时间戳就可以了;而不同策略的关键就在于WatermarkGenerator 的实现。**

  **整体说来，Flink有两种不同的生成水位线的方式:一种是周期性的(Periodic),另-种是断点式的(Punctuated)**

  **WatermarkGenerator接口中有两个方法，onEvent()和onPeriodicEmit(),前者是在每个事件到来时调用，而后者由框架周期性调用。周期性调用的方法中发出水位线，自然就是周期性生成水位线;而在事件触发的方法中发出水位线，自然就是断点式生成了。两种方式的不同就集中体现在这两个方法的实现上**

  没听懂????
  
  ```java
          // 有序流的watermark 生成
          stream.assignTimestampsAndWatermarks(WatermarkStrategy.<Event>forMonotonousTimestamps()
                  .withTimestampAssigner(new SerializableTimestampAssigner<Event>() {
                      @Override
                      public long extractTimestamp(Event element, long recordTimestamp) {
                          return element.timestamp;
                      }
                  }));
          // 乱序流的watermark生成,延迟2秒
  				// withTimestampAssigner()方法我理解的意思就是 抽取event.timestamp 作为水位线
          stream.assignTimestampsAndWatermarks(WatermarkStrategy.<Event>forBoundedOutOfOrderness(Duration.ofSeconds(2))
                  .withTimestampAssigner(new SerializableTimestampAssigner<Event>() {
                      @Override
                      public long extractTimestamp(Event element, long recordTimestamp) {
                          return element.timestamp;
                      }
                  }));
  
  		// 自定义实现周期性发送水位线的例子(自定义水位线生成):
  		// 等同于 <Event>forBoundedOutOfOrderness(Duration.ofSeconds(5)
      public static class CustomWatermarkStrategy implements WatermarkStrategy<Event> {
          @Override
          public WatermarkGenerator<Event> createWatermarkGenerator(WatermarkGeneratorSupplier.Context context) {
              return new CustomBoundedOutOfOrdernessGenerator();
          }
  
          @Override
          public TimestampAssigner<Event> createTimestampAssigner(TimestampAssignerSupplier.Context context) {
              return new SerializableTimestampAssigner<Event>() {
                  @Override
                  public long extractTimestamp(Event element, long recordTimestamp) {
                      return element.timestamp; // 告诉程序数据源里的时间戳是哪一个字段
                  }
              };
          }
      }
  
      public static class CustomBoundedOutOfOrdernessGenerator implements WatermarkGenerator<Event> {
          private Long delayTime = 5000L; // 延迟时间 5秒
          private Long maxTs = -Long.MAX_VALUE + delayTime + 1L; // 观察到的最大时间戳
  
          @Override
          public void onEvent(Event event, long eventTimestamp, WatermarkOutput output) {
              // 每来一条数据就调用一次, 更新最大时间戳
            	// 运行水位线 examine and remember event timestamps , or emit a watermark based on the event itself
              maxTs = Math.max(event.timestamp, maxTs);
          }
  
          @Override
          public void onPeriodicEmit(WatermarkOutput output) {
              // 发送水位线,默认200ms调用一次
            	// called periodically, and might emit a new watermark, or not.
            	// <p>The interval in which this method is called and Watermarks are generated
  						// depends on {@link ExecutionConfig#getAutoWatermarkInterval()}.
              output.emitWatermark(new Watermark(maxTs - delayTime - 1L));
          }
      }
  ```

**5. 水位线的传递**

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221129063809654.png" alt="image-20221129063809654" style="zoom:50%;" />

**在流处理中，上有任务处理完水位线、时钟改变之后，要把当前的水位线再次发出，广播给所有的下游子任务。而当一个任务接收到多个上游并行任务传递来的水位线时，应当以最小的那个作为当前任务的事件时钟**

如上图，当前任务的上游，有4个并行子任务，所以会接收到来自4个分区的水位线；而下游有三个并行子任务，所以会向3个分区发出水位线。

#### 窗口(window)

- 一般真实的流都是无界的，怎样处理无界的数据？
- 可以把无限的数据流进行切分，得到有限的数据集进行处理 —— 也就是得到有界流
-  **窗口（ window）就是将无限流切割为有限流的一种方式，它会将流数据分发到有限大小的桶（ bucket）中进行分析，当到达窗口结束时间时，就对每个同中手机的数据进行计算处理**
-  窗口是可以同时有多个的

*举例子：假设按照时间段划分桶，接收到的数据马上能判断放到哪个桶，且多个桶的数据能并行被处理。（迟到的数据也可判断是原本属于哪个桶的）*

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221129070015624.png" alt="image-20221129070015624" style="zoom:50%;" />

##### 窗口的分类

- **TimeWindow 时间窗口**

- **CountWindow 计数窗口**

- **按照窗口分配数据的规则分类**
  - 滚动窗口(Tumbling Windows)
    - 固定大小，对数据 均匀切片
    - 窗口之间没重叠，也不会间隔，是首尾相连状态
    - 可基于时间定义，也可基于数据个数定义; 
    - 需要的参数就一个 就是窗口的大小，如定义一个长度1小时的滚动窗口，那么每小时进行一次统计；or 定义一个长度为10的滚动计数窗口，就会每10条数据进行一次统计；
    
  - 滑动窗口(Sliding Windows)
  
    - 与滚动窗口类似，滑动窗口大小也固定。区别在于窗口之间并非收尾相接，而是可以 "错开" 一定的位置;
  
    - 定义滑动窗口的参数有两个: 除去窗口的大小(window size)之外，还有一个 "滑动步长"(window slide)，它其实就代表了窗口计算的频率。
  
    - 滑动的距离代表了下个窗口开始的时间间隔，而窗口大小是固定的，所以也就就是两个窗口结束的时间间隔；
  
    - 窗口在结束时触发计算输出结果，那么滑动步长就代表了计算频率;
  
    - 可以有重叠(是否重叠和滑动距离有关系)
  
    - 滑动窗口是固定窗口的更广义的一种形式，滚动窗口可以看做是滑动窗口的一种特殊情况（即窗口大小和滑动步长相等）
  
      <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221129223636690.png" alt="image-20221129223636690" style="zoom:50%;" />
  
  - 会话窗口(Session Windows)
  
    - 会话窗口最重要的参数就是会话的超时时间，也就是两个会话窗口之间的最小距离；
  
    - 如果相邻的两个数据到来的时间间隔(gap) 小于 指定的窗口大小(size)，那说明还在保持会话，属于同一个窗口；否则gap 大于 size，那么新来的数据属于新的会话窗口，前一个窗口关闭了；
  
    - 具体是线上，可设置静态固定的大小(size)，也可以通过一个自定义的提取器(gap extractor)动态提取最小间隔gap的值;
  
      <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221129225033880.png" alt="image-20221129225033880" style="zoom:50%;" />
  
  - 全局窗口(Global Window)
  
    - 将相同key的所有数据都分配到一个窗口中；
    - 该窗口没有结束的时候，默认不会做触发计算；
    - 如果希望它能对数据进行计算处理，还需要自定义 "触发器" (Trigger)
  
    <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221129230707053.png" alt="image-20221129230707053" style="zoom:50%;" />

#### 窗口API 概览

##### 按键分区(keyed) 和 非按键分区(Non-keyed)

在定义窗口操作之前，首先需要确定,到底是基于按键分区(Keyed)的数据流KeyedStream来开窗，还是直接在没有按键分区的DataStream. 上开窗。也就是说，在调用窗口算子之前，是否有keyBy操作。

(1) **按键分区窗口(Keyed Windows)**

- 经过按键分区keyBy操作后，数据流会按照key被分为多条逻辑流(logical streams)，这就是KeyedStream;

- 基于KeyedStream进行窗口操作时,窗口计算会在多个并行子任务上同时执行;

- 相同key的数据会被发送到同一个并行子任务，而窗口操作会基于每个key进行单独的处理。所以可以认为，每个key上都定义了一组窗口，各自独立地进行统计计算。

- 在代码实现.上，我们需要先对DataStream 调用.keyBy()进行按键分区，然后再调用`.window()`定义窗口;

  ```java
  stream.keyBy(...).window(...)
  ```

(2) **非按键分区(Non-Keyed Windows)**

- 如果没有进行keyBy，那么原始的DataStream 就不会分成多条逻辑流。这时窗口逻辑只能在一个任务(task) 上执行， 就相当于并行度变成了1。所以在实际应用中一般不推荐使用这种方式。

- 在代码中，直接基于DataStream调用.windowAll()定义窗口

  ```java
  stream.windowAll(...)
  ```

  这里需要注意的是，对于非按键分区的窗口操作，手动调大窗口算子的并发度也是无效的，windowAll本身就是非并行的操作。

##### 代码中窗口API的调用

窗口操作分为两个部分: 窗口分配器(Window Assigners) 和 窗口函数(Windows Functions)

```java
stream.keyBy(<key selector>).window(<window assigner>).aggregate(<window function>);
```

其中`.window()`方法需要传入一个窗口分配器，它指明了窗口的类型；而后面的`.aggregate()`方法传入一个窗口函数作为参数，它用来定义窗口具体的处理逻辑。

#### 窗口分配器 (Window Assigners)

定义窗口分配器(Window Assigners)是构建窗口算子的第一步，它的作用就是定义数据应该被"分配"到哪个窗口。所以说，窗口分配器其实就是指定窗口的类型。

- `.window()`方法 定义窗口分配器，传入一个 `WindowAssigner`作为参数，返回一个`WindowStream`;

- 如果非按键分区窗口，则直接调用`.windowAll()`方法，同样传入一个`WindowAssigner`，返回的是`AllWindowedStream`;

- 窗口按照驱动类型可分为 时间窗口 和 计数窗口，而按照具体的分配规则，又有滚动窗口、滑动窗口、会话窗口、全局窗口四种。组合起来就有 滚动事件时间窗口、滑动事件时间窗口、滚动事件计数窗口、滑动事件计数窗口等；

- 滚动事件时间窗口

  ```java
  stream.keyBy(...).window(TumblingEventTimeWindows.of(Time.hours(1))); //滚动事件时间窗口
  ```

- 滑动事件事件窗口

  ```java
  .window(SlidingEventTimeWindows.of(Time.hours(1), Time.minutes(5))); // 滑动事件时间窗口
  ```

- 会话窗口(session window)

  ```java
  .window(EventTimeSessionWindows.withGap(Time.seconds(2))); // 会话窗口
  ```

- 滚动计数窗口(tumbling count window)

  ```java
  .countWindow(5)
  ```

- 滑动计数窗口

  ```java
  .countWindow(10, 2); // 滑动计数窗口
  ```

*DataStream的`windowAll()`类似分区的global操作，这个操作是non-parallel的(并行度强行为1)，所有的数据都会被传递到同一个算子operator上，官方建议如果非必要就不要用这个API*

#### TimeWindow(时间窗口)

 TimeWindow将指定时间范围内的所有数据组成一个window，一次对一个window里面的所有数据进行计算。

- 滚动窗口

  TimeWindow将指定时间范围内的所有数据组成一个window，一次对一个window里面的所有数据进行计算。

  ```java
  // 15秒内的最低温度
  DataStream<Tuple2<String, Double>> minTempPerWindowStream = dataStream 
    .map(new MapFunction<SensorReading, Tuple2<String, Double>>() { 
      @Override 
      public Tuple2<String, Double> map(SensorReading value) throws Exception {
        return new Tuple2<>(value.getId(), value.getTemperature()); 
      } 
    }) 
    .keyBy(data -> data.f0) 
    .timeWindow( Time.seconds(15) ) 
    .minBy(1);
  ```

- 滑动窗口

  ```java
  // sliding_size设置为了5s，也就是说，每5s就计算输出结果一次，每一次计算的window范围是最近15s内的所有元素
  DataStream<SensorReading> minTempPerWindowStream = dataStream 
    .keyBy(SensorReading::getId) 
    .timeWindow( Time.seconds(15), Time.seconds(5) ) 
    .minBy("temperature");
  ```

#### CountWindow(计数窗口)

CountWindow根据窗口中相同key元素的数量来触发执行，执行时只计算元素数量达到窗口大小的key对应的结果。
 **<font style="color:red">注意：CountWindow的window_size指的是相同Key的元素的个数，不是输入的所有元素的总数</font>**

- 滚动窗口

   默认的CountWindow是一个滚动窗口，只需要指定窗口大小即可，**当元素数量达到窗口大小时，就会触发窗口的执行**。

  ```java
  DataStream<SensorReading> minTempPerWindowStream = dataStream 
    .keyBy(SensorReading::getId) 
    .countWindow( 5 ) 
    .minBy("temperature");
  ```

- 滑动窗口

  ```java
  // 下面代码中的sliding_size设置为了2，也就是说，每收到两个相同key的数据就计算一次，每一次计算的window范围是10个元素
  DataStream<SensorReading> minTempPerWindowStream = dataStream 
    .keyBy(SensorReading::getId) 
    .countWindow( 10, 2 ) 
    .minBy("temperature");
  ```

### window function

Window function 定义了要对窗口收集的数据做计算操作，主要分为两类:

- 增量聚合函数(incremental aggregation functions)

  - **每条数据到来就进行计算**，保持一个简单的状态。（来一条处理一条，但是不输出，到窗口临界位置才输出）

  - 典型的增量聚合函数有ReduceFunction, AggregateFunction。

  - 如 统计每个用户 10秒内点击次数

    ```java
            SingleOutputStreamOperator<Event> stream = env.addSource(new ClickLogs())
                    // 乱序流的watermark生成
                    .assignTimestampsAndWatermarks(WatermarkStrategy.<Event>forBoundedOutOfOrderness(Duration.ofSeconds(2))
                            .withTimestampAssigner(new SerializableTimestampAssigner<Event>() {
                                @Override
                                public long extractTimestamp(Event element, long recordTimestamp) {
                                    return element.timestamp;
                                }
                            }));       
    SingleOutputStreamOperator<Tuple2<String, Long>> userClickCount = stream
                    .map(new MapFunction<Event, Tuple2<String, Long>>() {
                        @Override
                        public Tuple2<String, Long> map(Event value) throws Exception {
                            return Tuple2.of(value.user, 1L);
                        }
                    }).keyBy(data -> data.f0)
                    .window(TumblingEventTimeWindows.of(Time.seconds(10)))
                    .reduce(new ReduceFunction<Tuple2<String, Long>>() {
                        @Override
                        public Tuple2<String, Long> reduce(Tuple2<String, Long> value1, Tuple2<String, Long> value2)
                                throws Exception {
                            return Tuple2.of(value1.f0, value1.f1 + value2.f1);
                        }
                    });
            userClickCount.print();
    ```
    
  - **<font style="color:red">聚合函数(AggregateFunction)</font>**

    - **ReduceFunction 可以解决大多数归约聚合的问题，但是该接口有一个限制，就是聚合状态的类型、输出结果的类型都必须和输入类型一样;**
    
    - **AggredateFunction 可以看做是 ReduceFunction 的通用版本，这里有三种类型: 输入类型(IN)、累加类型(ACC)、输出类型(OUT)**
    
    - 接口中四种方法:
    
      - **createAccumulator(): 创建一个累加器，这是为了聚合创建一个初始状态，每个聚合任务只会调用一次**
      - **add(): 将输入的元素添加到累加器中**
      - **getResult(): 从累加器中提取聚合的输出结果**
      - **merge(): 合并两个累加器，并将合并后的状态作为一个累加器返回**
    
    - 示例:
    
      ```java
      package com.luke.flink.datastreamdemo;
      import java.sql.Timestamp;
      import java.time.Duration;
      
      import org.apache.flink.api.common.eventtime.SerializableTimestampAssigner;
      import org.apache.flink.api.common.eventtime.WatermarkStrategy;
      import org.apache.flink.api.common.functions.AggregateFunction;
      import org.apache.flink.api.java.tuple.Tuple2;
      import org.apache.flink.api.java.tuple.Tuple3;
      import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
      import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
      import org.apache.flink.streaming.api.windowing.assigners.TumblingEventTimeWindows;
      import org.apache.flink.streaming.api.windowing.time.Time;
      
      public class windowAggregateTest {
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
      
              // 用 user.timestamp, 统计每个用户10秒内的其平均值
              stream.keyBy(data -> data.user)
                      .window(TumblingEventTimeWindows.of(Time.seconds(10)))
                      // Tuple3(String,Long,Integer) 第一个是用户名,第二个是 timestamp, 第三个是 count
                      .aggregate(new AggregateFunction<Event, Tuple3<String, Long, Integer>, Tuple2<String, String>>() {
                          @Override
                          public Tuple3<String, Long, Integer> createAccumulator() {
                              // 为了聚合创建一个初始状态
                              return Tuple3.of("", 0L, 0);
                          }
      
                          @Override
                          public Tuple3<String, Long, Integer> add(Event value, Tuple3<String, Long, Integer> accumulator) {
                              // 输入的元素添加到累加器中
                              return Tuple3.of(value.user, accumulator.f1 + value.timestamp, accumulator.f2 + 1);
                          }
      
                          @Override
                          public Tuple2<String, String> getResult(Tuple3<String, Long, Integer> accumulator) {
                              // 从累加器中提取聚合的输出结果
                              Timestamp timestamp = new Timestamp(accumulator.f1 / accumulator.f2);
                              return Tuple2.of(accumulator.f0, timestamp.toString());
                          }
      
                          @Override
                          public Tuple3<String, Long, Integer> merge(Tuple3<String, Long, Integer> a,
                                  Tuple3<String, Long, Integer> b) {
                              // (窗口合并时候,一般在会话窗口)合并两个累加器，并将合并后的状态作为一个累加器返回
                              return Tuple3.of(b.f0, a.f1 + b.f1, a.f2 + b.f2);
                          }
                      })
                      .print();
      
              env.execute();
          }
      }
      // 输出结果
      (Alice,2022-11-30 23:01:44.71)
      (Mary,2022-11-30 23:01:46.045)
      (Cary,2022-11-30 23:01:49.717)
      
      (Blob,2022-11-30 23:01:50.718)
      (Alice,2022-11-30 23:01:52.721)
      (Cary,2022-11-30 23:01:54.222)
      (Mary,2022-11-30 23:01:56.725)
      ```
      
      统计每一天的pv 和uv:
      
      ```java
              // 所有数据在一起统计pv 和 uv
              stream.keyBy(data -> true)
                      .window(TumblingEventTimeWindows.of(Time.minutes(24 * 60)))
                      .aggregate(new PvUv())
                      .print();
      
          // 根据日志统计pvuv, pv:页面一天的点击次数, uv: 一天内有多少用户进入
          // Tuple2<Long,HashSet<String>> 第一个保留的就是pv, 页面总点击次数, HashSet<String>保留的就是user名去重
          // Tuple2<Long,Long> 第一个保留的是pv,第二个是uv的数字值;
          public static class PvUv implements AggregateFunction<Event, Tuple2<Long, HashSet<String>>, Tuple2<Long, Long>> {
      
              @Override
              public Tuple2<Long, HashSet<String>> createAccumulator() {
                  return Tuple2.of(0L, new HashSet<String>()); // 初始化 accumulator
              }
      
              @Override
              public Tuple2<Long, HashSet<String>> add(Event value, Tuple2<Long, HashSet<String>> accumulator) {
                  // 每来一条数据,pv个数加1,将user放到 HashSet 中
                  accumulator.f1.add(value.user);
                  return Tuple2.of(accumulator.f0 + 1, accumulator.f1);
              }
      
              @Override
              public Tuple2<Long, Long> getResult(Tuple2<Long, HashSet<String>> accumulator) {
                  return Tuple2.of(accumulator.f0, Long.valueOf(accumulator.f1.size()));
              }
      
              @Override
              public Tuple2<Long, HashSet<String>> merge(Tuple2<Long, HashSet<String>> a, Tuple2<Long, HashSet<String>> b) {
                  return null;
              }
          }
      ```

- 全窗口函数(full window functions)

  - **先把窗口所有数据收集起来，等到计算的时候会遍历所有数据**。（来一个放一个，窗口临界位置才遍历且计算、输出）
  
  - ProcessWindowFunction，WindowFunction
  
  - 有些场景下，我们要做的计算必须基于全部数据才有效(比如算数据的中位数)，这时做数据聚合就没意义了。
  
  - 和增量聚合函数不同，全窗口函数需要先收集窗口中的数据，并在内部缓存起来，等到窗口要输出结果的时候再取出数据进行计算；
  
  - 全窗口函数有两种:
  
    - 窗口函数(WindowFunction): 较老版本实现，能提供的上下文信息较少;
  
    - 处理窗口函数 ProcessWindowFunction(): 推荐
  
      ```java
              // 使用 processWindowFunction计算uv
              stream.keyBy(data -> true)
                      .window(TumblingEventTimeWindows.of(Time.seconds(10)))
                      .process(new UvCountByWindow())
                      .print();
      
          // 实现自定义的 ProcessWindowFunction,输出 每个页面的uv 信息
          // <Event, String, Boolean, TimeWindow>
          // 第一个参数是 输入的value类型, 第二个参数是 输出的value类型;
          // 第三个参数是 key类型,就是keyBy中的输出类型,所以是 Boolean类型; 第四个参数是 window类型;
          public static class UvCountByWindow extends ProcessWindowFunction<Event, String, Boolean, TimeWindow> {
              @Override
              public void process(Boolean key, ProcessWindowFunction<Event, String, Boolean, TimeWindow>.Context context,
                      Iterable<Event> elements, Collector<String> out) throws Exception {
                  // context 包含了window信息等
      
                  // 用HashSet保存user
                  HashSet<String> userSet = new HashSet<>();
                  // 从elements 中遍历数据放到 userSet中
                  for (Event e : elements) {
                      userSet.add(e.user);
                  }
                  Integer uv = userSet.size();
                  // 结合窗口信息
                  Long start = context.window().getStart();
                  Long end = context.window().getEnd();
      
                  out.collect("窗口 " + new Timestamp(start) + " ~ " + new Timestamp(end) + " UV值为:" + uv);
              }
          }
      
      //结果输出:
      窗口 2022-12-04 01:14:20.0 ~ 2022-12-04 01:14:30.0 UV值为:4
      窗口 2022-12-04 01:14:30.0 ~ 2022-12-04 01:14:40.0 UV值为:3
      窗口 2022-12-04 01:14:40.0 ~ 2022-12-04 01:14:50.0 UV值为:4
      窗口 2022-12-04 01:14:50.0 ~ 2022-12-04 01:15:00.0 UV值为:4
      窗口 2022-12-04 01:15:00.0 ~ 2022-12-04 01:15:10.0 UV值为:4
      ```
      
      **<font style="color:red">全窗口函数能拿到对应的窗口信息，但是全部数据攒起来做批处理 有不是我们想要的。我们怎么在AggredateFunction 中也能拿到窗口信息呢？ </font>**
      
      方法就是将`AggredateFunction`和`ProcessWindowFunction`结合起来使用。
      有一个方法是: `aggrate(AggredateFunction<T,ACC,V>,ProcessWindowFunction<V,R,K,W>)`
      
      - `AggredateFunction`方法中 `createAccumulator()`初始化第一条数据、`add()`:每一行数据都调用 这俩方法逻辑都没变，唯一变的是`getResult()`，他将结果作为`ProcessWindowFunction()`的输入;
      
      - 示例:
      
        ```java
        package com.luke.flink.datastreamdemo;
        
        import java.sql.Timestamp;
        import java.time.Duration;
        import java.util.HashSet;
        
        import org.apache.flink.api.common.eventtime.SerializableTimestampAssigner;
        import org.apache.flink.api.common.eventtime.WatermarkStrategy;
        import org.apache.flink.api.common.functions.AggregateFunction;
        import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
        import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
        import org.apache.flink.streaming.api.functions.windowing.ProcessWindowFunction;
        import org.apache.flink.streaming.api.windowing.assigners.TumblingEventTimeWindows;
        import org.apache.flink.streaming.api.windowing.time.Time;
        import org.apache.flink.streaming.api.windowing.windows.TimeWindow;
        import org.apache.flink.util.Collector;
        
        public class UvCountExample {
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
                // 使用 AggregateFunction 和 ProcessWindowFunction 结合计算uv
                stream.keyBy(data -> true)
                        .window(TumblingEventTimeWindows.of(Time.seconds(10)))
                        .aggregate(new UvAgg(), new UvCountResult())
                        .print();
        
                env.execute();
            }
        
            // 自定义实现 AggregateFunction,增量聚合计算uv值
            public static class UvAgg implements AggregateFunction<Event, HashSet<String>, Long> {
                @Override
                public HashSet<String> createAccumulator() {
                    return new HashSet<String>();
                }
        
                @Override
                public HashSet<String> add(Event value, HashSet<String> accumulator) {
                    accumulator.add(value.user);
                    return accumulator;
                }
        
                @Override
                public Long getResult(HashSet<String> accumulator) {
                    return (long) accumulator.size();
                }
        
                @Override
                public HashSet<String> merge(HashSet<String> a, HashSet<String> b) {
                    return null;
                }
            }
        
            // 自定义实现 ProcessWindowFunction,包装窗口信息输出
            // 第一个参数输入就是 UvAgg 的 输出,所以是Long;
          	// 第二个参数是输出类型
          	// 第三个参数key类型,就是keyBy(data->true)表达式得结果,所以是 Blooean 类型
            public static class UvCountResult extends ProcessWindowFunction<Long, String, Boolean, TimeWindow> {
                @Override
                public void process(Boolean key, ProcessWindowFunction<Long, String, Boolean, TimeWindow>.Context context,
                        Iterable<Long> elements, Collector<String> out) throws Exception {
                    Long uv = elements.iterator().next();
                    // 结合窗口信息
                    Long start = context.window().getStart();
                    Long end = context.window().getEnd();
                    out.collect("窗口 " + new Timestamp(start) + " ~ " + new Timestamp(end) + " UV值为:" + uv);
                }
            }
        }
        
        //输出:
        窗口 2022-12-04 01:54:00.0 ~ 2022-12-04 01:54:10.0 UV值为:4
        窗口 2022-12-04 01:54:10.0 ~ 2022-12-04 01:54:20.0 UV值为:4
        窗口 2022-12-04 01:54:20.0 ~ 2022-12-04 01:54:30.0 UV值为:3
        窗口 2022-12-04 01:54:30.0 ~ 2022-12-04 01:54:40.0 UV值为:4
        ```
  
- <font style="color:red">统计热门商品</font>

  - 先统计每个url的访问次数
  
    ```java
    public class UrlCountViewExample {
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
    
            // 统计每个url的访问量
            stream.keyBy(data -> data.url)
                    .window(TumblingEventTimeWindows.of(Time.seconds(10)))
                    .aggregate(new UrlViewCountAgg(), new UrlViewCountResult())
                    .print();
            env.execute();
        }
    
        // 第一个是输入类型; 第二个是 中间聚合类型; 第三个是 输出类型
        public static class UrlViewCountAgg implements AggregateFunction<Event, Long, Long> {
            @Override
            public Long createAccumulator() {
                return 0L;
            }
            @Override
            public Long add(Event value, Long accumulator) {
                return accumulator + 1;
            }
            @Override
            public Long getResult(Long accumulator) {
                return accumulator;
            }
            @Override
            public Long merge(Long a, Long b) {
                return null;
            }
        }
    
        // 第一个是Long 来自于 UrlViewCountAgg的输出;
        // 第二个是输出类型 UrlViewCount
        // 第三个是 keyBy的输出类型, String;
        // 第四个是时间窗口类型;
        public static class UrlViewCountResult extends ProcessWindowFunction<Long, UrlViewCount, String, TimeWindow> {
            @Override
            public void process(String key, ProcessWindowFunction<Long, UrlViewCount, String, TimeWindow>.Context context,
                    Iterable<Long> elements, Collector<UrlViewCount> out) throws Exception {
                // 窗口信息
                Long start = context.window().getStart();
                Long end = context.window().getEnd();
                Long uv = elements.iterator().next();
                out.collect(new UrlViewCount(key, uv, start, end));
            }
        }
    }
    ```

#### 其他API

1. **触发器(Trigger)**

   **触发器是用来控制窗口什么时候触发计算，比如时间到了，个数够了**。
   所谓"触发计算"，本质就是执行窗口函数，所以可以认为是计算得到结果并输出的过程。
   基于WindowStream调用`.trigger`方法，就可以传入一个自定义的窗口触发器`Trigger`

   ```java
   stream.keyBy(...).window(...).trigger(new MyTrigger())
   ```

   Trigger是窗口算子的内部属性，每个窗口分配器(WindowAssigner) 都会对应一个默认的触发器;对于Flink内置的窗口类型，它们的触发器都已经做了实现。
   例如，所有事件时间窗口，默认的触发器都是EventTimeTrigger;类似还有ProcessingTimeTrigger和CountTrigger.
   所以一般情况下是不需要自定义触发器的。

2. **移除器(Evictor)**

   移除器主要用来定义移除某些数据的逻辑。基于WindowedStream调用.evictor()方法，就可以传入一个自定义的移除器(Evictor)。 
   Evictor是一个接口，不同的窗口类型都有各自预实现的移除器。
   ```
   stream. keyBy(...).window(...).evictor (new MyEvictor ())
   ```
   Evictor接口定义了两个方法:
   - `evictBefore()`: 定义执行窗口函数之前的移除数据操作
   - `evictAfter()`: 定义执行窗口函数之后的以处数据操作
   默认情况下,预实现的移除器都是在执行窗口函数(window fucntions)之前移除数据的。

3. **允许延迟(Allowed Lateness)**

   在事件时间语义下，窗口中可能会出现数据迟到的情况。
   为了解决迟到数据的问题，Flink提供了一个特殊的接口，可以为窗口算子设置一个“允许的最大延迟”(Allowed Lateness)。
   也就是说，我们可以设定允许延迟一段时间， 在这段时间内，窗口不会销毁，继续到来的数据依然可以进入窗口中并触发计算。
   直到水位线推进到了窗口结束时间+延迟时间，才真正将窗口的内容清空，正式关闭窗口。
   基于`WindowedStream`调用`allowedLateness()`方法，传入一个Time类型的延迟时间，就可以表示允许这段时间内的延迟数据。

   ```java
   stream.keyBy(...).window(TumblingEventTimeWindows.of(Time.hours(1))).allowedLateness(Time.minutes(1))
   ```

   定义了1小时的滚动窗口，并设置了允许一分钟的延迟数据。

4. **将迟到的数据放入侧输入流`sideOutputLateData(outputTag)`**

   ```java
   OutputTag<Event> outputTag= new OutputTag<Event>("late"){};
   stream.keyBy(...).window(TumblingEventTimeWindows.of(Time.hours(1))).sideOutputLateData(outputTag);
   ```

   将吃到数据放入侧输出流之后，还应该将它提取出来。基于窗口处理完成之后的`DataStream`，调用`.getSideOutput()`方法，传入对应的输出标签，就可以获取到吃到数据所在的流了。
   ```java
   OutputTag<Event> outputTag= new OutputTag<Event>("late"){};
   
   SingleOutputStreamOperator<AggResult> winAggStream=stream.keyBy(...).window(TumblingEventTimeWindows.of(Time.hours(1)))
   .sideOutputLateData(outputTag)
   .aggregate(new MyAggregateFunction());
   
   DataStream<Event> lateStream= winAggStream.getSideOutput(outputTag);
   ```
