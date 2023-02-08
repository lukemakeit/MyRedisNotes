## 容错机制

### 检查点

##### 1. 周期性保存

##### 2. 保存的时间点

当检查点的保存被触发时，任务可能正在处理某个数据，这是应该怎么办？

最简单的想法是，某个时刻全部任务按下 "暂停键"，这样状态不再更改，大家可以复制保存；保存完毕后，再同时恢复数据处理。

然而这个简单粗暴的行为，非常难搞定，执行到代码的哪一步了，很难确定。

但这样依然会有问题:分布式系统的节点之间需要通过网络通信来传递数据，如果我们保存检查点的时候刚好有数据在网络传输的路上，那么下游任务是没法将数据保存起来的;故障重启之后，我们只能期待上游任务重新发送这个数据。然而上游任务是无法知道下游任务是否收到数据的，只能盲目地重发，这可能导致下游将数据处理两次，结果就会出现错误。

**所以我们最终的选择是:<font style="color:red">当所有任务都恰好处理完一个相同的输入数据的时候，将它们的状态保存下来</font>**首先，这样避免了除状态之外其他额外信息的存储，提高了检查点保存的效率。
其次，一个数据要么就是被所有任务完整地处理完，状态得到了保存;要么就是没处理完，状态全部没保存:这就相当于构建了一个“事务”(transaction)。 如果出现故障，我们恢复到之前保存的状态，故障时正在处理的所有数据都需要重新处理;**所以我们只需要让源(source)任务向数据源重新提交偏移量、请求重放数据就可以了**。这需要源任务可以把偏移量作为算子状态保存下来，而且外部数据源能够重置偏移量;Kafka就是满足这些要求的一个最好的例子。

##### 3. 检查点保存流程

检查点的保存，最关键的就是等所有任务将"同一个数据"处理完毕。

例如在前面示例的 WordCount 中，输入:

`hello world hello flink hello world helo flink`

处理代码简化:

```java
env.addSource(...)
	.map(word -> Tuple2.of(word,1L)). returns(Types.TUPLE(Types.STRING,Types.lONG))
	.keyBy(t->t.f0)
	.sum(1);
```

源(Source)任务从外部数据源读取数据，并记录当前的偏移量，作为算子状态(OperatorState)保存下来。然后将数据发给下游的Map任务，它会将一个单词转换成(word, count)二元组，初始count都是1，也就是(“hello", 1)、(“world", 1)这样的形式;这是一个无状态的算子任务。
进而以word作为键(key) 进行分区，调用sum()方法就可以对count值进行求和统计了;Sum算子会把当前求和的结果作为按键分区状态(Keyed State)保存下来。最后得到的就是当前单词的频次统计(word, count)，如下图所示。

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221214233007421.png" alt="image-20221214233007421" style="zoom:50%;" />

##### 4. 从检查点恢复状态

在运行流处理程序时，Flink会周期性地保存检查点。当发生故障时，就需要找到最近一次成功保存的检查点来恢复状态。
例如在上节的word count 示例中，我们处理完三个数据后保存了一个检查点。之后继续运行，又正常处理了一个数据“flink",在处理第五个数据“hello" 时发生了故障。

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221214233557433.png" alt="image-20221214233557433" style="zoom:50%;" />

这里Source任务已经处理完毕，所以偏移量为5；Map任务也就处理完成了。而Sum任务在处理中发生了故障，此时状态并未保存。

恢复状态步骤:

- 重启应用：所有任务状态会清空;

- 读取检查点，重置状态

  找到最近一次保存的检查点，从中读出算子任务的快照，分别填充到对应的状态中。这样，Flink内部所有任务的状态，就恢复到了保存检查点的那一刻，也就是刚好处理了第三个数据的时刻。此时key为 flink  的数据并未到来，所以初始化为0

  <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221214234005896.png" alt="image-20221214234005896" style="zoom:50%;" />

- 重放数据

  为了不丢失数据，我们应该从保存检查点后开始重新读取数据，这可以通过Source任务向外部数据源重新提交偏移量(offset)实现。

  <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221214234220864.png" alt="image-20221214234220864" style="zoom:50%;" />

  这样，整个系统的状态已经完全回退到了检查点保存完成的那一刻。

- 继续处理数据；

##### 5. 检查点算法

我们已经知道，Flink 保存检查点的时间点，是所有任务都处理完同一个输入数据的时候。但是不同的任务处理数据的速度不同，当第一个Source任务处理到某个数据时，后面的Sum任务可能还在处理之前的数据;而且数据经过任务处理之后类型和值都会发生变化，面对着“面
目全非”的数据，不同的任务怎么知道处理的是“同一个”呢?

一个简单的想法是，当接到JobManager发出的保存检查点的指令后，Source 算子任务处理完当前数据就暂停等待，不再读取新的数据了。这样我们就可以保证在流中只有需要保存到检查点的数据，只要把它们全部处理完，就可以保证所有任务刚好处理完最后一个数据;这时把所有状态保存起来，合并之后就是一个检查点了。**这样做最大的问题，就是每个任务的进度可能不同;先执行完的任务为了保证状态一致不能进行其他工作，只能等待。当先保存完状态的任务需要等待其他任务时，就导致了资源的闲置和性能的降低**

**所以更好的做法是，在不暂停整体流处理的前提下，将状态备份保存到检查点。在Flink中，采用了基于Chandy-Lamport算法的分布式快照**

- **检查点分界线**

  现在的目标是，在不暂停流处理的前提下，让每个任务"认出"触发检查点保存的那个数据。

  所以可以借鉴水位线(watermark)的设计，在数据流中插入一个特殊的数据结构，专门用来表示触发检查点保存的时间点。**<font style="color:red">收到保存检查点的指令后，Source任务可以在当前数据流中插入这个结构;之后的所有任务只要遇到它就开始对状态做持久化快照保存。由于数据流是保持顺序依次处理的，因此遇到这个标识就代表之前的数据都处理完了，可以保存一个检查点;而在它之后的数据，引起的状态改变就不会体现在这个检查点中，而需要保存到下一个检查点</font>**

  这种特殊的数据形式，把一条流上的数据按照不同的检查点分隔开,所以就叫作检查点的“分界线”(Checkpoint Barrier)。

  与水位线很类似，检查点分界线也是一条特殊的数据，**由Source算子注入到常规的数据流中**，它的位置是限定好的，不能超过其他数据，也不能被后面的数据超过。**<font style="color:red">检查点分界线中带有一个检查点ID,这是当前要保存的检查点的唯一标识</font>**

  <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221215063145041.png" alt="image-20221215063145041" style="zoom:50%;" />

- **分布式快照算法**

  我们先回忆一下水位线( watermark)的处理: **上游任务向多个并行 下游任务传递时，需要广播出去;而多个上游任务向同一个下游任务传递时，则需要下游任务为每个上游并行任务维护一个“分区水位线”，取其中最小的那个作为当前任务的事件时钟**

  那barier在并行数据流中的传递，是不是也有类似的规则呢?
  watermark指示的是“之前的数据全部到齐了”，而barrier 指示的是“之前所有数据的状态更改保存入当前检查点”:它们都是一个“截止时间”的标志。所以在处理多个分区的传递时，也要以是否还会有数据到来作为一个判断标准。

  具体实现上，** Flink 使用了Chandy-Lamport 算法的一种变体， 被称为“异步分界线快照”(asynchronous barrier snapshotting)算法**
  算法的核心就是两个原则:

  - **当上游任务向多个并行 下游任务发送barrier时，需要广播出去**
  - **而当多个，上游任务向同一个下游任务传递barrier时，需要在下游任务执行“分界线对齐”(barrieralignment)操作，也就是需要<font style="color:red">等到所有并行分区的barrier都到齐，才可以开始状态的保存</font>**

  具体流程:

  - JobManager发送指令，触发检查点保存；Source任务保存组昂天，插入分界线

    **JobManager会周期性地向每个TaskManager 发送一条带有新检查点ID的消息，通过这种方式来启动检查点**。收到指令后，TaskManger 会在所有Source 任务中插入一个分界线(barrier)，并将偏移量保存到远程的持久化存储中。

    <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221215064228660.png" alt="image-20221215064228660" style="zoom:50%;" />

  - 状态快照保存完成，分界线向下游传递

    Source和Map之间是一对一(forward)的传输关系，所以barrier可以直接传递给对应的Map任务。

    <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221215064646884.png" alt="image-20221215064646884" style="zoom:50%;" />

  - 向下游多个并行任务广播分界线，执行分界线对齐

    <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221215064921513.png" alt="image-20221215064921513" style="zoom:50%;" />

    此时的Sum 2收到了来自上游两个Map任务的barrier， 说明第一条流第三个数据、 第二条流第一个数据都已经处理完，可以进行状态的保存了;而Sum 1只收到了来自Map 2的barrier，所以这时需要等待分界线对齐。

    **在等待的过程中，如果分界线尚未到达的分区任务Map1又传来了数据(hello,1)，说明这是需要保存到检查点的，Sum 1任务应该正常继续处理数据，状态更新为3;而如果分界线已经到达的分区任务Map2又传来数据，这已经是下一个检查点要保存的内容了，就不应立即处理，而是要缓存起来、等到状态保存之后再做处理。**

  - 分界线对齐后，保存状态到持久化存储

    各个分区的分界线都对齐后，就可以对当前状态做快照，保存到持久化存储了。存储完成之后，同样将barrier向下游继续传递，并通知JobManager保存完毕。

    <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221215070103005.png" alt="image-20221215070103005" style="zoom:50%;" />

  - 先处理缓存数据，然后继续正常处理

    完成检查点保存之后,任务就可以继续正常处理数据了。这时如果有等待分界线对齐时缓存的数据，需要先做处理;然后再按照顺序依次处理新到的数据。

    **当JobManager收到所有任务成功保存状态的信息，就可以确认当前检查点成功保存。之后遇到故障就可以从这里恢复了**

    **由于分界线对齐要求先到达的分区做缓存等待，工定程度上会影响处理的速度;当出现背压(backpressure) 时，下游任务会堆积大量的缓冲数据，检查点可能需要很久才可以保存完毕。为了应对这种场景，Flink 1.11 之后提供了不对齐的检查点保存方式，可以将未处理的缓冲数据( in-flight data)也保存进检查点。这样，当我们遇到一个分区barrier时就不需等待对齐，而是可以直接启动状态的保存了**

  ##### 6. 检查点配置

  - 启用检查点
  
    ```java
    env.enableCheckpointing(1000L); //参数是毫秒,现在每隔1s,source产生一个 Barrier
    ```
  
  - **检查点存储(Checkpoint Storage)**
  
    检查点具体的持久化存储位置，取决于“检查点存储”(CheckpointStorage) 的设置。
  
    默认情况下，检查点存储在JdbManager的堆(heap)内存中。而对于大状态的持久化保存，Flink也提供了在其他存储位置进行保存的接口，这就是CheckpointStorage。
  
    具体可以通过调用检查点配置.setCheckpointStorage()`来配置，需要传入一个CheckpointStorage的实现类。
    Flink 主要提供了两种CheckpointStorage:作业管理器的堆内存(`JobManagerCheckpointStorage`)和文件系统(`FileSystemCheckpointStorage`)。
  
    ```java
    //配置存储检查点到JobManager堆内存
    env.getCheckpointConfig().setCheckpointstorage(new JobManage rCheckpointStorage()) ;
    //配置存储检查点到文件系统
    env.getCheckpointConfig().setCheckpointstorage(new FileSys temCheckpointStorage ("hdfs://namenode:40010/flink/checkpoints")) ;
    ```
  
    对于实际生成环境，我们一般将`CheckpointStorage`配置为高可用的分布式文件系统(HDFS, S3等);
    
  - **其他配置项**
  
    ```java
    env.getCheckpointConfig().setCheckpointTimeout(60000L); //设置超时时间，超过这个时间还没保存结束，则丢弃
    env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.AT_LEAST_ONCE); //至少一次
    env.getCheckpointConfig().setMinPauseBetweenCheckpoints(500L); // 如检查点本身耗费了 500ms, 下一个检查点和上一个检查点至少间隔 500ms
    env.getCheckpointConfig().setMaxConcurrentCheckpoints(2); // 同时保存检查点 的并发度, 如A耗时很长,B到来前A还没结束
    env.getCheckpointConfig().enableUnalignedCheckpoints(); //  非对齐的 barrier 操作
    env.getCheckpointConfig().enableExternalizedCheckpoints(RETAIN_ON_CANCEL); // 开启外部检查点? 如job被cancel，检查点是否删除?
    env.getCheckpointConfig().setTolerableCheckpointFailureNumber(0); //是否允许检查点保存失败,默认不允许失败
    ```
    
  
  ##### 7. 保存点(Savepoint)
  
  - 保存点原理 和 检查点一致，不过检查点是Flink自动的行为，保存点是用户手动行为;
  
  - 保存点中的状态快照，是以算子ID和状态名称组织起来的，相当于一个键值对。从保存点启动应用程序时，Flink 会将保存点的状态数据重新分配给相应的算子任务;
  
  - **保存点的用途**
  
    - 保存点与检查点最大的区别，就是触发的时机。检查点是由Flink自动管理的，定期创建，发生故障之后自动读取进行恢复，这是一个“自动存盘”的功能;而保存点不会自动创建，必须由用户明确地手动触发保存操作，所以就是“手动存盘”。因此两者尽管原理一-致，但用途就有所差别了:检查点主要用来做故障恢复，是容错机制的核心;保存点则更加灵活，可以用来做有计划的手动备份和恢复;
  
    - 具体应用场景如:
  
      - **版本管理和归档管理: 对重要节点手动备份，设置为某一版本，归档(archive)存储应用程序的状态**
      - **更新Flink版本: 升级版本只需创建一个保存点，停掉应用、升级Flink后，从保存点重启后就可以继续处理了**
      - **更新应用程序: 需要升级前后程序是兼容的;**
      - **调整并行度: 如果运行的程序发现需要资源不足 or 已经有大量剩余，则可通过从保存点重启的方式，将应用程序的并行度增大 or 减少**
      - **暂停应用程序: 暂停程序 让其他更重要的程序先执行。也可以通过 暂停和重启，对优先的集群资源合理调配;**
  
    - 需要注意的是，保存点能够在程序更改时候依然兼容，前提是状态的拓扑结果和数据类型不变。我们知道保存点中状态都是以 算子ID—状态名称这样的 key-value 组织起来的，<font style="color:red">算子ID可以再代码中直接调用`SingleOutputStreamOperator`的`.uid()`方法来进行指定</font>
  
      ```java
      env.addSource(new StatefulSource()).uid("source-id").map(new StatefulMapper()).uid("mapper-id").print()
      ```
  
      **对于没有设置ID的算子，Flink默认会自动进行设置，所以在重新启动应用后可能会导致ID不同而无法兼容以前的状态。所以为了方便后续的维护，强烈建议在程序中为每一个算子手动指定ID**
  
  - **使用保存点**
  
    - 创建保存点，命令行:
  
      ```
      bin/flink savepoint :jobId [:targetDirectory]
      ```
  
      jobId就是作业ID，目标路径 targetDirectory 是可选的保存路径;
  
      默认保存路径，可通过配置`flink-conf.yaml`中的`state.savepoints.dir`来设定
      ```java
      state.savepoints.dir: hdfs://flink/savepoints
      ```
  
      对于单独的作业，我们也可以再程序代码中通过执行环境来设置:
  
      ```java
      env.setDefaultSavepointDir("hdfs://flink/savepoints");
      ```
  
      犹豫创建保存点一般都是希望更改环境之后重启，所以创建之后往往紧接着就是停掉作业的操作。所以我们可以将 stop 和创建保存点用一个命令完成:
  
      ```java
      bin/flink stop --savepointPath [:targetDirectory]
      ```
  
    - **从保存点重启应用**
  
      ```java
      bin/flin run -s :savepointPath [:runArgs]
      ```
  

#### 状态一致性

##### 什么是状态一致性

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221216222732319.png" alt="image-20221216222732319" style="zoom:50%;" />

- 对于流处理器内部来说,所谓的状态一致性， 其实就是我们所说的计算结果要保证准确;
- 一条数据不应该丢失，也不应该重复计算;
- 在遇到故障时可以恢复状态，恢复以后的重新计算，结果应该也是完全正确的；

##### 状态一致性分类

- `AT_MOST_ONCE` 最多一次

  当任务故障时，最简单的做法是什么都不干，既不恢复丢失的状态，也不重播丢失的数据。`at-most-once`语义的含义是最多处理一次事件;

- `AT_LEAST_ONCE` 至少一次

  在大多数的真实场景应用中，我们不希望丢失事件。这种类型的保障称为`at_least_once`，意思是所有的事件都得到处理，而有一些事件还可能被多次处理。

- `EXACTLY_ONCE` 精确一次

  恰好处理一次是最严格的保证，也是最难实现的。恰好处理一次语义不仅仅 意味着没有事件丢失，还意味着针对每一个数据，内部状态仅仅更新一次。

**一致性检查点 Checkpoints**

- Flink使用了一种轻量级快照机制一检查点(checkpoint) 来保证exactly-once语义;
- **有状态流应用的一致检查点，其实就是:所有任务的状态,在某个时间点的一份拷贝(一份快照)。而这个时间点，应该是所有任务都恰
好处理完一个相同的输入数据的时候**
- **应用状态的一致检查点，是Flink故障恢复机制的核心**

**端到端(end-to-end)状态一致性**

- 目前我们看到的一致性保证都是由流处理器实现的，也就是说都是在Flink流处理器内部保证的;**而在真实应用中，流处理应用除了流处理器以外还包含了数据源(例如Kafka)和输出到持久化系统**
- **端到端的一致性保证，意味着结果的正确性贯穿了整个流处理应用的始终;每一个组件都保证了它自己的一致性;**
- **整个端到端的一致性级别取决于所有组件中一致性最弱的组件;**

**端到端 exactly-once**

- 内部保证——checkpoint
- **source端——可重设数据的读取开始位置**
- **sink端——从故障恢复时，数据不会写入外部系统**
  - 幂等写入
  - 事务写入

**幂等写入 idempotent writes**

所谓幂等操作，是说一个操作，可以重复执行很多次，最后的结果都是一样的。这对外部系统要求较高。

**事务写入 Transactional writes**

- 事务 Transaction
  - 应用程序中一系列严密的操作，所有操作必须成功完成，否则在每个操作中所做的所有更改都会被撤销
  - 具有原子操作：一个事务中的一系列操作要么全部成功，要门一个不做
- 实现思想: **构建的事务对应着checkpoint,等到 checkpoint 真正完成的时候，才把所有对应的结果写入sink系统中（意思是Flink先缓存，或者说 prepare 等 commit?）**
- 实现方式
  - 预写日志 WAL，Write-Ahead-log
    - **先把结果数据先当成状态保存，然后收到checkpoint完成的通知时，一次性写入sink系统**
    - 简单易于实现，由于数据提前在状态后端中做了缓存，所以无论什么sink系统，都能用这种方式一批搞定
    - DataStream API 提供了一个模版类: GenericWriteAheadSink，来实现这种事务性Sink
    - 如果只是缓存，没有写日志，不算 WAL吧?否则 写一半失败后，怎么从日志中恢复呢？
  - 两阶段提交 
    - 对于每个checkpoint，sink任务会启动要给事务，并将接下来所有接收的数据添加到事务中
    - 然后将这些数据写入外部sink系统，但不提交他们——这时只是预提交
    - 当它收到checkpoint完成的通知时，它才正式提交事务，实现结果的真正写入
    - 这种真正实现了 exactly-once，它需要一个提交事务支持的外部sink系统。Flink提供了 TwoPhaseCommitSinkFunction接口
    - **两阶段提交对外部系统的要求:**
      - 外部sink系统必须提供事务支持， 或者sink任务必须能够模拟外部系统上的事务
      - 在checkpoint的间隔期间里，必须能够开启一个事务并接受数据写入
      - 在收到checkpoint完成的通知之前，事务必须是"等待提交的状态。在故障恢复的情况下，这可能需要-些时间。如果这个时候sink系统关闭
      事务(例如超时了)，那么未提交的数据就会丢失
      -  sink任务必须能够在进程失败后恢复事务
      - 提交事务必须是幂等操作

**不同Source和Sink的一致性保证**

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221216230510928.png" alt="image-20221216230510928" style="zoom:50%;" />

#### Flink 和 kafka连接时的精确一次保证

- 输入端: 输入数据源端的Kafka可以对数据进行持久化保存，并可以重置偏移量(offset)。**所以我们可以在Source任务(`FlinkKafkaConsumer`)中将当前读取的偏移量保存为算子状态，写入到检查点中;当发生故障时,从检查点中读取恢复状态,并由连接器`FlinkKafkaConsumer`向Kafka重新提交偏移量，就可以重新消费数据、保证结果的一致性了**

- 输出端: **还是用Kafka，支持两阶段提交**

- 具体步骤:

  先预提交 最后确认全部检查点ok后，在做提交

  <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221216231730383.png" alt="image-20221216231730383" style="zoom:50%;" />

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221216231823556.png" alt="image-20221216231823556" style="zoom:50%;" />

- **需要的配置**

  在具体应用中，实现真正的端到端`exactly-once`，还需要有一些额外的配置:

  - 必须启用检查点;

  - 在`FlinkKafkaProducer`的构造函数中传入参数`Semantic.EXACTLY_ ONCE`
  - 配置Kafka读取数据的消费者的隔离级别
    这里所说的Kafka，是写入的外部系统。预提交阶段数据已经写入，只是被标记为“未提交”(uncommitted)，而Kafka中默认的隔离级别isolation.evel 是read_ uncommitted, 也就是可以读取未提交的数据。这样一来， 外部应用就可以直接消费未提交的数据，对于事务性的保证就失效了。**所以应该将隔离级别配置为read_ committed**，表示消费者遇到未提交的消息时，会停止从分区中消费数据，直到消息被标记为已提交才会再次恢复消费。当然，这样做的话，外部应用消费数据就会有显著的延迟。
  - 事务超时配置
    **Flink的Kafka连接器中配置的事务超时时间transaction.timeout.ms默认是1小时,而Kafka集群配置的事务最大超时时间transaction.max.timeout.ms 默认是15 分钟。所以在检查点保存时间很长时，有可能出现Kafka已经认为事务超时了，丢弃了预提交的数据;而Sink任务认为还可以继续等待。如果接下来检查点保存成功，发生故障后回滚到这个检查点的状态，这部分数据就被真正丢掉了。所以这两个超时时间，前者应该小于等于后者**

  