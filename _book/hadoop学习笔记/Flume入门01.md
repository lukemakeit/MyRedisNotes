### Flume

Flume是一个高可用的，高可靠的，分布式的**海量日志采集、聚合和传输的系统。** 用作采集文本文件的，不能采集视频、音频等。
Flume最主要的作用就是: **实时读取服务器本地磁盘的数据，将数据写入HDFS**
<img src="/Users/lukexwang/Library/Application Support/typora-user-images/image-20221007195553997.png" alt="image-20221007195553997" style="zoom:50%;" />

**架构图**
<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221007202911253.png" alt="image-20221007202911253" style="zoom:50%;" />

**Agent**
Agent是一个JVM进程，以事件的形式将数据从源头送到目的。
Agent主要由三部分组成: **Source、Channel、Sink**

**Source**
负责接收数据到Flume Agent组件。Source组件可以处理各种类型、各种格式的日志，如 `exec`、`http`、`netcat`等。

**Sink**
Sink不断轮询Chanel中的事件且批量移除他们，并将这些事件批量写入到存储 or 索引系统、或者被发送到另一个Flume Agent。
Sink组件的目的地包括**hdfs、logger(标准输出)、file、Hbase**等。

**Channel**
Channel是位于Source和Sink之间的缓冲区。因此，Channel允许Source和Sink运作在不同的速率上。Channel是线程安全的，可以同时处理几个Source的写入操作和几个Sink的读取操作。
Flume自带两种Channel: **Memory Channel 和 File  Channel。**

**Event**
传输单元，Flume数据传输的基本单元，以Event的形式将数据从源头送到目的地。
Event由**Header**和**Body**两部分组成，Header用来存放event的一些属性，为K-V结构，Body用来存放该条数据，形式为字节数组。
<img src="/Users/lukexwang/Library/Application Support/typora-user-images/image-20221007204414493.png" alt="image-20221007204414493" style="zoom:50%;" />

##### 安装

- 下载网址: [https://flume.apache.org/download.html](https://flume.apache.org/download.html)
- 下载: apache-flume-1.9.0-bin.tar.gz
- 解压后删除文件: apache-flume-1.9.0-bin/lib/guava-11.0.2.jar

##### 官方案例

1. 通过netcat工具向本机的9999端口发送数据

   ````shell
   $ netcat -lk 9999 #开启服务端
   ````

2. Flume监控本机的9999端口，通过Flume的source端读取数据;

3. Flume将获取的数据通过Sink端写出到控制台(`logger`);

   ```shell
   $ cd /usr/local/apache-flume-1.9.0-bin
   $ mkdir job
   $ cd job
   $ vim  net-flume-logger.conf
   # example.conf: A single-node Flume configuration
   
   # Name the components on this agent
   a1.sources = r1
   a1.sinks = k1
   a1.channels = c1
   
   # Describe/configure the source
   a1.sources.r1.type = netcat
   a1.sources.r1.bind = localhost
   a1.sources.r1.port = 44444
   
   # Describe the sink
   a1.sinks.k1.type = logger
   
   # Use a channel which buffers events in memory
   a1.channels.c1.type = memory
   a1.channels.c1.capacity = 1000 #容量, 最大事务个数
   a1.channels.c1.transactionCapacity = 100 #单次发送最多多少个events，一定要小于 capacity
   
   # Bind the source and sink to the channel # 绑定source、sink 和 channel的关系
   a1.sources.r1.channels = c1
   a1.sinks.k1.channel = c1
   ```

   - `a1`是agent的名字，同一台机器的多个任务需要不同的名字;
   - 一个source发送两个channel没问题;
   - 一个sink只能绑定一个channel;

4. 启动命令:
   ```shell
   $ ./bin/flume-ng agent -n a1 -c conf -f job/net-flume-logger.conf -Dflume.root.logger=INFO,console
   或
   $ ./bin/flume-ng agent --name a1 --conf conf --conf-file job/net-flume-logger.conf -Dflume.root.logger=INFO,console
   ```

**案例二: 实时读取本地文件到HDFS**

```java
/* job/file-to-hdfs.conf */
a1.sources = r1
a1.sinks = k1
a1.channels = c1

a1.sources.r1.type = exec
a1.sources.r1.command = tail -F /usr/local/apache-flume-1.9.0-bin/tmp/my.log

# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100

# Bind the source and sink to the channel
a1.sinks.k1.type = hdfs
a1.sinks.k1.channel = c1
a1.sinks.k1.hdfs.path = hdfs://hadoop101:8020/flume/my-%Y-%m-%d-%H%M%S
# 上传文件的前缀
a1.sinks.k1.hdfs.filePrefix = events-
# 是否按照时间滚动文件夹
a1.sinks.k1.hdfs.round = true
a1.sinks.k1.hdfs.roundValue = 1
# 重新定义时间单位
a1.sinks.k1.hdfs.roundUnit = second
# 是否使用本地时间戳
a1.sinks.k1.hdfs.useLocalTimeStamp = true
# 设置文件类型,可支持压缩
a1.sinks.k1.hdfs.fileType=DataStream
# 多久生成一个新的文件
a1.sinks.k1.hdfs.rollInterval=30
# 设置每个文件的滚动大小
a1.sinks.k1.hdfs.rollSize=13421770
# 文件的滚动与Event数量无关
a1.sinks.k1.hdfs.rollCount=0

# Bind the source and sink to the channel
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

运行命令:
```shell
$ ./bin/flume-ng agent -n a1 -c conf -f job/file-to-hdfs.conf
```

生成的结果是按秒出现的文件夹:
![image-20221009065559823](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221009065559823.png)

**缺点: 不支持断点续传**

**案例三: 实时读取目录文件到HDFS**

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221009070032601.png" alt="image-20221009070032601" style="zoom: 33%;" />

设置环境变量:
```shell
$ cat conf/flume-env.sh
export JAVA_OPTS="-Xms100m -Xmx256m  -Dcom.sun.management.jmxremote"
```

配置文件:

```java
/* job/flume-dir-to-hdfs.conf */
a3.sources = r3
a3.sinks = k3
a3.channels = c3

a3.sources.r3.type = spooldir
a3.sources.r3.spoolDir = /usr/local/apache-flume-1.9.0-bin/tmp
a3.sources.r3.fileSuffix = .COMPLETED
a3.sources.r3.fileHeader = true
#忽略以.tmp结尾的文件,不上传
a3.sources.r3.ignorePattern = ([^ ]*\.tmp)

# Describe the sink
a3.sinks.k3.type = hdfs
a3.sinks.k3.hdfs.path = hdfs://hadoop101:8020/flume/upload/my-%Y-%m-%d-%H
# 上传文件的前缀
a3.sinks.k3.hdfs.filePrefix = upload-
# 是否按照时间滚动文件夹
a3.sinks.k3.hdfs.round = true
a3.sinks.k3.hdfs.roundValue = 1
# 重新定义时间单位
a3.sinks.k3.hdfs.roundUnit = hour
# 是否使用本地时间戳
a3.sinks.k3.hdfs.useLocalTimeStamp = true
# 积攒多少个Event才flush到HDFS一次
a3.sinks.k3.hdfs.batchSize = 100
# 设置文件类型,可支持压缩
a3.sinks.k3.hdfs.fileType=DataStream
a3.sinks.k3.hdfs.writeFormat=Text
# 多久生成一个新的文件
a3.sinks.k3.hdfs.rollInterval=20
# 设置每个文件的滚动大小
a3.sinks.k3.hdfs.rollSize=134217700
# 文件的滚动与Event数量无关
a3.sinks.k3.hdfs.rollCount=0

#Use a channel which buffers events in memory
a3.channels.c3.type=memory
a3.channels.c3.capacity=1000
a3.channels.c3.transactionCapacity=100

# Bind the source and sink to the channel
a3.sources.r3.channels = c3
a3.sinks.k3.channel = c3
```

运行命令:
```shell
$ cd /usr/local/apache-flume-1.9.0-bin && ./bin/flume-ng agent -n a3 -c conf -f job/flume-dir-to-hdfs.conf -Dflume.root.logger=INFO,console
```

**注意**

1. 重名的 不会重复上传;
2. 已`.COMPLETED`为后缀的不会上传;
3. 直接修改已经存在`.COMPLETED`，不会再上传了;

#### 实时监控目录下的多个追加文件

- `Exec source`适用于监控一个实施追加的文件, 不能实现断点续传;
- `Spooldir source`适用于同步新文件，但不适合对实时追加的日志文件监听并同步;
- `Taildir Source`适用于监听多个追加的文件，并能断点续传;

**案例: 使用Flume监听整个目录的实时追加文件,并上传到HDFS**

#### Flume事务

**Put事务流程:**

- `dbPut`: 将批数据先写入临时缓冲区 putList;
- `doCommit`:检查channel内存队列是否足够合并;
- `doRollback`: channel内存队列空间不足，回滚数据

**Task事务:**

- `doTake`: 将数据取到临时缓冲区taskList, 并将数据发送到HDFS
- `doCommit`: 如果数据全部发送成功,则清除临时缓冲区taskList
- `doRollback`: 数据发送过程中如果出现异常, rollback将临时缓冲区taskList中的数据归还给channel内存队列

![image-20221017230709140](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221017230709140.png)

#### Flume Agent内部原理

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221017232439335.png" alt="image-20221017232439335" style="zoom:67%;" />

- `DefaultSinkProcessor`： 一个channel只能绑定一个sink;
- `LoadBalancingSinkProcessor`: 如一个channel绑定了 3个sink，则轮询发送给3个sink;
- `FailoverSinkProcessor`:  如一个channel绑定了 3个sink，在某个sink故障后，做故障转移。可以为不同的sink配置不同的优先级;

#### Flume 拓扑结构

##### 简单串联

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221017232725825.png" alt="image-20221017232725825" style="zoom:50%;" />

这种模式是将多个flume 顺序连接起来了,从最初的 source 开始到最终sink 传送的目的存储系统。
此模式不建议桥接过多的flume 数量,flume 数量过多不仅会影响传输速率，而且一旦传输过程中某个节点 flume 宕机，会影响整个传输系统。`AVRO RPC`: 通过ip:端口就能通信数据。

##### 复制和多路复用

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221017233359585.png" alt="image-20221017233359585" style="zoom:50%;" />

Flume支持将事件流向一个或者多个目的地。这种模式可以将相同数据复制到多个channel中 或者将不同数据分发到不同的channel中，sink可以选择传送到不同的目的地。

##### 负载均衡和故障转移

<img src="/Users/lukexwang/Library/Application Support/typora-user-images/image-20221017233630591.png" alt="image-20221017233630591" style="zoom:50%;" />

Flume支持使用将多个sink逻辑分到一个sink组，sink组配合不同的SinkProcessor。并发写提升了写性能。

##### 聚合

<img src="/Users/lukexwang/Library/Application Support/typora-user-images/image-20221017233934219.png" alt="image-20221017233934219" style="zoom:50%;" />



