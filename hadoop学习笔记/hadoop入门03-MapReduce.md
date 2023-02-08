## MapReduce

优点:

1. 易于编程, 用户只关心业务逻辑。实现框架的接口;
2. 良好扩展性: 可以动态增加服务器,解决计算资源不足问题;
3. 高容错性: 任何一台机器挂掉,可以将任务转移到其他节点
4. 适合海量数据计算;

缺点:

1. 不擅长实时计算;
2. 不擅长流式计算;
3. 不擅长DAG有向无环图计算;

- **MapReduce运算程序一般需要分成2阶段:Map阶段和Reduce阶段**
- **Map阶段的并发MapTask,完全并行运行,互不相干**
- Reduce阶段的并发ReduceTask,完全互不相干,但是他们的数据依赖于上衣阶段的MapTask并发实例的输出
- MapReduce编程模型只能包含一个Map阶段和一个Reduce阶段, 如果用户业务逻辑复杂,那么只能多个MapReduce程序,串行运行;

**Hadoop中常用数据序列化类型**

| Java 类型  | Hadoop Writeable类型 |
| ---------- | -------------------- |
| Boolean    | BooleanWritable      |
| Byte       | ByteWritable         |
| Int        | IntWritable          |
| Float      | FloatWritable        |
| Long       | LongWritable         |
| Double     | DoubleWritable       |
| **String** | Text                 |
| Map        | MapWritable          |
| Array      | ArrayWritable        |
| Null       | NullWritable         |

**MapReduce进程**

1. MrAppMaster: 负责整个程序的过程调度和状态协调;
2. MapTask: 负责Map阶段的整个数据处理流程;
3. ReduceTask: 负责Reduce阶段整个数据处理流程;

**Mapper阶段**

1. 用户自定义的Mapper要继承自己的父类`Mapper`;
2. Mapper的**输入数据是KV对的形式**(KV类型自定义)
3. Mapper中业务逻辑写在`map()`方法中
4. Mapper的**输出数据是KV对的形式**(KV的类型可自定义)
5. **`map()`方法(MapTask进程)对每一个`<K,V>`调用一次。比如 word count程序中，每一行对应key是行号, 该行的内容是value，进而每一行调用一次map方法**

**Reducer阶段**

1. 用户自定义的Reducer要继承自己的父类`Reducer`

2. Reducer的输入数据类型对应Mapper的输出数据类型，也是KV

3. Recuder的业务逻辑写在`reduce()`方法中

4. **`ReduceTask`进程对每一组相同k的`k,v`组调用一次`reduce()`方法**

   如

   ```shell
    原始内容:
    a  a
    b  c
    c  b
    
    Map阶段处理完成后的结果:
    (a,1) (a,1)
    (b,1) (c,1)
    (c,1) (b,1)
    
    Reducer中输入是:
    a,(1,1) 后面的(1,1)是一个迭代器,是一个集合,调用一次 reduce 方法
    b,(1,1) 调用一次 reduce 方法
    c,(1,1) 调用一次 reduce 方法
   ```

**Driver阶段**

相当于YARN集群的客户端，用于提交我们整个程序到YARN集群，提交的是封装了MapReduce程序相关运行参数的job对象。

**示例**
`pom.xml`文件:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.luke.hadoop</groupId>
  <artifactId>hdfs-client</artifactId>
  <packaging>jar</packaging>
  <version>0.0.1-snapshot</version>
  <name>hdfs-client</name>
  <url>http://maven.apache.org</url>
  <dependencies>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>3.8.1</version>
      <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>log4j</groupId>
        <artifactId>log4j</artifactId>
        <version>1.2.17</version>
    </dependency>
    <dependency>
      <groupId>org.apache.hadoop</groupId>
      <artifactId>hadoop-client</artifactId>
      <version>3.2.4</version>
    </dependency>
  </dependencies>
  <properties>
    <maven.compiler.source>1.6</maven.compiler.source>
    <maven.compiler.target>1.6</maven.compiler.target>
  </properties>
  <build>
    <plugins>
      <plugin>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.6.1</version>
        <configuration>
          <source>1.8</source>
          <target>1.8</target>
        </configuration>
      </plugin>
      <plugin>
        <artifactId>maven-assembly-plugin</artifactId>
        <configuration>
          <descriptorRefs>
            <descriptorRef>jar-with-dependencies</descriptorRef>
          </descriptorRefs>
        </configuration>
        <executions>
          <execution>
            <id>make-assembly</id>
            <phase>package</phase>
            <goals>
              <goal>single</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```

```java
/* WordCountMapper.java */
package com.luke.mapreduce.wordcount;

import java.io.IOException;

import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;

/*
 * KEYIN map阶段输入的key类型:LongWritable
 * VALUEIN map阶段输入value类型:Text
 * KeyOUT map阶段输出的Key类型:Text
 * VALUEOUT map阶段输出的value类型: IntWritable
*/
public class WordCountMapper extends Mapper<LongWritable, Text, Text, IntWritable> {
    private Text outK = new Text();
    private IntWritable outV = new IntWritable();

    // context 做上下文联络,做mapper 和 reducer之间的联络
    @Override
    protected void map(LongWritable key, Text value, Mapper<LongWritable, Text, Text, IntWritable>.Context context)
            throws IOException, InterruptedException {
        // 单行内容
        String line = value.toString();

        // 切割
        String[] words = line.split(" ");

        // 循环写出
        for (String each : words) {
            outK.set(each);
            outV.set(1);
            context.write(outK, outV);
        }
    }
}
```

```java
/* WordCountReducer.java */
package com.luke.mapreduce.wordcount;
import java.io.IOException;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Reducer;

/*
 * KEYIN reducer阶段的输入是 mapper阶段的输出,
 * VALUEIN 
 * KeyOUT reducer阶段输出的Key类型:Text
 * VALUEOUT reducer阶段输出的value类型: IntWritable
*/
public class WordCountReducer extends Reducer<Text, IntWritable, Text, IntWritable> {
    private IntWritable sum = new IntWritable();

    @Override
    protected void reduce(Text key, Iterable<IntWritable> values,
            Reducer<Text, IntWritable, Text, IntWritable>.Context context) throws IOException, InterruptedException {
        // 传入的值如 atguigu,(1,1)
        // key是atguigu, value是 (1,1)
        int tmp = 0;
        for (IntWritable value : values) {
            tmp += value.get();
        }
        sum.set(tmp);
        context.write(key, sum);
    }
}
```

```java
/* WordCountDriver.java */
package com.luke.mapreduce.wordcount;

import java.io.IOException;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

public class WordCountDriver {
    public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
        // 1. 获取job
        Configuration conf = new Configuration();
        Job job = Job.getInstance(conf);
        // 2 设置jar包路径
        job.setJarByClass(WordCountDriver.class);
        // 3 关联 mapper 和 reducer
        job.setMapperClass(WordCountMapper.class);
        job.setReducerClass(WordCountReducer.class);
        // 4. 设置map输出的kv 类型
        job.setMapOutputKeyClass(Text.class);
        job.setMapOutputValueClass(IntWritable.class);

        // 5 设置最终输出的KV类型(其实就是reducer阶段的输出类型)
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(IntWritable.class);
        // 6 设置输入路径和输出路径
        FileInputFormat.setInputPaths(job, new Path(args[0]));
        FileOutputFormat.setOutputPath(job, new Path(args[1]));

        // 7 提交job
        boolean ret = job.waitForCompletion(true);
        System.exit(ret ? 0 : 1);
    }
}
```

**最后的打包运行:**

- **打包**: `mvn package`
- 运行: `hadoop jar target/hdfs-client-0.0.1-snapshot-jar-with-dependencies.jar com.luke.mapreduce.wordcount.WordCountDriver /wcinput/words /wcoutput`

#### MapReduce序列化

**序列化**就是<mark style="color:red">将内存中的对象，转换成字节序列</mark>(或其他数据传输协议)便于存储到磁盘(持久化)和网络传输;
**反序列化**就是将收到字节序列(或其他数据传输协议) 或者 <mark style="color:red">磁盘的持久化数据，转换成内存中的对象</mark>;
序列化的目的: 便于从一台机器传输到另外一台机器;

上面有介绍常用的序列化类型，如`BooleanWritable`、`ByteWritable`等。

**自定义`bean`对象实现序列化接口**

基本的序列化类型不能满足所有需求，则需要在Hadoop框架内传递一个bean对象，该对象来实现序列化接口。

1. 必须实现`Writable`接口;

2. 反序列化时，需要反射调用空参构造函数，所以必须有空参构造

   ```java
   public FlowBean(){
   	super();
   }
   ```

3. 重写序列化方法

   ```java
   @Override
   public void write(DataOutput out) throws IOException{
   	out.WriteLong(upFlow);
   	out.WriteLong(downFlow);
   	out.WriteLong(sumFlow);
   }
   ```

4. 重写反序列化方法
   ```java
   @Override
   public void readFields(DataOutput in) throws IOException{
   	upFlow = in.readLong();
   	downFlow = in.readLong();
   	sumFlow = in.readLong();
   }
   ```

5. <mark style="color:red">注意反序列化的顺序和序列化的顺序完全一致</mark>，队列？

6. 想要把结果显示在文件中，需要重写`toString()`，可用`\t`分开，方便后续用;

7. **如果需要将自定义的`bean`放在key中传输，则还需要实现`Comparable`接口**，因为MapReduce框中的`Shuffle`过程要求对key必须能排序。
   也就是上面MapReduce的`mapper`、`reducer`函数中的`KEY`、`VALUE`

   ```java
   @Override
   public int compareTo(FlowBean o) throws IOException{
   	//倒序排列,从大到小
   	return this.sumFlow > o.getSumFlow()? -1 : 1;
   }
   ```

**序列化案例实操:**

背景: 统计每隔一手机号耗费的总上行流量、总下行流量、总流量。

1. 输入数据: phone_data.txt

2. 输入数据格式:

   ```tex
   id	手机号码	网络ip	上行流量	下行流量	网络状态码
   7	13566667777	120.196.100.99	1116	954	200
   ```

3. 期望输出数据格式:(总流量是 上行流量 + 下行流量)

   ```java
   手机号码	上行流量	下行流量	总流量
   13566667777	1116	954	2070
   ```

4. 示例输入数据`phone_data.txt`

   ```text
   1       13736230513     192.196.100.1   www.atguigu.com 2481    24681   200
   2       13846544121     192.196.100.2           264     0       200
   3       13956435636     192.196.100.3           132     1512    200
   4       13966251146     192.168.100.1           240     0       404
   5       18271575951     192.168.100.2   www.atguigu.com 1527    2106    200
   6       84188413        192.168.100.3   www.atguigu.com 4116    1432    200
   7       13590439668     192.168.100.4           1116    954     200
   8       15910133277     192.168.100.5   www.hao123.com  3156    2936    200
   9       13729199489     192.168.100.6           240     0       200
   10      13630577991     192.168.100.7   www.shouhu.com  6960    690     200
   11      15043685818     192.168.100.8   www.baidu.com   3659    3538    200
   12      15959002129     192.168.100.9   www.atguigu.com 1938    180     500
   13      13560439638     192.168.100.10          918     4938    200
   14      13470253144     192.168.100.11          180     180     200
   15      13682846555     192.168.100.12  www.qq.com      1938    2910    200
   16      13992314666     192.168.100.13  www.gaga.com    3008    3720    200
   17      13509468723     192.168.100.14  www.qinghua.com 7335    110349  404
   18      18390173782     192.168.100.15  www.sogou.com   9531    2412    200
   19      13975057813     192.168.100.16  www.baidu.com   11058   48243   200
   20      13768778790     192.168.100.17          120     120     200
   21      13568436656     192.168.100.18  www.alibaba.com 2481    24681   200
   22      13568436656     192.168.100.19         1116    954     200
   ```

   如果有相同的手机号如`13568436656`, 则是 将所有行的 上行流量 加一起等于上行流量，所有下行流量加一起等于下行流量，而后是总流量。

   ```java
   /* FlowBean.java */
   package com.luke.mapreduce.writable;
   import java.io.DataInput;
   import java.io.DataOutput;
   import java.io.IOException;
   import org.apache.hadoop.io.Writable;
   /*
    * 1. 定义类实现 writable 接口
    * 2. 重写序列化和反序列化方法
    * 3. 重写空参构造
    * 4. toString()方法
   */
   public class FlowBean implements Writable {
       private long upFlow; // 上行流量
       private long downFlow; // 下行流量
       private long sumFlow; // 总流量
   
       public long getUpFlow() {
           return upFlow;
       }
       public long getDownFlow() {
           return downFlow;
       }
       public long getSumFlow() {
           return sumFlow;
       }
       public void setUpFlow(long upFlow) {
           this.upFlow = upFlow;
       }
       public void setDownFlow(long downFlow) {
           this.downFlow = downFlow;
       }
       public void setSumFlow() {
           this.sumFlow = this.upFlow + this.downFlow;
       }
       public FlowBean() {}
       @Override
       public void write(DataOutput out) throws IOException {
           out.writeLong(upFlow);
           out.writeLong(downFlow);
           out.writeLong(sumFlow);
       }
       @Override
       public void readFields(DataInput in) throws IOException {
           this.upFlow = in.readLong();
           this.downFlow = in.readLong();
           this.sumFlow = in.readLong();
       }
       @Override
       public String toString() {
           return this.upFlow + "\t" + this.downFlow + "\t" + this.sumFlow;
       }
   }
   ```
   
   ```java
   /* FlowMapper.java */
   package com.luke.mapreduce.writable;
   
   import java.io.IOException;
   import org.apache.hadoop.io.LongWritable;
   import org.apache.hadoop.io.Text;
   import org.apache.hadoop.mapreduce.Mapper;
   
   public class FlowMapper extends Mapper<LongWritable, Text, Text, FlowBean> {
       private Text outK = new Text();
       private FlowBean outV = new FlowBean();
   
       @Override
       protected void map(LongWritable key, Text value, Mapper<LongWritable, Text, Text, FlowBean>.Context context)
               throws IOException, InterruptedException {
           // 1 获取一行
           // 1 13736230513 192.196.100.1 www.atguigu.com 2481 24681 200
           String line = value.toString();
           String[] list = line.split("\\s+");
         
           String phoneNum = list[1];
           String upFlow = list[list.length - 3];
           String downFlow = list[list.length - 2];
   
           outK.set(phoneNum);
           outV.setUpFlow(Long.parseLong(upFlow));
           outV.setDownFlow(Long.parseLong(downFlow));
           outV.setSumFlow();
           context.write(outK, outV);
       }
   }
   ```
   
   ```java
   /* FlowReducer.java */
   package com.luke.mapreduce.writable;
   
   import java.io.IOException;
   import org.apache.hadoop.io.Text;
   import org.apache.hadoop.mapreduce.Reducer;
   
   public class FlowReducer extends Reducer<Text, FlowBean, Text, FlowBean> {
       private FlowBean outV = new FlowBean();
       private long sumUp;
       private long sumDown;
       @Override
       protected void reduce(Text key, Iterable<FlowBean> values, Reducer<Text, FlowBean, Text, FlowBean>.Context context)
               throws IOException, InterruptedException {
           sumUp = 0;
           sumDown = 0;
           for (FlowBean value : values) {
               sumUp += value.getUpFlow();
               sumDown += value.getDownFlow();
           }
           outV.setUpFlow(sumUp);
           outV.setDownFlow(sumDown);
           outV.setSumFlow();
           context.write(key, outV);
       }
   }
   ```
   
   ```java
   /* FlowDriver.java */
   package com.luke.mapreduce.writable;
   
   import java.io.IOException;
   
   import org.apache.hadoop.conf.Configuration;
   import org.apache.hadoop.fs.Path;
   import org.apache.hadoop.io.Text;
   import org.apache.hadoop.mapreduce.Job;
   import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
   import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
   
   public class FlowDriver {
       public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
           // 1. 获取job
           Configuration conf = new Configuration();
           Job job = Job.getInstance(conf);
           // 2 设置jar包路径
           job.setJarByClass(FlowDriver.class);
           // 3 关联 mapper 和 reducer
           job.setMapperClass(FlowMapper.class);
           job.setReducerClass(FlowReducer.class);
           // 4. 设置map输出的kv 类型
           job.setMapOutputKeyClass(Text.class);
           job.setMapOutputValueClass(FlowBean.class);
   
           // 5 设置最终输出的KV类型(其实就是reducer阶段的输出类型)
           job.setOutputKeyClass(Text.class);
           job.setOutputValueClass(FlowBean.class);
           // 6 设置输入路径和输出路径
           FileInputFormat.setInputPaths(job, new Path("/root/code/java/hdfs-client/phone_data.txt"));
           FileOutputFormat.setOutputPath(job, new Path("/root/code/java/hdfs-client/retdir"));
   
           // 7 提交job
           boolean ret = job.waitForCompletion(true);
           System.exit(ret ? 0 : 1);
       }
   }
   ```
   
   

