### Flink运行时架构

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220813154949854.png" alt="image-20220813154949854" style="zoom:67%;" />

#### 作业管理器(JobManager)

控制一个应用程序执行的主进程，是Flink集群中任务管理和调度的核心。

- **JobMaster**
  
  - **一个Job对应一个JobMaster，可以有多个JobMaster;**
  
  - JobMaster是JobManager中最核心的组件，负责处理单独的作业(job);
  
  - **在作业提交时,JobMaster会先接收到要执行的应用。一般由客户端提交来的，包括:jar包，数据流图(dataflow graph), 和作业图(JobGraph);**
  
  - **JobMaster会把JobGraph转换成一个物理层面的数据流图，这个图被叫做"执行图"(ExecutionGraph)，它包含了所有可以并发执行的任务**
  
    JobMaster会想资源管理器(ResourceManager)发出请求，申请执行任务必要的资源。一旦它获取到了足够的资源，就会将执行图分发到真正运行它们的TaskManager上。
  
  - 在运行过程中，JobMaster会负责所有需要中央协调的操作，比如检查点(checkpoints)的协调，检查点即定期的存盘，防止故障后状态数据丢失；
  
- **资源管理器(ResourceManager)**
  - ResourceManager主要负责资源的分配和管理，在Flink集群中只有一个。**所谓"资源"，主要只TaskManager的任务槽(task slots)。任务槽就是Flink集群中的资源调度单元，包含了机器用来执行计算的一组CPU和内存资源。每一个任务Task都需要分配到一个slot上执行;**
  - 在实际执行中，**任务槽(task slots)实际上是做了一些资源隔离的，主要做的是内存隔离**;
  
- **分发器(Dispatcher)**
  - **Dispatcher 主要负责提供一个REST接口，用来提交应用，并负责为每个新提交的作业启动一个新的JobMaster组件**;
  - Dispatcher 会启动一个Web UI，用来方便地展示和监控作业执行的信息。Dispatcher 在架构中并不是必需的，在不同的部署模式中可能会被忽略掉。

#### 任务管理器(TaskManager)

- Flink的工作进程。**通常在Flink中会有多个TaskManager运行**，每个TaskManager都包含了一定数量的插槽(slots)。**插槽的数量限制了TaskManager能够并行处理的任务数量**;
- **启动之后，TaskManager会向资源管理器注册它的插槽；收到资源管理器的指令后，TaskManager就会将一个或者多个插槽提供给JobMaster调用**。JobMaster就可以向插槽分配任务(tasks)来执行了；
- 在执行过程中，一个TaskManager可以跟其它运行同一个应用程序的TaskManager交换数据；

#### 作业提交大致流程

![image-20220813162037563](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220813162037563.png)

**Standalone(的会话session)模式作业提交流程**

JobManager是提前启动的;

![image-20220813162502166](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220813162502166.png)

**yarn(的会话session)模式作业提交流程**

- JobManager会提前存在;
- TaskManager是在有需求时才启动;
- Flink的资源管理器 向 YARN的资源管理器请求资源;

![image-20220813163159092](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220813163159092.png)

**yarn(的单作业)模式作业提交流程**

- JobManager 和TaskManager 提前都没有，只有任务提交时才会创建;
- 任务提交时 先提交给 YARN 的资源管理器;
- YARN的资源管理器 启动一个带有Flink JobManager的YARN Application Master;
- Flink的资源管理器 还是需要向 YARN的资源管理器 申请 TaskManager的资源;

![image-20220813163306642](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220813163306642.png)

**问题:**

- 怎样从Flink程序得到任务(JobManager怎么把flink程序变成一个个小的task?
- 一个流处理程序，到底包含多少个任务? JobMaster的核心功能;
- 最终执行任务，需要占用多少slots?

#### 程序与数据流(DataFlow)

- <font style="color:red">**所有的Flink程序都是由三部分组成的: `Source`、`Transformation`和`Sink`**</font>

  - **Source: 负责读取数据源;**
  - **Transformation利用各种算子进行处理和加工(如map、flat_map);**
  - **Sink负责输出;**

- 运行时，Flink上运行的程序会被映射成"逻辑数据流(dataflows)",它包含了以上三部分;

- 每一个dataflow以一个或多个sources开始以一个或多个sinks结束。dataflow类似于任意的有向无环图(DAG);

- 在大部分情况下，**程序中的转换运算(transformations)跟dataflow中算子(operator)是一一对应的关系**

  比如`flat_map`对应一个算子, map对应一个算子, key_by+ sum对应一个算子;

##### 并行度(Parallelism)

- <font style="color:red">**每一个算子(operator)可以包含一个或多个子任务(operator subtask)，比如下面的map()[1]、map()[2]、Source[1]、Source[2]，这些子任务在不同的线程、不同的物理机或不同的容器中完全独立地执行**</font>

- **一个特定算子的子任务(subtask)的个数被称之为其并行度(parallelism)。所以不同算子可以对应不同的不行度;**

- 我们可以给每个算子设置其并行度,通过`set_parallelism()`来完成,

  比如`ds.flat_map(split).set_parallelism(1)`;

  比如`ds.print().set_parallelism(1) `全局统一输出;

  `env.set_parallelism(1)`这段代码会将全局的每个算子的并行度设置为1;

  还有`flnk_conf.yaml`文件中的`parallelism.default:1` 是全集群设置;

- 两种并行:

  - 任务并行: 如前后执行的任务 可同时执行;
  - 数据并行: 同一个算子，可拆分成多份，同时处理多个数据;


<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220813170253311.png" alt="image-20220813170253311" style="zoom:67%;" />

#### 数据传输形式

- 一个程序中，不同的算子可能具有不同的并行度;

- 算子之间传输数据的形式可以是`one-to-one(forwarding)`模式也可以是`redistributing`模式，具体哪一种模式，取决于算子的种类;

- **`one-to-one`: stream维护者分区以及元素的顺序(比如source和map之间,source的数据直接丢给map用,不需要跨机器?)**
  **这意味着map算子的子任务看到的元素个数以及顺序跟source算子的子任务生产的元素个数、顺序相同。map、filter、flatmap等算子都是`one-to-one`的对应关系**;

- **`redistributing`:stream的分区会发生改变。每一个算子的子任务依据所选择的transformation发送数据到不同的目标任务。比如source 朝两个 map 传输数据，则平分数据;**

  **如, keyBy基于hashCode重分区、而broadcast 和 rebalance会随机重分区，这些算子都会引起redistribute过程，而redistribute过程类似于Spark的shuffle过程**

- **算子链(Operator Chains)**

  - Flink采用了一种称为任务链的优化技术，可在特定条件下减少本地通信的开销。为了满足任务链的要求，必须将两个或更多算子设为相同的并行度，并通过本地转发(local forward)方式链接；

  - **相同并行度的`one-to-one`操作**，Flink这样相连的算子链接在一起形成一个task，原本算子称为里面的subtask;

    ![image-20220813174559959](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220813174559959.png)

  - **并行度相同 并且是one-to-one条件，缺一不可**;

  <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221028010159311.png" alt="image-20221028010159311" style="zoom:50%;" />

#### 执行图(ExecutionGraph)

- Flink中执行图分为四层: StreamGraph -> JobGraph -> ExecutionGraph->物理执行图
- **StreamGraph(也就是dataflow):** 根据用户通过StreamAPI 编写的代码生成的最初的图。用于表示程序的拓扑结构;
- **JobGraph**: StreamGraph经过优化后生成了JobGraph, 提交给JobManager的数据结构。**主要优化为: 将多个符合条件的节点chain在一起作为一个节点。在session、pre-mode模式中，这些都是在flink client侧完成的；application mode不同;**
- **ExecutionGraph**:JobManager根据JobGraph生成ExecutionGraph。ExecutionGraph是JobGraph的并行化版本，是调度层最核心的数据结构;
- **物理执行图**: JobManager根据ExecutionGraph对Job进行调度后，在各个TaskManager上部署Task后形成的"图"，并不是一个具体的数据结构。

![image-20220814081650524](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220814081650524.png)

![image-20220814081918952](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220814081918952.png)

![image-20220814082027104](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220814082027104.png)

#### 任务(task)和任务槽(task slot)

- Flink中每一个TaskManager都是一个JVM进程，它可能会在独立的线程上执行一个或多个任务;
- **为了控制一个TaskManager能接受多少个task，TaskManager通过task slot来进行控制(一个TaskManager至少一个slot)**
- 默认情况下，**Flink允许子任务共享slot。这样的结果是，一个slot可以保存作业的整个管道;**
- **当我们将资源密集型和非资源密集型的任务同时放到一个slot中，它们就可以自行分配对资源占用的比例，从而保证最重的活平均分配给所有的TaskManager**
- **同一个算子(task)的并行子任务(subtasks) 必须同时执行，所以最大的算子(task)并行度就是我们我们需要占据的slots数**;

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220814085038949.png" alt="image-20220814085038949" style="zoom: 50%;" />



**2个TaskManager=>6个slot=>跑13个任务**

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220814085512638.png" alt="image-20220814085512638" style="zoom:50%;" />

- Task Slot
  - 静态概念，是指TaskManager具有的并发执行能力;
  - **通过参数`taskmanager.numberOfTaskSlots`进行配置，一般是CPU核数**
- 并行度(parallelism)
  - **动态概念，也就是TaskManager运行程序时实际使用的并发能力。比如一个TaskManager上有5个task slot，但是只用了一个，此时parallelism就是1**
  - 通过参数`paralleism.default`进行配置;

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220814090717177.png" alt="image-20220814090717177" style="zoom:50%;" />

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221029094204559.png" alt="image-20221029094204559" style="zoom:50%;" />

**全局并行度为9，并单独设置Sink并行度为1**

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220814091005476.png" alt="image-20220814091005476" style="zoom:50%;" />



<font style="color:red">**一些特别的方法调用**</font>

- **disableChaining():关闭算子链，比如禁止flatmap和前一个任务(source)、后一个任务(reduce)一个算子链**
- **startNewChain():开启新的算子链,**
- **slotSharingGroup():设置算子的slot共享组，在一个共享组中的算子才能设置共享slot。这样能人为减少slot共享。(需要重新查资料)**
