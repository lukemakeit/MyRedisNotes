Flink提交作业和执行任务，需要几个关键组件: 客户端(client)、作业管理器(JobManager) 和 任务管理器(TaskManager)配合;

- 我们的代码由客户端获取并做转换，之后提交给JobManager;
- **JobManager是Flink集群的 管事人，对作业进行中央调度管理**；
- **她获取到执行的作业后，进一步处理转换，然后分发任务给众多的TaskManager**；
- **TaskManager是真正 干活的人，数据的处理操作都是它们来做的**；

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221026061957352.png" alt="image-20221026061957352" style="zoom:50%;" />

#### 集群启动Flink

1. 下载`1.3.6`版本, 地址: [https://flink.apache.org/downloads.html#all-stable-releases](https://flink.apache.org/downloads.html#all-stable-releases)

2. 各个机器对应的角色:

   - hadoop101=> JobManager
   - hadoop100 => TaskManager
   - Hadoop102 => TaskManager

3. 相关配置:

   ```shell
   cd /usr/local/flink-1.13.6
   vim conf/flink-conf.yaml
   
   jobmanager.rpc.address: hadoop101
   jobmanager.rpc.port: 6123
   jobmanager.memory.process.size: 1024m
   taskmanager.memory.process.size: 1024m
   taskmanager.numberOfTaskSlots: 1
   parallelism.default: 1
   
   vim conf/masters
   hadoop101:8081
   
   vim conf/workers
   hadoop100
   hadoop102
   ```

   上面的配置需要拷贝到 `hadoop100`、`hadoop102`机器上:

   ```shell
   scp -r flink-1.13.6 root@hadoop100:/usr/local/
   scp -r flink-1.13.6 root@hadoop102:/usr/local/
   ```

   所有机器上配置环境变量:

   ```shell
   echo "" >>/etc/profile
   echo "export FLINK_HOME=/usr/local/flink-1.13.6" >>/etc/profile
   echo "export PATH=\$FLINK_HOME/bin:\$PATH" >>/etc/profile
   ```

4. 启动集群:

   ```shell
   $ cd /usr/local/flink-1.13.6
   $ ./bin/start-cluster.sh 
   Starting cluster.
   [INFO] 1 instance(s) of standalonesession are already running on hadoop101.
   Starting standalonesession daemon on host hadoop101.
   Starting taskexecutor daemon on host hadoop100.
   Starting taskexecutor daemon on host hadoop102.
   ```

5. 关闭集群:

   ```shell
   cd /usr/local/flink-1.13.6
   ./bin/stop-cluster.sh
   ```

6. 查看日志，如果`JobManager`或`TaskManager`没有正常启动，都可以在相关机器的以下位置查看数据

   ```shell
   $ cd /usr/local/flink-1.13.6
   $ ls -lrht log/*
   $ tail log/flink-root-taskexecutor-0-hadoop100.log
   ```

   

**错误补充:**

1. `TaskManager.sh`启动失败，错误信息:

   ```java
   Error: VM option 'UseG1GC' is experimental and must be enabled via -XX:+UnlockExperimentalVMOptions.
   Error: Could not create the Java Virtual Machine.
   Error: A fatal exception has occurred. Program will exit.
   ```

   删除`-XX:+UseG1GC`信息:

   <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221027062520336.png" alt="image-20221027062520336" style="zoom:50%;" />

**打包并提交任务**

- 打包: `mvn package`

- 在`hadoop100`机器上启动`nc`

  ```java
  $ nc -lk 7777
  ```

- 提交任务:

  ```shell
  $ cd /usr/local/flink-1.13.6
  $ ./bin/flink run -m hadoop101:8081 -c com.luke.flink.wordcount.StreamWordCount -p 2 /root/code/java/flink-demo01/target/flink-demo01-0.0.1-snapshot.jar
  Job has been submitted with JobID 3ac051c651fb7eaa0b4ff71db858167d
  ```

- 在`nc`中输入信息:

  ```shell
  $ nc -lk 7777
  hello world
  hello java
  hello flink
  ```

- 在页面的TaskManager上查看`stdout`的结果:

  <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221027065121110.png" alt="image-20221027065121110" style="zoom: 33%;" />

  <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221027065159754.png" alt="image-20221027065159754" style="zoom:33%;" />

- 取消任务:

  - 页面上取消任务:

    <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221027065446564.png" alt="image-20221027065446564" style="zoom:50%;" />

  - 命令行做cancel

    ```shell
    $ ./bin/flink cancel 3ac051c651fb7eaa0b4ff71db858167d
    ```

### 部署模式

区别主要在于: 集群的生命周期以及资源的分配方式；以及应用的main方法到底在哪里执行——**客户端(client)还是jobmanager**。

#### 会话模式(session mode)

最符合常规思维。
先启动一个集群，保持一个会话，在该会话中通过client提交作业。
作业启动时所有资源已经确定，所以所有提交的作业会竞争集群中的资源。
会话模式适合 单个规模小、执行时间短的大量作业。

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220704062100591.png" alt="image-20220704062100591" style="zoom:33%;" />

优点:

- 集群不根据作业状态而改变;

缺点:

- 资源共享，资源如果不够，提交作业失败;

#### 单作业模式(pre-job mode)

每个提交的作业启动一个集群，即为单作业模式(pre-job mode)。同样由客户端运行应用程序，然后启动集群，作业被提交给JobManager，进而分发给TaskManager执行。作业完成，集群关闭，所有资源会释放。
这些特性使得单作业模式运行更稳定，所以实际应用首选单作业模式。
注意: Flink本身无法直接如此运行，所以单作业模式一般需借助一些资源管理框架来启动，如yarn、kubernetes。

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220704062737413.png" alt="image-20220704062737413" style="zoom:33%;" />

#### 应用模式(Application mode)

前面提到的两种模式下，**应用代码都是在客户端上(flink)上执行，将作业拆分出来，然后由客户端提交给JobManager。但这种客户端需要占用大量网络带宽，去下载依赖和把二进制数据传给JobManage**r；加上很多情况下我们提交作业用的是一个客户端，就会加重客户端所在节点的资源消耗。

解决方法是，我们不再需要客户端，直接把应用提交到JobManager上运行。这也代表着我们需要为每个提交的应用单独启动一个JobManager，也就是创建一个集群。这个 JobManager 只为这一个应用而存在，执行结束后JobManager就关闭了，这就是所谓的应用模式。

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220704064040550.png" alt="image-20220704064040550" style="zoom:33%;" />

应用模式和单作业模式，都是提交作业之后才创建集群。**单作业模式是通过客户端提交的，客户端解析出的每个作业对应一个集群；而应用模式下，是直接由JobManager 执行应用程序的，并且即使应用包含了多个作业，也只创建一个集群**。



下面说下**不同的资源提供者(Resource Provider)的场景**，介绍Flink的部署方式。

#### 独立模式(Standalone), 只有flink,不借助任务资源管理平台

独立模式(Standalone)是部署Flink最基本也是最简单的方式：所需的所有Flink组件，都只是操作系统运行的一个JVM进程。我们上面启动Flink集群就是独立模式。
独立模式是单独运行的，不依赖任何外部资源管理平台。代价：如果资源不足 或 出现故障，没有自动扩展或重新分配资源的保证，必须手动处理。所以独立模式一般只用在开发测试环境 或 作业非常少的场景。

- **会话模式(session mode)**
视频里的集群就是 独立(Standalone)集群的会话模式部署。

- **单作业模式(pre-job mode)**
  Flink无法以单作业方式启动集群，一般需要借助一些资源管理平台。所以Flink的独立模式中(Standalone)没有单作业(pre-job mode)模式部署。

- **应用模式部署(application mode,使用很少)**
  应用模式下不会提前创建集群，所以不能用`start-cluster.sh`脚本。我们可以使用同样在bin目录下的`standalone-job.sh`来创建一个JobManager。
  具体步骤:

1. 进入到Flink的安装目录下，将应用程序的jar包放到lib/目录下:

   ```shell
   cp ./FlinkTutorial-1.0-SNAPSHOT.jar lib/
   ```

2. 执行以下命令，启动`JobManager`:

   ```java
   ./bin/standalone-job.sh start --job-classname com.atguigu.wc.StreamWordCount
   ```

   这里直接指定了作业入口类，脚本会到lib目录扫描所有jar包。

3. 同样是使用bin目录下的脚本，启动`TaskManager`

   ```python
   ./bin/taskmanager.sh start
   ```

4. 如果希望停掉集群，同样使用脚本，命令:

   ```shell
   ./bin/standalone-job.sh stop
   ./bin/taskmanager.sh stop
   ```

#### YARN模式

YARN上部署的过程是: 客户端把Flink应用提交给Yarn的ResourceManager，Yarn的ResourceManager会向Yarn的NodeManager申请容器。在这些容器中，Flink会部署 JobManager 和 TaskManager的实例，从而启动集群。Flink会根据运行在JobManager上的作业所需的Slot数量动态分配 TaskManager资源。
