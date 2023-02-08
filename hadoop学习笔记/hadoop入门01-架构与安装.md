### Hadoop的组成

- HDFS(Hadoop Distributed File System) 数据存储
- MapReduce: 计算
- YARN: 资源调度
- Common 辅助工具

**HDFS架构**

- NameNode(nn): 存储文件的元数据，如文件名，文件目录结构，文件属性(生成时间、副本数、文件权限)，以及每个文件的块列表和块所在的DataNode等;
- SecondaryNameNode(2nn): 每隔一段时间对NameNode保存的元数据进行备份;
- DataNode(dn): 在本地文件系统存储文件块数据 以及 块数据的校验和;

**YARN:资源管理器**

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220820160430885.png" alt="image-20220820160430885" style="zoom:50%;" />

- ResourceManager(RM): 整个集群资源(内存和CPU)的老大;
- NodeManager(NM): 单个节点服务器资源老大;
- ApplicationMaster(AM): 单个任务运行的老大;
- Container:容器, 相当一台独立的服务器。里面封装了任务运行所需的资源，如内存、cpu、磁盘、网络等;

说明

1. 客户端可以有多个;
2. 集群上可以运行多个ApplicationMaster;

**HDFS YARN MapReduce的关系**

![image-20220820161423884](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220820161423884.png)

**大数据生态体系**

**Zookeeper 数据平台配置和调度**

**数据来源层**

- 数据库MySQL等: 结构化数据，通过Sqoop组件做数据传递；
- 文件日志: 半结构化数据，通过Flume组件做日志手机;
- 视频、ppt等: 非结构化数据， Kafa消息队列;

**数据存储层**

- HDFS 文件存储
- HBase 非关系型数据库
- Kafka消息队列;

**资源管理层**

- YARN 资源管理

**数据计算层**

- MapReduce 离线计算
  - Hive 数据查询
- Spark Core 内存计算
  - Spark Mlib 数据挖掘
  - Spark SQL 数据查询
- 实时计算
  - Spark Streaming 实时计算
  - Flink

**任务调度层**

- 任务调度层
  - Oozie 任务调度
  - AzkaBan 任务调度

![image-20220820162900483](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220820162900483.png)

### 基础环境搭建

**创建虚拟机hadoop100、hadoop101、hadoop102;**

mac/windows本地配置`/etc/hosts`
```shell
10.211.55.6 hadoop100
10.211.55.7 hadoop101
10.211.55.8 hadoop102
```

jdk下载地址: [https://www.oracle.com/java/technologies/downloads/#java8](https://www.oracle.com/java/technologies/downloads/#java8)
(hive依赖java8，所以下载java8就好)

Haddop 下载地址: [https://hadoop.apache.org/releases.html](https://hadoop.apache.org/releases.html)
rsync命令:

```shell
rsync -av $pdir/$fname  $user@$host:$pdir/$fname
```

- `-a`: 归档拷贝;
- `-v`: 显示复制过程;

Ubuntu 默认用`ssh root`登录不了，配置一下:
```shell
echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
service ssh restart
```

- ResourceManager: hadoop100

- NameNode: hadoop101

- SecondaryNameNode: hadoop102

- **目录:`/usr/local/hadoop-3.2.4/etc/hadoop/`**


```xml
<!-- core-site.xml 文件 -->
<configuration>
        <!-- NameNode 的地址 -->
        <property>
                <name>fs.defaultFS</name>
                <value>hdfs://hadoop101:8020</value>
        </property>
        <!-- 指定hadoop数据的存储目录 -->
        <property>
                <name>hadoop.tmp.dir</name>
                <value>/data/hadoop/data</value>
        </property>
        <!-- 配置HDFS网页登录使用的静态用户为luke -->
        <property>
                <name>hadoop.http.staticuser.user</name>
                <value>root</value>
        </property>
</configuration>

<!-- hdfs-site.xml -->
<configuration>
        <!-- NameNode web端访问地址 -->
        <property>
                <name>df.namenode.http-address</name>
                <value>hadoop102:9870</value>
        </property>
        <!-- SecondaryNameNode web端访问地址 -->
        <property>
                <name>dfs.namenode.secondary.http-address</name>
                <value>hadoop102:9868</value>
        </property>
</configuration>

<!-- yarn-default.xml -->
<configuration>
        <!-- 指定MR走shuffle -->
        <property>
                <name>yarn.nodemanager.aux-services</name>
                <value>mapreduce_shuffle</value>
        </property>
        <property>
                <name>yarn.nodemanager.aux-services.mapreduce_shuffle.class</name>
                <value>org.apache.hadoop.mapred.ShuffleHandler</value>
        </property>
        <!-- 指定ResourceManager地址 -->
        <property>
                <name>yarn.resourcemanager.hostname</name>
                <value>hadoop100</value>
        </property>
        <!-- 环境遍历的继承 -->
        <property>
                <name>yarn.nodemanager.env-whitelist</name>
                <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_MAPRED_HOME</value>
        </property>
</configuration>

MapReduce配置文件: mapred-site.xml
<configuration>
        <!-- 指定MapReduce程序运行在yarn上 -->
        <property>
                <name>mapreduce.framework.name</name>
                <value>yarn</value>
        </property>
</configuration>
```

**NameNode:hadoop101 需要设置到 hadoop100、hadoop101、hadoop102的免密登录**
workers:

```shell
hadoop100
hadoop101
hadoop102
```

NameNode: hadoop101 上执行:
```shell
$ hdfs namenode -format //初始化namenode的hdfs磁盘
# 生成内容如下
$ cat /data/hadoop/data/dfs/name/current/VERSION 
#Wed Aug 17 15:20:57 UTC 2022
namespaceID=497654228
blockpoolID=BP-949790399-10.211.55.7-1660749657510
storageType=NAME_NODE
cTime=1660749657510
clusterID=CID-abd3e968-3717-4210-b03b-c2470bdf6b82
layoutVersion=-65
```

启动集群:

```shell
# 所有hadoop节点加入 JAVA_HOME
cd /usr/local/hadoop-3.2.4 && vim etc/hadoop/hadoop-env.sh
export JAVA_HOME="/usr/local/jdk1.8.0_351"

#一些变量
export HDFS_NAMENODE_USER="root"
export HDFS_DATANODE_USER="root"
export HDFS_SECONDARYNAMENODE_USER="root"
export YARN_RESOURCEMANAGER_USER="root"
export YARN_NODEMANAGER_USER="root"

#NameNode上启动集群
cd /usr/local/hadoop-3.2.4
./sbin/start-dfs.sh
```

此时可以在Web上访问NameNode的9870端口:
<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220819070433082.png" alt="image-20220819070433082" style="zoom:50%;" />

**在hadoop100上启动ResourceManager**

```shell
export HDFS_NAMENODE_USER="root"
export HDFS_DATANODE_USER="root"
export HDFS_SECONDARYNAMENODE_USER="root"
export YARN_RESOURCEMANAGER_USER="root"
export YARN_NODEMANAGER_USER="root"

cd /usr/local/hadoop-3.2.4
./sbin/start-yarn.sh
```

查看ResourceManager(hadoop100:8088):
<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220819070843189.png" alt="image-20220819070843189" style="zoom:50%;" />

集群基本测试:
```shell
# hadoop101 创建目录 并 上传文件
$ hadoop fs -mkdir /input
$ hadoop fs -put /home/luke/jdk-11.0.16_linux-aarch64_bin.tar.gz /input
```

查看变化: 
<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220820092000559.png" alt="image-20220820092000559" style="zoom:50%;" />

![image-20220820092740138](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220820092740138.png)

实际数据存储在什么地方?
根据下面的配置, 在 NameNode的`/data/hadoop/data`下。

```xml
        <!-- NameNode 的地址 -->
        <property>
                <name>fs.defaultFS</name>
                <value>hdfs://hadoop101:8020</value>
        </property>
        <!-- 指定hadoop数据的存储目录 -->
        <property>
                <name>hadoop.tmp.dir</name>
                <value>/data/hadoop/data</value>
        </property>
```

执行一些计算任务，让yarn这个资源管理器也用起来
```shell
$ hadoop fs -mkdir /wcoutput
$ hadoop fs -put /home/luke/words /wcinput #上传文件
$ hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.2.4.jar wordcount /wcinput /wcoutput # /wcoutput不能已存在
```

![image-20220820102516685](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220820102516685.png)

**hadoop集群重新初始化的步骤**

- 到每台机器上删除`/usr/local/hadoop-3.2.4/logs`、`/data/hadoop/data/`(根据配置文件来的);

- 重新格式化: `hdfs namenode -format`;

- 重启hdfs 和 yarn:
  ```shell
  # NameNode: hadoop101
  cd /usr/local/hadoop-3.2.4
  ./sbin/start-dfs.sh
  
  # ResourceManager: hadoop100
  cd /usr/local/hadoop-3.2.4
  ./sbin/start-yarn.sh
  ```

- 注意: 每个DataNode的版本号，只有相互认识(ClusterID相同)才能干活

  ```shell
  cat /data/hadoop/data/dfs/data/current/VERSION 
  #Sat Aug 20 02:02:17 UTC 2022
  datanodeUuid=b6a15fc0-20be-478b-a141-a6577fad8c72
  storageType=DATA_NODE
  cTime=0
  clusterID=CID-abd3e968-3717-4210-b03b-c2470bdf6b82
  layoutVersion=-57
  storageID=DS-4385f79a-f913-4586-bd23-e7b454bfe909
  ```

#### 开启历史服务器

修改配置文件:`etc/hadoop/mapred-site.xml`。修改后将该配置文件分发到其他机器上;
```xml
        <!-- 历史服务器地址 -->
        <property>
                <name>mapreduce.jobhistory.address</name>
                <value>hadoop100:10020</value>
        </property>
        <!-- 历史服务器web端地址 -->
        <property>
                <name>mapreduce.jobhistory.webapp.address</name>
                <value>hadoop100:19888</value>
        </property>
```

启动历史服务器:
```shell
$ mapred --daemon start historyserver
```

查看JobHistory:

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220820110137917.png" alt="image-20220820110137917" style="zoom:50%;" />

##### 文件聚集功能

日志聚集功能的好处: 方便的查看运行程序的详情，方便开发和调试。

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220820110427987.png" alt="image-20220820110427987" style="zoom:50%;" />

注意: 开启日志聚集功能，需要重启 NodeManager、ResourceManager 和 HistoryServer。

文件: `etc/hadoop/yarn-site.xml`

```xml
        <!-- 开启日志聚集功能 -->
        <property>
                <name>yarn.log-aggregation-enable</name>
                <value>true</value>
        </property>
        <!-- 开启日志聚集服务器地址 -->
        <property>
                <name>yarn.log.server.url</name>
                <value>http://hadoop100:19888/jobhistory/logs</value>
        </property>
        <!-- 日志保留时间设置7天=604800秒 -->
        <property>
                <name>yarn.log-aggregation.retain-seconds</name>
                <value>604800</value>
        </property>
```

重启:

```shell
./sbin/stop-yarn.sh
./sbin/start-yarn.sh

mapred --daemon  stop historyserver
mapred --daemon  start historyserver
```

重新执行一个wordcount任务:
```shell
$ hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.2.4.jar wordcount /wcinput /wcoutput
```

![image-20220820113517101](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220820113517101.png)

**启动/暂停总结**

1) 各个模块分开启动和停止

   HDFS

   ```shell
   cd /usr/local/hadoop-3.2.4
   ./sbin/stop-dfs.sh
   ./sbin/start-dfs.sh
   ```

   YARN

   ```shell
   cd /usr/local/hadoop-3.2.4
   ./sbin/stop-yarn.sh
   ./sbin/start-yarn.sh
   ```

2) 各个服务组件逐一启动/暂停

   HDFS 中的一个组件

   ```shell
   hdfs --daemon start/stop NameNode/DataNode/SecondaryNameNode
   ```

   启动/暂停YARN

   ```shell
   yarn --daemon start/stop ResourceManager/NodeManager
   ```

**常用端口号**

1. HDFS NameNode 内部常用端口号: 8020/9000/9820

   HDFS NameNode 对用户的查询端口号: 9870

   Yarn 查看任务运行情况: 8088

   历史服务器: 19888
   
   - HDFS  webUI端口: [http://hadoop101:9870/](http://hadoop101:9870/), 服务端口:`hadoop101:8020` 
   - Yarn webUI端口: [http://hadoop100:8088/cluster](http://hadoop100:8088/cluster)，服务端口号: `hadoop100:8032`
   - 历史服务 webUI端口: [http://hadoop100:19888/jobhistory/app，服务端口号: `hadoop:10020`](http://hadoop100:19888/jobhistory/app)

**集群时间同步**

1. 如果服务器能连接外网，则不需要时间同步;

2. 服务器连接不了外网，则需要在多个节点间挑选一条时间服务器

   ntpd 服务

   ```
   systemctl status ntpd
   systemctl start ntpd
   systemctl is-enabled ntpd
   ```

   修改某台机器的ntp.conf配置文件

   ```
   vim /etc/ntpd.conf
   ```

