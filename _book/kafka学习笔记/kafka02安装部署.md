### Kafka 安装部署

- hadoop100  zk + kafka
- hadoop101  zk + kafka
- hadoop102  zk + kafka

##### 下载介质

[https://kafka.apache.org/downloads](https://kafka.apache.org/downloads)

##### zookeeper安装

- 下载地址: [https://archive.apache.org/dist/zookeeper/](https://archive.apache.org/dist/zookeeper/)

  [https://archive.apache.org/dist/zookeeper/zookeeper-3.5.10/](https://archive.apache.org/dist/zookeeper/zookeeper-3.5.10/)

  需要下载`-bin`版本的，比如: `apache-zookeeper-3.5.10-bin.tar.gz`

- 解压并重命名:

  ```shell
  wget https://archive.apache.org/dist/zookeeper/zookeeper-3.5.10/apache-zookeeper-3.5.10-bin.tar.gz
  tar -zxf apache-zookeeper-3.5.10-bin.tar.gz
  mv apache-zookeeper-3.5.10-bin zookeeper-3.5.10-bin
  ```

- 配置`/usr/local/zookeeper-3.5.10-bin/conf/zoo.cfg`:

  ```properties
  tickTime=2000 # 通信心跳时间,2000ms
  initLimit=10 # Leader 和 Flower之间通信时限,初始化通信不能超过10 * tickTime =2s
  syncLimit=5 # Leader和Follower之间通信如果超过 syncLimit*tickTIme,则leader认为Follower死亡,从服务器中剔除Follower
  dataDir=/data/zookeeper # 保存Zookeeper中的数据
  clientPort=2181 #客户端连接端口号
  ```

- 在`dataDir=/data/zookeeper`目录下装机一个`myid`的文件，在文件中添加与server对应的编号

  - `hadoop100`: myid内容0;
  - `hadoop101`: myid内容1;
  - `hadoop102`: myid内容2;

- 配置文件`/usr/local/zookeeper-3.5.10-bin/conf/zoo.cfg`中添加如下配置:

  ```properties
  ######cluster#####
  server.0=hadoop100:2888:3888
  server.1=hadoop101:2888:3888
  server.2=hadoop102:2888:3888
  ```

  配置参数解读:`server.A=B:C:D`

  - A是一个数字，表示这个是第几号服务器，和该服务器中`$dataDir/myid`内容对应。**Zookeeper启动时读取此文件，拿到里面的数据与zoo.cfg.里面的配置信比较。进而判断到底是哪个server。**

  - B 这个服务器的地址;
  - C  这个服务器Follower与集群中Leader服务器交换信息的端口;
  - D 万一集群中的Leader服务器挂了，需要一个端口来重新选举。选出新的Leader，这个端口号就是用来执行选举时服务器之间相互通信;

- 分发最新的配置文件:

  ```shell
  $ rsync -av /usr/local/zookeeper-3.5.10-bin/conf/zoo.cfg  root@hadoop100:/usr/local/zookeeper-3.5.10-bin/conf/zoo.cfg
  
  $ rsync -av /usr/local/zookeeper-3.5.10-bin/conf/zoo.cfg  root@hadoop102:/usr/local/zookeeper-3.5.10-bin/conf/zoo.cfg
  ```

- 启动集群:

  ```shell
  /* hadoop101 机器 */
  $ ./bin/zkServer.sh start
  ZooKeeper JMX enabled by default
  Using config: /usr/local/zookeeper-3.5.10-bin/bin/../conf/zoo.cfg
  Starting zookeeper ... STARTED
  
  /* hadoop102机器 */
  $ ./bin/zkServer.sh start
  ZooKeeper JMX enabled by default
  Using config: /usr/local/zookeeper-3.5.10-bin/bin/../conf/zoo.cfg
  Starting zookeeper ... STARTED
  
  /* hadoop100机器 */
  $ ./bin/zkServer.sh start
  ZooKeeper JMX enabled by default
  Using config: /usr/local/zookeeper-3.5.10-bin/bin/../conf/zoo.cfg
  Starting zookeeper ... STARTED
  
  /* hadoop101 机器 */
  ./bin/zkServer.sh status
  ZooKeeper JMX enabled by default
  Using config: /usr/local/zookeeper-3.5.10-bin/bin/../conf/zoo.cfg
  Client port found: 2181. Client address: localhost. Client SSL: false.
  Mode: leader
  ```

- `zookeeper`集群启动，停止脚本`zk.sh`

  ```shell
  #!/bin/bash
  
  case $1 in
  "start") {
      for i in hadoop100 hadoop101 hadoop102
      do
          echo "--------- zookeeper $i start ---------"
          ssh root@$i "source /etc/profile && cd /usr/local/zookeeper-3.5.10-bin && ./bin/zkServer.sh start"
      done
  }
  ;;
  "stop") {
      for i in hadoop100 hadoop101 hadoop102
      do
          echo "--------- zookeeper $i stop ---------"
          ssh root@$i "source /etc/profile &&  cd /usr/local/zookeeper-3.5.10-bin && ./bin/zkServer.sh stop"
      done
  }
  ;;
  "status") {
      for i in hadoop100 hadoop101 hadoop102
      do
          echo "--------- zookeeper $i status ---------"
          ssh root@$i "source /etc/profile && cd /usr/local/zookeeper-3.5.10-bin && ./bin/zkServer.sh status"
      done
  }
  ;;
  esac
  ```

##### 配置集群

```properties
# hadoop100, 文件 config/server.properties
broker.id=0
log.dirs=/data/kafka/log
zookeeper.connect=hadoop100:2181,hadoop101:2181,hadoop102:2181/kafka

#  hadoop101, 文件 config/server.properties
broker.id=1
log.dirs=/data/kafka/log
zookeeper.connect=hadoop100:2181,hadoop101:2181,hadoop102:2181/kafka

#  hadoop102, 文件 config/server.properties
broker.id=2
log.dirs=/data/kafka/log
zookeeper.connect=hadoop100:2181,hadoop101:2181,hadoop102:2181/kafka
```

配置环境变量:

```shell
export KAFKA_HOME=/usr/local/kafka_2.12-3.2.3
export PATH=$KAFKA_HOME/bin:$PATH
```

启动:

```shell
# hadoop100 
cd /usr/local/kafka_2.12-3.2.3 && ./bin/kafka-server-start.sh -daemon config/server.properties

# hadoop101
cd /usr/local/kafka_2.12-3.2.3 && ./bin/kafka-server-start.sh -daemon config/server.properties

# hadoop102
cd /usr/local/kafka_2.12-3.2.3 && ./bin/kafka-server-start.sh -daemon config/server.properties
```

错误信息:
```java
Error: VM option 'UseG1GC' is experimental and must be enabled via -XX:+UnlockExperimentalVMOptions.
Error: Could not create the Java Virtual Machine.
Error: A fatal exception has occurred. Program will exit.
```

修复,文件`bin/kafka-run-class.sh`:
<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221030143141487.png" alt="image-20221030143141487" style="zoom:50%;" />

**kafka的启停脚本:**

```shell
#!/bin/bash

case $1 in
"start") {
    for i in hadoop100 hadoop101 hadoop102
    do
        echo "--------- kafka $i start ---------"
        ssh root@$i "source /etc/profile && cd /usr/local/kafka_2.12-3.2.3 && ./bin/kafka-server-start.sh -daemon config/server.properties"
    done
}
;;
"stop") {
    for i in hadoop100 hadoop101 hadoop102
    do
        echo "--------- kafka $i stop ---------"
        ssh root@$i "source /etc/profile &&  cd /usr/local/kafka_2.12-3.2.3 && ./bin/kafka-server-stop.sh"
    done
}
;;
esac
```

**注意: 先关kafka，再关ZK。如果先关ZK，后续kafka则无法停止，因为会不断重连ZK，只能通过 `kill -9`关闭。**

