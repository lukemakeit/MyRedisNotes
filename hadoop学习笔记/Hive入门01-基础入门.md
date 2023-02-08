### Hive

Hive是基于Hadoop的一个数据仓库工具，可以将 **结构化的数据文件映射成一张表**，并提供 **类SQL** 查询功能。
结构化体现为如分隔符一样，分隔出来的列一样。
Hive: 将SQL转换成MapReduce程序。我们就不再需要做前面的mapTask、reduceTask 那些事情了。
MapReduce能解决的，Hive不一定能解决；Hive能解决的，MapReduce不一定能解决。

- Hive处理的数据存储在HDFS中
- Hive分析数据底层的实现是MapReduce
- 执行程序运行在yarn上

#### **Hive优缺点**

- **优点**
  - 操作接口采用类SQL语法，提供快速开发的能力 (简单、容易上手)
  - 避免去写MapReduce，减少开发人员的学习成本
  - **Hive的执行延迟比较高，因此Hive长用于数据分析，对实时性不高的场合**
  - Hive优势在于处理大数据，对于处理小数据没优势，因为Hive的执行延迟比较高
  - Hive支持用户自定义函数，用户可根据自己的需求来实现自己的函数
- **缺点**
  - **HQL表达能力有限**
    - 迭代式算法无法表达，即第一个MR的结果作为第二个MR的输入;
    - 数据挖掘方面不擅长，由于MapReduce数据处理流程的限制，效率更高的算法却无法实现
  - **Hive效率较低**
    - Hive自动生成的MapReduce作业，通常情况下不够智能化
    - Hive调优比较困难，粒度较粗

#### Hive架构原理

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221018002801250.png" alt="image-20221018002801250" style="zoom:50%;" />

- Meta store: 元数据存储，即Hive中的表和HDFS中的路径做映射。可以用MySQL存;

**和传统数据库的比较**

- 查询语言
- 数据更新: 数据仓库的内容是读多写少，**因此 Hive中不建议对数据的改写，所有数据都是在加载的时候定义好的**。而数据库中数据通常需要经常修改的，因此可以使用`INSERT INTO ... VALUES `添加数据，使用`UPDATE ... SET`修改数据。update的流程一般是先把数据下载下来，覆盖，再写回去，成本非常高；
- 执行延迟: 没有索引，需扫描全部数据。
- 数据规模

#### Hive安装

- 官网地址: [https://hive.apache.org/](https://hive.apache.org/)
- 文档地址: [https://cwiki.apache.org/confluence/display/Hive//GettingStarted](https://cwiki.apache.org/confluence/display/Hive//GettingStarted)
- 下载地址: [https://www.apache.org/dyn/closer.cgi/hive/](https://www.apache.org/dyn/closer.cgi/hive/)

```shell
cd /usr/local && tar -zxf apache-hive-3.1.3-bin.tar.gz
ln -s apache-hive-3.1.3-bin hive

echo "export HIVE_HOME=/usr/local/hive" >>/etc/profile
echo "export PATH=\$HIVE_HOME/bin:\$PATH" >>/etc/profile
```

解决冲突日志包冲突:

```shell
cd /usr/loca/hive
mv lib/log4j-slf4j-impl-2.17.1.jar lib/log4j-slf4j-impl-2.17.1.bak
```

**初始化源数据库:**

```shell
$ bin/schematool -dbType derby -initSchema
Metastore connection URL:        jdbc:derby:;databaseName=metastore_db;create=true
Metastore Connection Driver :    org.apache.derby.jdbc.EmbeddedDriver
Metastore connection User:       APP
Starting metastore schema initialization to 3.1.0
Initialization script hive-schema-3.1.0.derby.sql
Initialization script completed
schemaTool completed
```

如果报`Exception in thread "main" java.lang.NoSuchMethodError: com.google.common.base.Preconditions.checkArgument`错误，则是因为 **hive内依赖的guava.jar和hadoop内的版本不一致造成的**。咱们将hadoop中的高版本替换掉hive中的低版本即可

```shell
$ ls -lrht lib/guava-19.0.jar
$ ls -lrht ../hadoop-3.2.4/share/hadoop/common/lib/guava-27.0-jre.jar
$ mv lib/guava-19.0.jar lib/guava-19.0.bak
$ cp ../hadoop-3.2.4/share/hadoop/common/lib/guava-27.0-jre.jar lib/
```

- HDFS  webUI端口: [http://hadoop101:9870/](http://hadoop101:9870/), 服务端口:`hadoop101:8020` 
- Yarn webUI端口: [http://hadoop100:8088/cluster](http://hadoop100:8088/cluster)，服务端口号: `hadoop100:8032`
- 历史服务 webUI端口: [http://hadoop100:19888/jobhistory/app，服务端口号: `hadoop:10020`](http://hadoop100:19888/jobhistory/app)

**启动hive(derby数据库)**

```shell
$ cd /usr/local/hive
$ bin/hive
> 
```

**注意:**
- **hive默认的日志位置: `/tmp/${user}/hive.log`，我前面用的用户是root，所以位置是`/tmp/root/hive.log`**
- **使用默认保存数据的`derby`是无法启动多个 `bin/hive`的。只能单用户用;**

**执行命令:**

```sql
hive> show databases;
hive> create table test (id int,name string);
hive> show tables;
hive> insert into test values(1,'a');
hive> select * from test;
OK
1       a
```

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221021000713776.png" alt="image-20221021000713776" style="zoom:67%;" />

执行上面命令后，数据保存在 HDFS中的`/user`目录下。

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221021000842732.png" alt="image-20221021000842732" style="zoom: 33%;" />

#### 启动Hive(MySQL数据库)

- **第一步: MySQL安装和启动;**

- **第二步: 拷贝`cp mysql-connector-java-5.1.49.jar /usr/local/hive/lib`**

- **第三步: 生成配置文件,`/usr/local/hive/conf/hive-site.xml`**

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
  <configuration>
     <property>
       <!--mysql的地址以及对应的数据库-->
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:mysql://10.211.55.7:20000/db_hive?createDatabaseIfNotExist=true</value>
     </property>
     <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>com.mysql.jdbc.Driver</value>
     </property>
     <property>
          <!--连接MySQL的用户名-->
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>hive</value>
     </property>
     <property>
       <!--连接MySQL的密码-->
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>xxxxx</value>
     </property>
     <property>
          <!--元数据存储版本的验证-->
          <name>hive.metastore.schema.verification</name>
          <value>false</value>
     </property>
     <property>
          <!--元数据存储授权-->
          <name>hive.metastore.event.db.notification.api.auth</name>
          <value>false</value>
     </property>
     <property>
        <name>hive.exec.scratchdir</name>
        <value>/user/hive/tmp</value>
     </property>
     <property>
          <!--Hive默认在HDFS的工作目录-->
        <name>hive.metastore.warehouse.dir</name>
        <value>/user/hive/warehouse</value>
     </property>
     <property>
        <name>hive.querylog.location</name>
        <value>/user/hive/log</value>
     </property>
  </configuration>
  ```

- **第四步:初始化MySQL数据库**

  ```shell
  $ cd /usr/local/hive && bin/schematool -initSchema -dbType mysql -verbose
  ```

- **第五步: 启动Hive**

  ```shell
  $ cd /usr/local/hive && bin/hive
  hive> show databases;
  hive> create table test (id int,name string);
  hive> show tables;
  hive> insert into test values(1,'a');
  hive> select * from test;
  OK
  1       a
  ```

  现在我们新建一个std.txt文件 并上传到HDFS的`/user/hive/warehouse/test`目录下:

  (下面是sublime打开的情况)

  <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221021005758086.png" alt="image-20221021005758086" style="zoom:33%;" />

  ```shell
  $ hadoop fs -put std.txt /user/hive/warehouse/test
  
  # hive 再次查询
  hive> select * from test;
  OK
  1       a
  2       b
  3       c
  4       d
  5       e
  11      aa
  Time taken: 0.12 seconds, Fetched: 6 row(s)
  ```

  **查看MySQL中的一些元数据:**

  ```sql
  mysql> select * from DBS\G
  *************************** 1. row ***************************
            DB_ID: 1
             DESC: Default Hive database
  DB_LOCATION_URI: hdfs://hadoop101:8020/user/hive/warehouse
             NAME: default
       OWNER_NAME: public
       OWNER_TYPE: ROLE
        CTLG_NAME: hive
  1 row in set (0.01 sec)
  
  mysql> select * from TBLS\G
  *************************** 1. row ***************************
              TBL_ID: 1
         CREATE_TIME: 1666284411
               DB_ID: 1
    LAST_ACCESS_TIME: 0
               OWNER: root
          OWNER_TYPE: USER
           RETENTION: 0
               SD_ID: 1
            TBL_NAME: test
            TBL_TYPE: MANAGED_TABLE
  VIEW_EXPANDED_TEXT: NULL
  VIEW_ORIGINAL_TEXT: NULL
  IS_REWRITE_ENABLED: 0x00
  1 row in set (0.00 sec)
  ```

  `DB_LOCATION_URI: hdfs://hadoop101:8020/user/hive/warehouse` 和`TBL_NAME: test`组合起来就是对应的 `select * from test`需要查询的目录啦
