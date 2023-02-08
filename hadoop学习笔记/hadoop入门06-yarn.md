## Yarn

Yarn是一个资源调度平台，负责为运算程序共服务器运算资源，相当于一个**分布式操作系统平台**，而MapReduce等运算程序则相当于运行于**操作系统之上的应用程序**。

#### Yarn 基础架构

Yarn主要由`ResourceManager`、`NodeManager`、`ApplicationMaster`(简写为`App Mstr`)和`Container`等组件构成。

`ResourceManager`：是整个集群的老大；
`NodeManager`：每个Node节点的老大;
`ApplicationMaster`:每个任务的老大;
`Container`:

**ResourceManager(RM) 主要作用**

1. 处理客户端请求
2. 监控NodeManager
3. 启动或监控 ApplicationMaster
4. 资源的分配和调度

**NodeManager(NM)主要作用如下:**

1. 管理单个节点上的资源
2. 处理来自`ResourceManager`的命令
3. 处理来自`Application Master`(简写为`App Mstr`)的命令

**ApplicationMaster(AM,简写为`App Mstr`) 作用如下:**

1. 为应用程序申请资源并分配给内部的任务。如maptask 和 reducetask，换一个Node执行;
2. 任务的监控和容错

**Container作用如下:**

Container是YARN中的资源抽象，它封装了某个节点上的多个资源维度。如**内存、CPU、磁盘、网路等**

![image-20221005150521142](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221005150521142.png)

#### Yarn的工作机制

![image-20221005151127493](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221005151127493.png)

这张图中，MapTask是先申请资源运行的，ReduceTask是后申请资源运行的。
ReduceTask运行完成后，MapTask/ReduceTask的资源才会释放。

#### Mapreduce/HDFS/Yarn 关系

#### 调度器和调度算法

作业调度器主要由三种: FIFO、容量(Capacity Scheduler)和公平(Fair Scheduler)。`Apache Hadoop 3.1.3`默认调度器是`Capacity Scheduler`
CDH框架默认调度器是`Fair Scheduler`
具体设置详见: `yarn-default.xml`文件

```xml
<property>
	<name>yarn.resourcemanager.scheduler.class</name>
	<value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduler</value>
</property>
```

##### FIFO(First In First Out)

单队列, 根据提交作业的先后顺序，先来先服务。第一个job执行后 才执行第二个job、第三个job...

##### 容量调度器(Hadoop 3.x默认调度器)

Capacity Scheduler是Yahoo开发的多用户调度器。

![image-20221005154620450](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221005154620450.png)

1. **多队列: 每个队列可配置一定的资源量，每个队列采用FIFO调度策略**;
2. **容量保证: 管理员可为每个队列设置资源最低保证 和 资源使用上限**;
3. **灵活性: 如果一个队列中的资源有剩余，可以暂时共享给那些需要资源的队列，而一旦队列有新的应用程序提交，则其他队列调度的资源会归还给该队列;**
4. **多租户**
   支持多用户共享集群和多应用程序同时运行;
   为了防止同一个用户的作业独占队列中的资源，该调度器会对 **同一用户提交的作业所占资源进行限定。**

**容量调度器资源分配算法**

```
root
|---queueA 20%
|---queueB 50%
|---queueC 30%
    |-- ss 50%
    |-- cls 50%
```

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221005155732387.png" alt="image-20221005155732387" style="zoom:50%;" />

按照图中逐层的分配方法，依次是:`queue`、`作业`、`container`

1. **队列资源分配: 从root开始，使用深度优先算法，优先选择资源占用率最低的队列分配资源。比如队列`queueA 20%`。目标是先用小任务先执行完，重点留大资源给大任务；**
2. **作业资源分配:默认按照提交作业的优先级和提交时间顺序分配资源;**
3. **容器资源分配:**
   - 按照容器的**优先级**分配资源
   - 如果优先级相同，按照**数据本地性原则**
     - 任务和数据在同一节点
     - 任务和数据在同一机架
     - 任务和数据不在同一个节点也不在同一机架

##### 公平调度器

Fair Scheduler 是Facebook开发的多用户调度器。
![image-20221005160522711](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221005160522711.png)

1. 与容量调度器相同的点:

   - 多队列: 支持多队列作业
   - 容量保证: 管理员可为每个队列设置资源最低保证和资源使用上限
   - 灵活性: 如果一个队列中的资源有剩余，可以暂时共享给那些需要资源的队列，而一旦该队里有新的应用程序提交，则其他队列借调的资源会归还给该队列;
   - 多租户: 支持多个用户共享集群和多应用程序同时运行；为了防止同一个用户的作业独占队列资源，该调度器会对同一用户提交的作业所占资源总量做限制。

2. 与容量调度器不同的点

   - 核心调度策略不同
     - 容量调度器: 优先选择**资源利用率低**的队列
     - 公平调度器: 优先选择**<mark style="color:red">对资源的缺额</mark>**比例大的
   - 每个队列可以单独设置资源分配方式
     - 容量调度器: **FIFO、<mark style="color:red">DRF</mark>**
     - 公平调度器: **FIFO、<mark style="color:red">FAIR、DRF</mark>**

3. **公平调度——缺额**

   - **公平调度器设计的目标是: 在时间尺度上，所有作业获得公平的资源。某一时刻一个作业应获资源和实际获取资源的差距叫 "缺额"**
   - **调度器会优先为<mark style="color:red">缺额大</mark>的作业分配资源**

4. **公平调度器队列资源分配方式: FIFO、FAIR、DRF**

   - **FIFO策略**

     公平调度器每个队列资源分配策略如果选择FIFO的话，此时公平调度器相当于上面讲的容量调度器;

   - **FAIR策略**

     Fair策略(默认)。比如一个队列中有两个应用程序同时运行，则每个应用程序可得到`1/2`的资源；如果三个应用程序同时运行，则每个应用程序可获得`1/3`的资源。

     具体资源分配流程和容量调度器一致:

     - 选择队列
     - 选择作业
     - 选择容器

     以上三步，每一步都按照公平策略分配资源。

     **实际最小资源份额: mindshare=Min(资源需求量，配置的最小资源)**

     **是否饥饿: isNeedy= job当前资源使用量 < mindshare(实际最小资源份额)**

     **资源分配比(比例越低，缺得越多): minShareRatio= job当前资源使用量/Max(mindshare,1)**

     **资源使用权重比: useToWeightRatio=  job当前资源使用量/权重**

     <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221005162750237.png" alt="image-20221005162750237" style="zoom: 50%;" />

**案例**

**队列层面的分配**

![image-20221005164556606](/Users/lukexwang/Library/Application Support/typora-user-images/image-20221005164556606.png)

**作业层面的分配**

![image-20221005164656030](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221005164656030.png)

- **<mark style="color:red">DRF策略</mark>**

  <mark style="color:red">DRF(Dominant Resource Fairness)</mark>，前面说的资源都是单一标准，例如只考虑内存(Yarn的默认情况)。但很多时候我们资源有多种，例如 内存、CPU、网络带宽等，这样我们很难衡量两个应用应该分配的资源比例。

  那么在YARN中，我们用DRF来决定如何调度：
  假设集群一共有100 CPU和10T 内存，而应用A需要(2 CPU, 300GB），应用B需要 (6 CPU, 100GB)
  则两个应用分别需要A （2%CPU, 3%内存）和B（6%CPU, 1%内存）的资源，这就意味着A是内存主导的，B是CPU主导的，针对这种情况，我们可以选择DRF策略对不同应用进行不同资源(CPU和内存）的一个不同比例的限制。 

#### yarn在生产环境下需要配置哪些参数

#### 命令行操作yarn

- `hadoop`的任务查看界面: hadoop100: 8088

**yarn application**

- `yarn application -list`: 查看任务列表

  ```shell
  $ yarn application -list
  2022-10-05 09:15:29,318 INFO client.RMProxy: Connecting to ResourceManager at hadoop100/10.211.55.6:8032
  Total number of applications (application-types: [], states: [SUBMITTED, ACCEPTED, RUNNING] and tags: []):0
                  Application-Id      Application-Name        Application-Type          User           Queue                   State             Final-State             Progress                        Tracking-URL
  ```

- `yarn application -list -appStates` (所有状态: ALL、NEW、NEW_SAVING、SUBMITTED、ACCEPTED、RUNNING、FINISHED、FAILED、KILLED)

  ````shell
  $ yarn application -list -appStates FINISHED
  ````

- `yarn application -kill {任务名}`: Kill掉Application

  ```shell
  $ yarn application -kill application_1660966238500_0001
  ```

**yarn logs**

- `yarn logs`查看日志:

  ```shell
  $ yarn logs -applicationId application_1660966238500_0001
  ```

- `yarn logs -applicationId <ApplicationId> -containerId <ContainerId>`: 查询Container的日志

  ```shell
  $ yarn logs -applicationId application_1660966238500_0003 -containerId container_1660966238500_0003_01_000001
  ```

**yarn applicationattempt: 查看尝试运行的任务**

- 列出所有Application尝试的列表: `yarn applicationattempt -list <ApplicationId>`

  ```shell
  $ yarn applicationattempt -list application_1660966238500_0003 2>/dev/null
  Total number of application attempts :1
           ApplicationAttempt-Id                 State                        AM-Container-Id                            Tracking-URL
  appattempt_1660966238500_0003_000001                FINISHED    container_1660966238500_0003_01_000001  http://hadoop100:8088/proxy/application_1660966238500_0003/
  ```

- 打印`ApplicationAttemp`状态: `yarn applicationattempt -status <ApplicationAttemptId>`

  ```shell
  $ yarn applicationattempt -status appattempt_1660966238500_0003_000001 2>/dev/null
  Application Attempt Report : 
          ApplicationAttempt-Id : appattempt_1660966238500_0003_000001
          State : FINISHED
          AMContainer : container_1660966238500_0003_01_000001
          Tracking-URL : http://hadoop100:8088/proxy/application_1660966238500_0003/
          RPC Port : 38121
          AM Host : hadoop100
          Diagnostics : 
  ```

**yarn container 容器查看(运行时候才能看到)**

- 列出所有Container: `yarn container -list <ApplicationAttemptId>`

  ```shell
  yarn container -list appattempt_1660966238500_0003_000001 2>/dev/null
  Total number of containers :0
                    Container-Id            Start Time             Finish Time                   State                    Host       Node Http Address                                LOG-URL
  ```

- 打印Container状态: `yarn container -status <ContainerId>`

  注意: **只有在运行中的任务才能看到 container的状态**

**yarn node: 查看节点状态**

- 列出所有节点: `yarn node -list -all`

  ```shell
  $ yarn node -list -all 2>/dev/null
  Total Nodes:3
           Node-Id             Node-State Node-Http-Address       Number-of-Running-Containers
   hadoop102:42931                RUNNING    hadoop102:8042                                  0
   hadoop101:34007                RUNNING    hadoop101:8042                                  0
   hadoop100:35589                RUNNING    hadoop100:8042                                  0
  ```

**yarn rmadmin 更新配置**

重新加载队列配置: `yarn rmadmin -refreshQueues`

**yarn queue查看队列**
打印队列信息: `yarn queue -status <QueueName>`

```shell
yarn queue -status default 2>/dev/null
Queue Information : 
Queue Name : default
        State : RUNNING
        Capacity : 100.00%
        Current Capacity : .00%
        Maximum Capacity : 100.00%
        Default Node Label expression : <DEFAULT_PARTITION>
        Accessible Node Labels : *
        Preemption : disabled
        Intra-queue Preemption : disabled
```

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221006140440653.png" alt="image-20221006140440653" style="zoom:50%;" />

#### yarn 生产环境核心参数

- **ResourceManager相关**
  - `yarn.resourcemanager.scheduler.class`: 配置调度器,默认容量
  - `yarn.resourcemanager.scheduler.client.thread-count`: ResourceManager处理调度器请求的线程数，默认50
- **NodeManager相关**
  - `yarn.nodemanager.resource.detect-hardware-capabilities` 是否让yarn自己检测硬件进行配置，默认false;
  - `yarn.nodemanager.resource.count-logical-processors-as-cores`: 是否让虚拟核数当做CPU核数，默认false
  - `yarn.nodemanager.resource.pcores-vcores-multiplier`: 虚拟核数和物理核数乘数，例如: 4核8线程，该参数就应该设置喂2，默认为`1.0`
  - `yarn.nodemanager.resource.memory-mb`: **NodeManager使用内存，默认8GB**
  - `yarn.nodemanager.resource.system-reserved-mb` **NodeManager为系统保留多少内存**
  - **以上两个参数配置一个即可**
  - `yarn.nodemanager.resource.cpu-vcores` NodeManager使用**CPU核数,默认8个**
  - `yarn.nodemanager.resource.pmem-check-enabled`: 是否开启物理内存检查限制container，默认打开
  - `yarn.nodemanager.resource.vmem-check.enabled`: 是否开启虚拟内存检查限制`container`，默认打开
  - `yarn.nodemanager.resource.vmem-pmem-ratio`: **虚拟内存物理内存比例，默认2:1**
- **Container相关**
  - `yarn.scheduler.minimum-allocation-mb` 容器最最小内存，默认1G
  - `yarn.scheduler.maximum-allocation-mb` 容器最最大内存，默认8G
  - `yarn.scheduler.minimum-allocation-vcores` 容器最小CPU核数，默认1个
  - `yarn.scheduler.maximum-allocation-vcores` 容器最大CPU核数，默认4个

