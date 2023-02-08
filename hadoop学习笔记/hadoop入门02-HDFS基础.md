## HDFS

#### 概述

HDFS(Hadoop Distributed File System), **它是一个分布式文件管理系统**，用于存储文件，通过目录树来定位文件。
其次，它是分布式的，由多台服务器联合实现其功能，集群中的服务有各自的角色。
HDFS使用场景: 一次写入，多次读出的场景。

**优点**

- 容错性
  - 数据自动保存多个副本，它通过增加副本的形式，提高容错性
  - 某一个副本丢失以后，它可以自动恢复
- 适合处理大数据
  - 数据规模：能够处理数据规模达到GB、TB 甚至 PB级别的数据
  - 文件规模： 能处理百万规模以上的文件数量
- 可以再廉价机器上构建，通过多副本机制，提高可靠性

**缺点**

1. 不适合低延迟数据访问，比如毫秒级的存储数据，是做不到的
2.  无法高效的对大量小文件进行存储
   - 存储大量小文件的话，它会占用NameNode大量的内存来存储文件目录和块信息。这样是不可取的，NameNode的内存总是有限的；每个文件块大概150 bytes；
   - 小文件存储的寻址时间会超过读取时间，违反了HDFS的设计目标；
3. 不支持并发写入、文件随机修改
   - 一个文件只能有一个写，不允许多个线程同时写；
   - **紧支持数据append(追加)，不支持文件的随机修改**

#### 架构

![image-20220821091551273](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220821091551273.png)

**NameNode(nn): 就是manager，它是一个主管、管理者**

1. 管理HDFS的Meta信息;
2. 不同的文件可配置副本策略；
3. 管理数据块(Block)映射信息，一个文件可能是多个文件块，每个文件块有不同的副本，存放在不同的DataNode上；
4. 处理客户端请求；

**DataNode:就是executer,NameNode下达命令，DataNode执行实际操作**

1. 存储实际的数据块
2. 执行数据块的读/写操作

**Client: 客户端**

1. **文件切分**。文件上传HDFS时，Client将文件分成一个一个的Block，然后进行上传。

   比如1GB的文件需要拆分，就是client完成的，按照NameNode的Block大小来拆分。Block 一般是128MB，1GB的文件就是8个Block；

2. 与NameNode交互，获取文件的位置信息;

3. 与DataNode交互,读取或写入数据;

4. Client提供一些命令来管理HDFS，比如NameNode格式化；

5. Client可通过一些命令来访问HDFS，比如对HDFS增删查改的操作；

**Secondary NameNode: 并非NameNode的热备。当NameNode挂掉时，它并不能马上替换NameNode进行提供服务**

1. 辅助NameNode，分担其工作量，比如定期合并Fsimage和Edits，并推送给NameNode；
2. 紧急情况下，可辅助恢复NameNode;

#### HDFS文件块大小

HDFS的文件在物理机上是分开存储(Block)，块的大小可通过配置参数(dfs.blocksize)来规定，**默认大小在Hadoop 2.x/3.x中是128MB，1.x版本是64M**。一个2MB的文件是不会完整占用一个128MB的Block的，128MB表示Block最大能存储的空间。

1. 集群中的block;

2. 如果寻址时间约为10ms，即查找到目标block的时间为10ms；

3. **寻址时间为传输时间的 1%时，则为最佳状态(专家)。因此传输时间=10ms/0.01=1s**

4. 目前磁盘的传输速率普遍为100MB/s;

5. block大小= 1s*100MB/s=100MB;

   如果DataNode用的是SSD硬盘，则设置blocksize=256MB;

**为何块大小不能设置太小，也不能设置太大？**

1. **HDFS块设置太小，会增加寻址时间，程序一直在找块的开始位置**；
2.  **如果块设置太大，从磁盘传输数据的时间会明显大于定位这个块开始位置所需的时间。导致程序在处理这块时，会非常慢**;
3. 总结起来说： HDFS块的大小主要取决于磁盘的传输速率。

### HDFS的shell操作

基本命令: `hadoop fs`具体命令 OR `hdfs dfs`具体命令，两者完全相同。

#### 常用命令

**`help`帮助命令: 使用方式如`hadoop fs -help rm`**

**创建文件夹: `hadoop fs -mkdir /sanguo`**

**上传**

- `-moveFromLocal: 从本地剪切粘贴到HDFS`

  ```shell
  $ echo "shuguo" >> shuguo.txt
  $ hadoop fs -moveFromLocal ./shuguo.txt /sanguo
  ```

- `-copyFromLocal:从本地复制粘贴到HDFS`

  ```sh
  $ echo "weiguo" >> weiguo.txt
  $ hadoop fs -copyFromLocal ./weiguo.txt /sanguo
  ```

- `-put`:等同于`-copyFromLocal`,生成环境更习惯用`put`

- `-appendToFile`:追加一个文件到已存在文件的末尾

  ```shell
  $ echo "liubei">> liubei.txt
  $ hadoop fs -appendToFile ./liubei.txt /sanguo/shuguo.txt
  ```

**下载**

- `-copyToLocal`:从`HDFS`拷贝到本地

  ```shell
  $ hadoop fs -copyToLocal /sanguo/shuguo.txt ./
  $ cat shuguo.txt 
  shuguo
  liubei
  ```

- `-get`:等同于`-copyToLocal`,生产环境更习惯用`get`

  ```shell
  $ hadoop fs -get /sanguo/shuguo.txt ./shuguo2.txt
  ```

**直接操作**

- `-ls`:显示目录信息

  ```shell
  $ hadoop fs -ls /sanguo
  Found 3 items
  -rw-r--r--   3 root supergroup         14 2022-08-21 02:13 /sanguo/shuguo.txt
  -rw-r--r--   3 root supergroup          7 2022-08-21 02:09 /sanguo/weiguo.txt
  -rw-r--r--   3 root supergroup          6 2022-08-21 02:15 /sanguo/wuguo.txt
  ```

- `-cat`:显示文件内容

  ```shell
  $ hadoop fs -cat /sanguo/shuguo.txt
  shuguo
  liubei
  ```

- `-chgrp`、`-chmod`、`-chown`: 与Linux系统中使用方法相同

  ```shell
  $ hadoop fs -ls /sanguo
  Found 3 items
  -rw-r--r--   3 root supergroup         14 2022-08-21 02:13 /sanguo/shuguo.txt
  -rw-r--r--   3 root supergroup          7 2022-08-21 02:09 /sanguo/weiguo.txt
  -rw-r--r--   3 root supergroup          6 2022-08-21 02:15 /sanguo/wuguo.txt
  
  $ hadoop fs -ls /sanguo
  Found 3 items
  -rw-rw-rw-   3 root supergroup         14 2022-08-21 02:13 /sanguo/shuguo.txt
  -rw-r--r--   3 root supergroup          7 2022-08-21 02:09 /sanguo/weiguo.txt
  -rw-r--r--   3 root supergroup          6 2022-08-21 02:15 /sanguo/wuguo.txt
  ```

- `-mkdir`: 创建路径

  ```shell
  $ hadoop fs -mkdir /jinguo
  ```

- `-cp`: 从HDFS的一个路径拷贝到HDFS的另一个路径

  ```shell
  $ hadoop fs -cp /sanguo/shuguo.txt /jinguo
  
  $ hadoop fs -ls /jinguo
  Found 1 items
  -rw-r--r--   3 root supergroup         14 2022-08-21 02:29 /jinguo/shuguo.txt
  ```

- `-mv`:在HDFS目录中移动文件，使用方法和`-cp`类似;

  ```shell
  $ hadoop fs -mv /sanguo/weiguo.txt /jinguo
  $ hadoop fs -mv /sanguo/wuguo.txt /jinguo
  ```

- `-tail`:显示一个文件的末尾1KB的数据

  ```shell
  $ hadoop fs -tail /jinguo/shuguo.txt
  shuguo
  liubei
  ```

- `-rm`: 删除文件or文件夹

  ```shell
  $ hadoop fs -rm /jinguo/shuguo.txt
  Deleted /jinguo/shuguo.txt
  ```

- `-rm -r`:递归删除目录和目录中的内容

  ```shell
  $ hadoop fs -ls /sanguo
  Found 1 items
  -rw-rw-rw-   3 root supergroup         14 2022-08-21 02:13 /sanguo/shuguo.txt
  
  $ hadoop fs -rm -r /sanguo
  Deleted /sanguo
  ```

- `-du`:统计文件夹的大小信息

  ```shell
  $ hadoop fs -du -s -h /jinguo
  13  39  /jinguo
  
  $ hadoop fs -du  -h /jinguo
  7  21  /jinguo/weiguo.txt
  6  18  /jinguo/wuguo.txt
  ```

  说明: 13表示文件大小；39表示13*3副本大小；

- `-setrep`: 设置HDFS中文件的副本数

  ```shell
  $ hadoop fs -ls /jinguo/weiguo.txt
  -rw-r--r--   3 root supergroup          7 2022-08-21 02:09 /jinguo/weiguo.txt
  
  $ hadoop fs -setrep 5 /jinguo/weiguo.txt
  
  $ hadoop fs -ls /jinguo/weiguo.txt
  -rw-r--r--   5 root supergroup          7 2022-08-21 02:09 /jinguo/weiguo.txt
  ```

  这里设置副本数为5，只是更新了NameNode中的元数据，是否真的有这么多副本，还得看DataNode的数量。
  因为目前只有3台设备，最多也就是3个副本，只有节点数的增加到5台时，副本数才能到5。

  ![image-20220821110757301](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220821110757301.png)

### HDFS API 访问

基础代码:
```java
import java.io.IOException;
import java.net.URI;
import java.net.URISyntaxException;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;

public class App {
    public static void main(String[] args) throws URISyntaxException, IOException, InterruptedException {
        System.out.println("Hello World!");
        // 连接集群的NameNode地址
        URI uri = new URI("hdfs://hadoop101:8020");
        // 创建一个配置文件
        Configuration conf = new Configuration();
        // 获取客户端对象
        String user = "root";
        // 注意: FileSystem是一个抽象类,无法直接new FileSystem
        FileSystem fs = FileSystem.get(uri, conf, user);

        Path src = new Path("/data/luke_tmp/laosun.txt");
        Path dst = new Path("/xiyou/huaguoshan");

        // 创建一个文件夹
        // fs.mkdirs(new Path("/xiyou/huaguoshan"));

        // 上传文件
        fs.copyFromLocalFile(false, false, src, dst);
        // 下载文件夹
        // fs.copyToLocalFile(false, new Path("/xiyou/huaguoshan"), new
        // Path("/data/luke_tmp"), true);

        // 删除目录
        // fs.delete(new Path("/xiyou"), true);

        // 移动 or 文件重命名
        // fs.rename(new Path("/xiyou/huaguoshan"), new Path("/xiyou/shuiliandong"));

        // 获取文件详情(权限,大小等)
        RemoteIterator<LocatedFileStatus> listFiles = fs.listFiles(new Path("/xiyou/shuiliandong/laosun.txt"), true);
        // 遍历迭代器
        while (listFiles.hasNext()) {
            LocatedFileStatus status = listFiles.next();
            System.out.println("======" + status.getPath() + "========");
            System.out.println(status.getPermission());
            System.out.println(status.getOwner());
            System.out.println(status.getGroup());
            System.out.println(status.getModificationTime());
            System.out.println(status.getReplication());
            System.out.println(status.getBlockSize());
            if (status.isFile()) {
                System.out.println("这是一个文件");
            }

            // 获取文件的块信息
            BlockLocation[] locs = status.getBlockLocations();
            System.out.println(locs);
        }
      
        // 关闭资源
        fs.close();
    }
}
```

**默认上传的文件replication是3,如果想修改这些属性，可通过如下方式修改:**

- 全局的配置: `/usr/local/hadoop-3.2.4/etc/hadoop/hdfs-site.xml`

- 项目本地配置: `src/main/resources/hdfs-site.xml`

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <?xml-stylesheet href="configuration.xsl" type="text/xsl"?>
  
  <configuration>
      <property>
          <name>dfs.replication</name>
          <value>1</value>
      </property>
  </configuration>
  ```

- 项目内代码配置

  ```java
   // 创建一个配置
   Configuration conf = new Configuration();
   conf.set("dfs.replication", "2");
  ```


#### HDFS的读写流程

##### 写数据流程

![image-20220917105526291](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220917105526291.png)

1. **客户端发送给DataNode, 先用chunk 512byte + chunksum 4byte, 构造成一个packet (64KB), 而后进行发送。每次发送一个packet对象**

2. client中有Ack队列? Ack队列在Packet应答成功后再删除;

3. 节点距离的计算？

   **在第4步中, NameNode给client返回的DataNode信息，会优先选择本地 or 靠近client的DataNode**

4. 副本存储节点选择

   - **第一个副本选择 本地 or 靠近client的DataNode**
   - **第二个副本选择不同机架的 DataNode，最后一个副本选择 和 第二个副本相同机架的DataNode**
   - 当然在选择副本过程中也会考虑DataNode的负载情况，继续确定是否选择该DataNode

   <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220917111104130.png" alt="image-20220917111104130" style="zoom:50%;" />

##### 读数据流程

1. **从哪个DataNode读数据一方面考虑DataNode和client的距离远近,另一方面考虑 DataNode的负载情况**
2. 从DataNode1读到第一个数据块(Block)之后，再从DataNode2读取第二个数据块，本地做追加。而不是并行同时读取两个数据块；

![image-20220917113337804](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220917113337804.png)

#### NameNode 和 2NameNode

- **fsimage存储数据**

- Edits记录追加

  记录信息为: a+10、a-30、a*20等

- 2NameNode会对 fsimage 和 Edits 数据进行合并

```shell
root@hadoop101:~# ls -lrht /data/hadoop/data/dfs/name/current/|tail -10
-rw-r--r-- 1 root root   42 Sep 13 22:33 edits_0000000000000000761-0000000000000000762
-rw-r--r-- 1 root root   42 Sep 13 23:33 edits_0000000000000000763-0000000000000000764
-rw-r--r-- 1 root root 1.6K Sep 15 15:50 edits_0000000000000000765-0000000000000000785
-rw-r--r-- 1 root root   62 Sep 15 15:50 fsimage_0000000000000000785.md5
-rw-r--r-- 1 root root 4.7K Sep 15 15:50 fsimage_0000000000000000785
-rw-r--r-- 1 root root  829 Sep 17 02:49 edits_0000000000000000786-0000000000000000797
-rw-r--r-- 1 root root    4 Sep 17 02:49 seen_txid
-rw-r--r-- 1 root root 1.0M Sep 17 02:49 edits_inprogress_0000000000000000798
-rw-r--r-- 1 root root   62 Sep 17 02:49 fsimage_0000000000000000797.md5
-rw-r--r-- 1 root root 4.7K Sep 17 02:49 fsimage_0000000000000000797
```

![image-20220917115621543](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220917115621543.png)

**第一个阶段: NameNode启动**

1) 第一次启动NameNode格式化后,创建Fsimage和Edits文件。如果不是第一次启动，直接加载编辑Edits文件和Fsimage文件到内存;
2) 客户端对元数据进行增删改的请求;
3) NameNode记录操作日志到Edits文件, 更新滚动日志
4) NameNode在内存中对元数据进行增删改;

**第二阶段:Secondary NameNode的工作**

1. Secondary NameNode询问NameNode是否需要CheckPoint，直接带回NameNode是否检查结果;
2. Secondary NameNode请求执行CheckPoint;
3. NameNode滚动正在写的Edits日志;
4. 将滚动前的Edits日志和Fsimage文件拷贝到Secondary NameNode;
5. Secondary NameNode 加载Edits日志和Fsimage文件到内存，并合并;
6. 生成新的镜像文件Fsimage.chkpoint;
7. 拷贝Fsimage.chkpoint 到NameNode;
8. NameNode将Fsimage.chkpoint重新命名成Fsimage;

##### Fsimage和Edits解析

NameNode被格式化后,将在`/data/hadoop/data/dfs/name/current/`目录中产生如下文件:
这个目录信息是由配置项`<name>hadoop.tmp.dir</name>`决定的, `core-site.xml`文件中。

```text
-rw-r--r-- 1 root root 1048576 Sep 17 06:49 edits_inprogress_0000000000000000806
-rw-r--r-- 1 root root    4790 Sep 17 05:49 fsimage_0000000000000000803
-rw-r--r-- 1 root root      62 Sep 17 05:49 fsimage_0000000000000000803.md5
-rw-r--r-- 1 root root    4790 Sep 17 06:49 fsimage_0000000000000000805
-rw-r--r-- 1 root root      62 Sep 17 06:49 fsimage_0000000000000000805.md5
-rw-r--r-- 1 root root       4 Sep 17 06:49 seen_txid
-rw-r--r-- 1 root root     214 Aug 20 02:58 VERSION
```

1. Fsimage文件: HDFS文件系统元数据的一个**永久性检查点**，其中包含HDFS文件系统的所有目录和文件inode的序列化信息;

2. Edits文件: 存放HDFS文件系统的所有更新操作的路径，文件系统客户端指定的所有写操作首先记录到Edits文件中;

3. seen_txid 文件保存的是一个数字，就是最后一个`edits_`的数字;

4. 每次NameNode**启动的时候**都会将Fsimage文件读入内存，加载Edits里面的更新操作，保证内存中的元数据信息是最新的、同步的，可以看成NameNode启动时就将Fsimage和Edits文件进行了合并;

5. oiv 查看Fsimage 文件

   **基本语法: hdfs oiv -p 文件类型 -i 镜像文件 -o 转换后文件输出路径**

   如: `hdfs oiv -p XML -i fsimage_0000000000000000803 -o /opt/software/fsimage.txt`

6. oev查看Edits文件
   **基本语法: hdfs oev -p 文件类型 -i Edits文件 -o 转换后文件输出路径**
   如: `hdfs oev -p XML -i edits_inprogress_0000000000000000806 -o /opt/software/edits.xml`

##### 检查点时间设置

1. 通常情况下, `SecondaryNameNode`每隔一小时执行一次

   ```xml
   [hdfs-default.xml]
   <property>
   	<name>dfs.namenode.checkpoint.period</name>
   	<value>3600s</value>
   </property>
   ```

2. 一分钟检查一次操作次数，当操作次数达到1百万时, `SecondaryNameNode`执行一次

   ```xml
   <property>
   	<name>dfs.namenode.checkpoint.txns</name>
   	<value>100000</value>
   	<description>操作动作次数</description>
   </property>
   <property>
   	<name>dfs.namenode.checkpoint.period</name>
   	<value>60s</value>
   	<description>1分钟检查一次操作次数</description>
   </property>
   ```

   #### DataNode的工作机制

   ![image-20220919063924726](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220919063924726.png)

   1. DataNode启动后向NameNode注册，通过后，周期性(6小时)向NameNode上报所有块信息

      ```xml
      <property>
      	<name>dfs.blockreport.intervalMsec</name>
      	<value>21600000</value>
      	<description>Determines block reporting interval in milliseconds</description>
      </property>
      ```

      DataNode自己节点块信息(如检查块有没有损坏等)列表时间，默认6小时
      ```xml
      <property>
      	<name>dfs.datanode.directoryscan.interval</name>
      	<value>21600s</value>
      </property>
      ```

   ##### 数据完整性

   DataNode节点保证数据完整性的方法:

   a. 当DataNode读取Block的时,它会计算checksum

   b. 如果计算后的Checksum 与 Block创建时值不一样，说明Block已损坏

   c. Client读取其他DataNode上的Block

   d. 场景校验算法 crc(32), md5(128), sha1(160)

   e. DataNode在其文件创建后周期验证 CheckSum

   ##### 掉线时限参数设置

   <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220919070715125.png" alt="image-20220919070715125" style="zoom:67%;" />

   其中默认的`dfs.namenode.heartbeat.recheck-interval`大小为5分钟, `dfs.heartbeat.interval`默认为3秒

   需要注意的是`hdfs-site.xml`配置文件中`heartbeat.recheck.interval`的单位为毫秒，`dfs.hearbeat.interval`单位为秒。

   ```xml
   <property>
   	<name>dfs.namenode.heartbeat.recheck-interval</name>
   	<value>300000</value>
   </property>
   <property>
   	<name>dfs.namenode.heartbeat.interval</name>
   	<value>3</value>
   </property>
   ```

   
