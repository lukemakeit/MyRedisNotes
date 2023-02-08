## MapReduce 框架原理

**总结**

1. InputFormat
   - 默认的是TextInputFormat, kv, key是偏移量,value是一行的内容;
   - 处理小文件`CombineTextInputFormat`把多个文件合并到一个一起处理;
2. Mapper:
   - `setup()`:初始化;
   - `map()`:用户的业务逻辑;
   - `clearup()`: 关闭资源;
3. 分区(`partition`)
   - 默认分区 hashPartitioner, 默认按照key的hash值%numreducetask 个数;
   - 自定义分区;
4. 排序(缓冲区溢写前就是需要排序)
   - 部分排序: 每个输出的文件内部有序;
   - 全排序: 一个reduceTask, 对所有数据排序(慎用，可能oom)；
   - 自定义排序: 实现WritableCompare接口，重写`compareTo`方法;
5. Combiner:
   - 前提: 不影响最终业务逻辑(求和没问题,请平均值不行);
   - map阶段提前预聚合, 解决数据倾斜的方法;
6. Reducer
   - `setup()`:初始化;
   - `reduce()`:用户的业务逻辑;
   - `clearup()`: 关闭资源;
7. OutputFormat:
   - 默认`TextOutputFormat`: 按行输出到文件;
   - 自定义输出;

### InputFormat 数据输入

#### 切片与MapTask 并行度决定机制

**问题引出**
MapTask的并行度决定Map阶段的任务处理并发度，进而影响整个Job的处理速度。
<mark style="color:red">思考: 1G的数据，启动8个MapTask，可以提高集群的并发处理能力。那么1K的数据，也启动8个MapTask，会提供集群性能吗？MapTask并行任务是否越多越好呢？哪些因素影响了MapTask并行度?</mark>

**并行度决定机制**

- 数据块: Block是HDFS物理上把数据分成一块一块。<mark style="color:red">数据块是HDFS存储数据的单元</mark>

- 数据切片: 数据切片只是在逻辑上对输入进行分片，并不会在磁盘上将其切分进行存储。<mark style="color:red">数据切片是MapReduce程序计算输入数据的单位</mark>，一个切片会对应启动一个MapTask。

  ![image-20220926064444801](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220926064444801.png)

  - <mark style="color:red">一个Job的Map阶段并行度有客户端在提交Job时的切片数决定的</mark>
  - **每一个Split切片分配一个MapTask并行实例处理**
  - **默认情况下，切片大小=BlockSize**
  - **切片时不考虑数据集整体，而是逐个针对每一个文件单独切片。比如有1.sql 200MB和 2.sql 56MB，并不会考虑将两者合并起来放在两个数据块中**

**FileInputFormat切片源码解析**

1. 程序先找到你数据存储的目录

2. 开始遍历处理(规划切片)目录中每一个文件

3. 遍历第一个文件ss.txt

   a. 计算文件大小`fs.sizeOf(ss.txt)`
   b. 计算切片大小: `computeSplitSize(Math.max(minSize,Math.min(maxSize,blocksize)))=blocksize=128M`

   - `mapreduce.input.fileinputformat.split.minsize=1` 默认值1。<mark style="color:red">`minsize(切片最大值)`: 参数调得比blockSize大，则可以让切片变得比blocksize还大</mark>
   - `mapreduce.input.fileinputformat.split.maxsize=Long.MAXVlaue` 。<mark style="color:red">`maxsize(切片最大值)`:参数如果调得比`blockSize`下，则会让切片变小，而且就等于配置的`maxSize`</mark>
   - blockSize 一般是由磁盘的传输速度决定，前面有提到；
   
   c. <mark style="color:red">默认情况下, 切片大小等于=blocksize。这样就不用跨节点传输数据了</mark>
   d. 开始切,形成第1个切片: `ss.txt——0:128M `, 第2个切片`ss.txt——128M:356M`, 第3个切片`ss.txt——256M:300M`
   (<mark style="color:red">每次切片时,都要判断切完城下的部分是否大于块的1.1倍,不大于1.1倍就划分一块切片即可</mark>)
   e. 将切片信息写到一个切片规划文件中;
   f. 整个切片的核心过程就在`getSplit()`方法中完成;
   g. <mark style="color:red">`InputSplit`只记录了切片的元数据信息，比如起始位置、长度以及所在节点列表等</mark>
   
4. 提交切片规划文件到YARN上，YARN上的`MrAppMaster`就可以根据切片规划文件开始计算`MapTask`个数;

5. 获取切片信息API

   ```java
   //获取切片的文件名称
   String name = inputSplit.getPath().getName();
   //根据文件烈性获取切片信息
   FileSplit inputSplit=(FileSplit) context.getInputSplit();
   ```

**TextInputFormat**

- FileInputFormat实现类
  思考: <mark style="color:red">在运行MapReduce程序时,输入的文件格式包括: 基于行的日志文件、二进制格式文件、数据库表等</mark>。针对不同的数据类型，MapReduce是如何读取这些数据的?
  FileInputFormat常见的接口实现类包括: `TextInputFormat`、`KeyValueTextInputFormat`、`NLineInputFormat`、`CombineTextInputFormat`和自定义`InputFormat`;

- **TextInputFormat**

  TextInputFormat 是默认的`FileInputFormat`实现类。按行读取每条记录。key是该行在整个文件中起始字节的偏移量，LongWritable类型。值是该行的内容，不包括任何行终止符(换行或回车)，Text类型;

**CombineTextInputFormat切片机制**
框架默认的`TextInputFormat`切片机制是对任务按文件规划切片，**不管文件大小，都会是一个单独的切片**，都会交给一个`MapTask`，这样<mark style="color:red">如果有大量小文件，就会产生大量MapTask，处理效率地下</mark>

**应用场景:**
CombineTextInputFormat用于小文件过多场景，可以将多个小文件从逻辑上规划到一个切片中，这样多个小文件就可以交给一个`MapTask`处理。
**虚拟存储切片最大值设置**
`CombineTextInputFormat.setMaxinputSplitSize(job,4194304)`;// 4m
**切片机制**
生成切片过程包括: **虚拟存储过程和切片过程两部分**
![image-20220926224426148](/Users/lukexwang/Library/Application Support/typora-user-images/image-20220926224426148.png)

如何用呢？上一章的`WordCount`程序增加如下代码:
```java
// 如果不设置 InputFormat, 它默认用的是 TextInputFormat.class
job.setInputFormatClass(CombineTextInputFormat.class);

//虚拟存储切片最大值设置4m
CombineTextInputFormat.setMaxInputSplitSize(job,20971520);// 20m
```

### MapReduce 工作流程

#### Shuffle机制

Map方法之后，Reduce方法之前的数据处理过程称之为Shuffle机制。

<mark style="color:red">Shuffle最主要的工作是 找到相同的key，然后将它的value集合到一起</mark>

- 环形缓冲区: 默认 100M，到80%后，开始逆写，也就是从最高地址(100%的位置)开始倒着写，再次写到80%时，如果 溢写已经完成，则继续覆盖。否则等待;
- 溢写: 就是将缓冲区中数据写到磁盘。溢写一般是环形缓冲区 使用到80%后开始进行，溢写前还会将环形缓冲区中的数据进行排序；
  - 排序是对 Key 索引进行的，是按照字典排序，使用快速排序方法;
  - 溢写会产生多个文件:索引文件`spill.index`、数据文件`splill.out`;
  - Combiner是可选流程，比如`(a,1) (a,1)`合并成`(a,2)`;
- 分区: 不同分区是进入不同ReduceTask中;
- 多次溢写的文件，需要做归并排序；
- Combiner后可配置压缩，减少网络传输开销;
- 最后是写入到磁盘，等待 ReducerTask来拉取;
- ReducerTask 拉取先放在内存，如果内存不够，放磁盘。ReducerTask拉取玩文件后，还要做一次大的归并排序：按照相同的Key分组；

![image-20220926232348260](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220926232348260.png)

#### Shuffle中的Partition分区

**问题引出**
要求统计结果**按照条件输出到不同文件中(分区)**。比如: 将统计结果按照手机归属地不同省份输出到不同文件中。
**默认Partioner分区**

```java
public class HashPartitioner<K,V> extends Partitioner<K,V>{
	public int getPartition(K key,V value, int numReduceTasks) {
		return (key.hashCode() & Inter.MAX_VALUE)%numReduceTasks;
	}
}
```

默认分区是根据Key的hashCode对ReduceTasks个数取余得到的。用户无法控制哪个key存储到哪个分区。
以前的结果总是输出到一个文件，通过`job.setNumReduceTasks(2)`就有会将结果输出到两个文件。

**自定义Partition步骤**

1. 自定义类继承Partitioner，重写`getPartition()`方法

   ```java
   public class CustomPartitioner extends Partitioner<Text,FlowBean>{
   	@Override
   	public int getPartition(Text key,FlowBean Value,int numPartitions) {
   		//控制分区代码逻辑
   		....
   		return partition;
   	}
   }
   ```

2. 在Job驱动中，设置自定义Partitioner
   `job.setPartitionerClass(CustomPartitioner)`

3. **自定义Partition后，要根据自定义Partitioner的逻辑设置相应数量的ReduceTask**
   `job.setNumReduceTasks(5)`

**分区总结**

1. <mark style="color:red">如果ReduceTask的数量 > getPartition的结果数,则会产生几个空的输出文件`part-r-00xx`;</mark>
2. <mark style="color:red">如果 `1<ReduceTask的数量<getPartition的结果数`, 则有一部分分区数据无处安放，会`Exeption`;</mark>
3. <mark style="color:red">如果ReduceTask数量为1,则不管MapTask端输出多少个分区文件,最终结果都交给这一个ReduceTask,最终也就只会产生一个文件`part-r-00000`;</mark>
4. <mark style="color:red">分区号必须从零开始，逐一累加;</mark>

**案例:**

```java
/* ProvincePartitioner.java */
package com.luke.mapreduce.writable;

import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Partitioner;

// 从Mapper出来 -> 经过Partitioner -> 进入Reducer,
// 所以Parritioner的输入和Mapper的输出一致
public class ProvincePartitioner extends Partitioner<Text, FlowBean> {
    @Override
    public int getPartition(Text key, FlowBean value, int numPartitions) {
        // key是手机号,转换为string
        String phone = key.toString();
        String prePhone = phone.substring(0, 3);
        int partition;
        if ("136".equals(prePhone)) {
            partition = 0;
        } else if ("137".equals(prePhone)) {
            partition = 1;
        } else if ("138".equals(prePhone)) {
            partition = 2;
        } else if ("139".equals(prePhone)) {
            partition = 3;
        } else {
            partition = 4;
        }
        return partition;
    }
}

/* FlowDriver.java 添加两行 */
job.setPartitionerClass(ProvincePartitioner.class);
job.setNumReduceTasks(5);
```

#### Shuffle中的排序

MapTask和ReduceTask均会对数据**按照Key**进行排序。该操作属于Hadoop的默认行为。**<mark style="color:red">任何应用程序中的数据均会被排序，不管逻辑上是否需要(为什么呢? 因为不排序,则Reduce阶段将相同Key做同一组输出是无法进行的)</mark>**
默认排序是按照**字典顺序排序**, 且实现该排序的方法是**快速排序**。

对于MapTask，它会将处理的结果暂时放在环形缓冲区中，**当环形缓冲区使用率到达一定阈值后，在对缓冲区中数据进行一次快速排序**，并将这些数据溢写到磁盘上，而当数据都写到磁盘后，他会对**磁盘上所有文件进行一次归并排序**;

对于ReduceTask，它从每个MapTask上远程拉取相应的数据文件,如果文件大小超过一定间值,则溢写磁盘上,否则存储在内存中。
**如果磁盘上文件数目达到一定國值,则进行一次归并排序以生成一个更大文件**;
**如果内存中文件大小或者数目超过一定國值,则进行一次合并后将数据溢写到磁盘上**。
**<mark style="color:red">当所有数据拷贝完毕后，ReduceTask统一对内存和磁盘上的所有数据进行一次归并排序</mark>**

1. 部分排序
   MapReduce根据输入记录的key对数据集排序。保证**输出到磁盘的每个文件内部有序。生产环境中一般做分区内有序，比如东北地区、华北地区，某省、某市 有序；**
2. 全排序
   **最终输出结果只有一个文件，且文件内部有序。**实现方式是**只设置一个ReduceTask**。但该方法在处理大型文件时效率极低，因为一台机器处理所有文件，完全丧失了MapRedue提供的并行架构。比如统计wordCoun，只有一个输出(一个分区)，分区内有序，生产环境中需慎用这种情况;
3. 辅助排序: (GroupingComparator分组) 生产环境中使用较少
   在Reduce端对key进行分组。应用于: 在接收的key为bean对象时，想让一个或几个字段相同(全部字段比较不相同)的key进入到同一个reduce方法时，可以采用分组排序。
4. 二次排序
   在二次排序(自定义排序)过程中，如果compareTo中的判断条件为两个即为二次排序。

**假如要对自定义的Bean对象做key传输,需要实现 <mark style="color:red">WritableComparable</mark>接口重写 compareTo方法，就可以实现排序**

```java
@Override
public int compareTo(FlowBean bean) {
	int result;
	// 按照总流量大小,倒序排序
	if (this.sumFlow > bean.getSumFlow()){
		result =-1;
	}else if (this.sumFlow < bean.getSumFlow()){
		result =1;
	}else{
		result=0;
	}
	return result;
}
```

**案例: WritableComparable排序案例实操(全排序)**
**上面案例中输出了 每个手机的总流量，现在要根据结果再次对总流量进行倒序排序**
结果示例:

```java
13630577991     6960    690     7650
13682846555     1938    2910    4848
13729199489     240     0       240
...
```

要实现目的，需要分两步: 

- 第一求出每个手机号的总流量(就是上面自定义FlowBean案例得到的结果)

- 第二用FlowBean做key，倒序排序

  - 输入数据:

    ```java
    13729199489     240     0       240
    13630577991     6960    690     7650
    13682846555     1938    2910    4848
    ...
    ```

  - 输出数据:

    ```java
    13630577991     6960    690     7650
    13682846555     1938    2910    4848
    13729199489     240     0       240
    ...
    ```

  - FlowBean需实现`WritableComparable`接口重写`compareTo`方法

    ```java
    public class FlowBean implements WritableComparable<FlowBean> {
      	...
        @Override
        public int compareTo(FlowBean o) {
            // 按照总流量的倒序进行排序
            if (this.sumFlow > o.sumFlow) {
                return -1;
            } else if (this.sumFlow < o.sumFlow) {
                return 1;
            } else {
                // 总流量相等,按照上行流量倒序
                if (this.upFlow > o.upFlow) {
                    return -1;
                } else if (this.upFlow < o.upFlow) {
                    return 1;
                }
                return 0;
            }
        }
    }
    ```
  
  - Mapper类: `context(FlowBean,手机号)`
  
    ```java
    public class FlowMapper extends Mapper<LongWritable, Text, FlowBean, Text> {
        private FlowBean outK = new FlowBean();
        private Text outV = new Text();
    
        @Override
        protected void map(LongWritable key, Text value, Mapper<LongWritable, Text, FlowBean, Text>.Context context)
                throws IOException, InterruptedException {
            // 1 获取一行
            // 13630577991 6960 690 7650
            String line = value.toString();
            String[] list = line.split("\\s+");
    
            String phoneNum = list[0];
            String upFlow = list[list.length - 3];
            String downFlow = list[list.length - 2];
    
            outV.set(phoneNum);
            outK.setUpFlow(Long.parseLong(upFlow));
            outK.setDownFlow(Long.parseLong(downFlow));
            outK.setSumFlow();
            context.write(outK, outV);
        }
    }
    ```
  
  - Reducer类: 打印，循环输出
  
    ```java
    public class FlowReducer extends Reducer<FlowBean, Text, Text, FlowBean> {
    
        @Override
        protected void reduce(FlowBean key, Iterable<Text> values, Reducer<FlowBean, Text, Text, FlowBean>.Context context)
                throws IOException, InterruptedException {
            // 具有相同上行流量Phone(Phone是Values)的会在一起
            for (Text value : values) {
                // 逐行打印,先打印phone,再打印value
                context.write(value, key);
            }
        }
    }
    ```
  
  - Driver类: 修改map 的最终输出、最终任务的最终输出
  
    ```java
    // 4. 设置map输出的kv 类型
    job.setMapOutputKeyClass(FlowBean.class);
    job.setMapOutputValueClass(Text.class);
    
    // ReduceTask是1,才能全排序
    job.setNumReduceTasks(1);
    
    // 5 设置最终输出的KV类型(其实就是reducer阶段的输出类型)
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(FlowBean.class);
    ```

#### Shuffle 又分区(Partition) 又排序，分区类按照总流量倒序

在上面排序的基础上做如下操作:

```java
/* 文件 provincePartitioner2.java */
package com.luke.mapreduce.partitionerandwritableComparable;

import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Partitioner;

// 从Mapper出来 -> 经过Partitioner -> 进入Reducer,
// 所以Parritioner的输入和Mapper的输出一致
public class provincePartitioner2 extends Partitioner<FlowBean, Text> {
    @Override
    public int getPartition(FlowBean key, Text value, int numPartitions) {
        // key 是每个号码的流量信息, value 是手机号string
        String phone = value.toString();
        String prePhone = phone.substring(0, 3);
        int partition;
        if ("136".equals(prePhone)) {
            partition = 0;
        } else if ("137".equals(prePhone)) {
            partition = 1;
        } else if ("138".equals(prePhone)) {
            partition = 2;
        } else if ("139".equals(prePhone)) {
            partition = 3;
        } else {
            partition = 4;
        }
        return partition;
    }
}

/* 文件 FlowDriver.java */
// 设置分区信息
job.setPartitionerClass(provincePartitioner2.class);
job.setNumReduceTasks(5);
```

#### Shuffle 中的Combiner

背景:

- Reduce task要处理的东西更多 更繁忙，所以如果 map阶段能做一些聚合更好;
- Map task 的个数更多，并发度更高;

**Combiner合并**

1. Combiner 是MR程序中Mapper和Reducer之外的一种组件。并非必选项；

2. Combiner组件的父类就是Reducer;

3. Combiner和Reducer的区别在于运行的位置。

   - **Combiner 是在Map阶段运行的，在每个MapTask所在节点运行，针对每个MapTask的结果做合并;**
   - **Reducer是接收全局所有Mapper的输出结果**;

4. **Combiner 的意义就是对每一个MapTask的输出进行局部汇总，以减少网络流量**;

5. **<mark style="color:red">Combiner能够应用的前提是不能影响最终的业务逻辑。</mark>** 而且，Combiner的输出KV应该能跟Reducer的输出KV类型对应起来;

   比如，求平均值:
   ```java
   Mapper两个Task
   3 5 7 => combiner (3+5+7)/3=5
   2 6 => combiner (2+6)/2=4
   
   Reducer阶段:
   (3+5+7+2+6)/5=23/5  不等于  (5+4)/2=9/2
   ```

**继续搞上面的 WordCount 输出案例**

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221001122218298.png" alt="image-20221001122218298" style="zoom:67%;" />

**方案一:**

1. 增加一个WordcountCombiner类继承Reducer;

2. 在WordcountCombiner中

   - 统计单词汇总
   - 统计结果输出;

   ```java
   /* WordCountCombiner.java */
   package com.luke.mapreduce.combinerwordcount;
   import java.io.IOException;
   import org.apache.hadoop.io.IntWritable;
   import org.apache.hadoop.io.Text;
   import org.apache.hadoop.mapreduce.Reducer;
   
   public class WordCountCombiner extends Reducer<Text, IntWritable, Text, IntWritable> {
       private IntWritable outV = new IntWritable(0);
   
       @Override
       protected void reduce(Text key, Iterable<IntWritable> values,
               Reducer<Text, IntWritable, Text, IntWritable>.Context context) throws IOException, InterruptedException {
           int sum = 0;
           for (IntWritable value : values) {
               sum += value.get();
           }
           outV.set(sum);
           context.write(key, outV);
       }
   }
   
   /*  WordCountDriver.java */
   // 设置combiner
   job.setCombinerClass(WordCountCombiner.class);
   ```

**方案二:**

1. 将`WordcountReducer`作为`Combiner`在`WordcountDriver`驱动类中指定。也就是Map阶段也有一些Reduce的小任务先跑;

   `job.setCombinnerClass(WordcountReducer)`

   ```
   /*  WordCountDriver.java */
   // 设置combiner
   job.setCombinerClass(WordCountReducer.class);
   ```

#### OutputFormat接口实现类

OutputFormat是MapReduce输出的基类，所有实现MapReduce输出都实现了 OutputFormat接口。下面我们介绍几种场景的OutputFormat实现类:

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221001132749483.png" alt="image-20221001132749483" style="zoom:50%;" />

**默认输出格式TextOutputFormat;**

**自定义OutputFormat:**

- 应用场景: 如将数据输出到MySQL/HBase/Elasticsearch等存储框架中;
- 自定义OutputFormat步骤
  - 自定义一个类继承`FileOutputFormat`
  - 改写`RecordWriter`,具体改写输出数据的方法:`write()`

**示例: 自定义OutputFormat的案例**

**需求**
过滤输入的log日志，包含atguigu的网站输出到`atguigu.log`,否则输出到`other.log`

**输入数据:**

```java
/* it.log */
http://www.baidu.com
http://www.google.com
http://cn.bing.com
http://www.atguigu.com
http://www.sohu.com
http://www.sina.com
http://www.sin2a.com
http://www.sin2desa.com
http://www.sindsafa.com
```

**实现:**

1. 创建一个类 LogRecordWriter继承RecordWriter:
   - 创建两个文件的输出流: `atguiguOut`、`otherOut`
   - 如果输入数据包含atguigu,输出到 `atguiguOut`流； 否则输出到`otherOut`流

源码:

```java
/* LogMapper.java */
package com.luke.mapreduce.outputformat;

import java.io.IOException;

import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;

public class LogMapper extends Mapper<LongWritable, Text, Text, NullWritable> {
    @Override
    protected void map(LongWritable key, Text value, Mapper<LongWritable, Text, Text, NullWritable>.Context context)
            throws IOException, InterruptedException {
        // http://www.baidu.com
        // http://www.google.com
        // 不做任何处理
        context.write(value, NullWritable.get());
    }
}

/* LogReducer.java */
package com.luke.mapreduce.outputformat;

import java.io.IOException;

import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Reducer;

public class LogReducer extends Reducer<Text, NullWritable, Text, NullWritable> {
    @Override
    protected void reduce(Text key, Iterable<NullWritable> values,
            Reducer<Text, NullWritable, Text, NullWritable>.Context context) throws IOException, InterruptedException {
        // 防止有相同的key, 如果不遍历, 则相同key只输出一个了
        for (NullWritable val : values) {
            context.write(key, NullWritable.get());
        }
    }
}


/* LogOutputFormat.java */
package com.luke.mapreduce.outputformat;

import java.io.IOException;

import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.RecordWriter;
import org.apache.hadoop.mapreduce.TaskAttemptContext;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

public class LogOutputFormat extends FileOutputFormat<Text, NullWritable> {
    @Override
    public RecordWriter<Text, NullWritable> getRecordWriter(TaskAttemptContext job)
            throws IOException, InterruptedException {
        LogRecordWriter lrw = new LogRecordWriter(job);
        return lrw;
    }
}


/* LogRecordWriter.java */
package com.luke.mapreduce.outputformat;

import java.io.IOException;

import org.apache.hadoop.fs.FSDataOutputStream;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IOUtils;
import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.RecordWriter;
import org.apache.hadoop.mapreduce.TaskAttemptContext;

public class LogRecordWriter extends RecordWriter<Text, NullWritable> {
    private FSDataOutputStream atguiguOut;
    private FSDataOutputStream otherOut;

    public LogRecordWriter(TaskAttemptContext job) {
        // 创建两个流
        FileSystem fs;
        try {
            fs = FileSystem.get(job.getConfiguration());
            this.atguiguOut = fs
                    .create(new Path("/root/code/java/hdfs-client/output/outputformat_ret/atguigu.log"));

            this.otherOut = fs
                    .create(new Path("/root/code/java/hdfs-client/output/outputformat_ret/other.log"));
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void write(Text key, NullWritable value) throws IOException, InterruptedException {
        // 具体写入
        String line = key.toString();
        if (line.contains("atguigu")) {
            atguiguOut.writeBytes(line + "\n");
        } else {
            otherOut.writeBytes(line + "\n");
        }
    }

    @Override
    public void close(TaskAttemptContext context) throws IOException, InterruptedException {
        IOUtils.closeStream(atguiguOut);
        IOUtils.closeStreams(otherOut);
    }
}

/* LogDriver.java */
...
        // 设置自定义的outputFormat
        job.setOutputFormatClass(LogOutputFormat.class);

        // 6. 设置输入路径和输出路径
        FileInputFormat.setInputPaths(job, new Path("/root/code/java/hdfs-client/input/it.log"));
        // 虽然我们自定义了outputformat, 但是因为我们的outputformat 继承自 fileoutputformat
        // 而 fileoutputformat要输出一个 _SUCCESS文件,所以还得指定一个输出目录
        FileOutputFormat.setOutputPath(job, new Path("/root/code/java/hdfs-client/output/outputformat_ret"));
...
```

#### MapTask 工作机制

<img src="/Users/lukexwang/Library/Application Support/typora-user-images/image-20221001201048720.png" alt="image-20221001201048720" style="zoom:67%;" />

- 1/2/3/4 步都是客户端提交时执行的，属于提交流程。不属于MapReduce;
- Read阶段: 第 5步，从默认TextInputFormat找那个读取数据;
- Map阶段: 第6步，进入用户自己写的map方法;
- Collect阶段: 第7/8步，环形缓冲区中进行的 分区、排序 都属于该阶段;
- 溢写阶段: 第9步，从环形缓冲区中写入到文件中；
- Merge阶段: 第10步，对溢写的文件进行 归并排序；

#### ReduceTask 工作机制

![image-20221001202208279](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221001202208279.png)

- Copy阶段: 第10步，从MapTask中拉取自己指定分区的文件;
- Sort阶段: 第13步，对拉取的文件进行合并文件，归并排序;
- Reduce阶段: 相同的key 进入用户定义的Reduce方法;

#### ReduceTask 并行度决定机制

回顾: MapTask并行度由切片个数决定，切片个数由输入文件好切片规则决定;

思考: ReduceTask并行度由谁决定?

1.  **设置ReduceTask并行度(个数)**

ReduceTask的并行度同样影响整个job的执行并发度和执行效率，但与MapTask的并发数由分片数决定不同，ReduceTask数量的决定是可以手动设置:
```java
// 默认值1, 手动设置 4
job.setNumReduceTasks(4);
```

2. **实验: 测试ReduceTask多少合适，正常生产环境中，也需要自己去测试**

   实验环境: 1个master节点，16个Slave节点: CPU 8GHZ，内存 2G

   实验结论: 
   <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221001205953142.png" alt="image-20221001205953142" style="zoom:50%;" />

3. **注意事项**

   - **ReduceTask=0, 表示没有Reduce阶段，输出文件个数和Map个数保持一致**;
   - **ReduceTask 默认值是1，所有输出文件为一个**;
   - **如果数据分布不均，就有可能在Reduce阶段产生数据倾斜**
   - **ReduceTask数量并不是任意设置的，还需要考虑业务逻辑需求，有些情况下，需要计算全局汇总结果，就只能1个ReduceTask**
   - **具体多少ReduceTask, 需要根据集群性能而定**
   -  **<mark style="color:red">如果分区数不是1，但是ReduceTask为1，是否执行分区过程。答案是: 不执行分区过程。因为在MapTask的源码中，执行分区的前提是先判断ReduceNum个数是否大于1。不大于1肯定不执行</mark>**

#### Reduce Join操作

1. Map端的主要工作: 为来自不同表或文件的key/value对，<mark style="color:red">打标签以区别不同来源的记录。</mark>然后用<mark style="color:red">连接字段作为key</mark>，其余部分和新加的标志作为value，最后进行输出;
2. Reduce端的主要工作: 在Reduce端<mark style="color:red">以连接字段作为key的分组已经完成，</mark>我们只需在每一个分组当中将那些来源于不同文件的记录(在Map阶段已经打标志)分开，最后进行合并就ok了。

**缺点:**
**合并操作是在Reduce阶段完成的，Reduce端处理压力大，Map端运算负载则很低，资源利用不足，且在Reduce端极容易产生数据倾斜。**

**案例实操:**
两个表:

- 订单数据表 `t_order`

| id   | <mark style="color:red">Pid</mark> | Amount |
| ---- | ---------------------------------- | ------ |
| 1001 | <mark style="color:red">01</mark>  | 1      |
| 1002 | <mark style="color:red">02</mark>  | 2      |
| 1003 | <mark style="color:red">03</mark>  | 3      |
| 1004 | <mark style="color:red">01</mark>  | 4      |
| 1005 | <mark style="color:red">02</mark>  | 5      |
| 1006 | <mark style="color:red">03</mark>  | 6      |

- 商品信息表 `t_product`

| <mark style="color:red">Pid</mark> | Pname |
| ---------------------------------- | ----- |
| <mark style="color:red">01</mark>  | 小米  |
| <mark style="color:red">02</mark>  | 华为  |
| <mark style="color:red">03</mark>  | 格力  |

![image-20221004155153077](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221004155153077.png)

**实现 TableBean**

```java
/* TableBean.java */
package com.luke.mapreduce.reduceJoin;
import java.io.DataInput;
import java.io.DataOutput;
import java.io.IOException;
import org.apache.hadoop.io.WritableComparable;

public class TableBean implements WritableComparable<TableBean> {
    private long orderID; // 订单id
    private long pid; // 商品id
    private long amount; // 商品数量
    private String pname; // 商品名称
    private String tname; // 表名

    public String getTname() {
        return tname;
    }
    public void setTname(String tname) {
        this.tname = tname;
    }
    public long getOrderID() {
        return orderID;
    }
    public void setOrderID(long orderID) {
        this.orderID = orderID;
    }
    public long getPid() {
        return pid;
    }
    public void setPid(long pid) {
        this.pid = pid;
    }
    public long getAmount() {
        return amount;
    }
    public void setAmount(long amount) {
        this.amount = amount;
    }
    public String getPname() {
        return pname;
    }
    public void setPname(String pname) {
        this.pname = pname;
    }
    public TableBean() {
    }
    @Override
    public void write(DataOutput out) throws IOException {
        out.writeLong(orderID);
        out.writeLong(pid);
        out.writeLong(amount);
        out.writeUTF(pname);
        out.writeUTF(tname);
    }
    @Override
    public void readFields(DataInput in) throws IOException {
        this.orderID = in.readLong();
        this.pid = in.readLong();
        this.amount = in.readLong();
        this.pname = in.readUTF();
        this.tname = in.readUTF();
    }
    @Override
    public String toString() {
        return this.orderID + "\t" + this.pname + "\t" + this.amount;
    }
    @Override
    public int compareTo(TableBean o) {
        // 按照 订单ID 正序
        if (this.orderID > o.orderID) {
            return 1;
        } else if (this.orderID < o.orderID) {
            return -1;
        } else {
            return 0;
        }
    }
}
```

```java
/* TableMapper.java */
package com.luke.mapreduce.reduceJoin;

import java.io.IOException;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.lib.input.FileSplit;

public class TableMapper extends Mapper<LongWritable, Text, LongWritable, TableBean> {
    private String fileName;
    private LongWritable outK = new LongWritable();
    private TableBean outV = new TableBean();
    // setup方法,两个文件，,每个执行一次
    @Override
    protected void setup(Mapper<LongWritable, Text, LongWritable, TableBean>.Context context)
            throws IOException, InterruptedException {
        // 读取的是 order.txt, pd.txt 两个文件,每个文件一个切片
        FileSplit split = (FileSplit) context.getInputSplit();
        fileName = split.getPath().getName();
    }
    // 注意输出的outK 就是用于Join的 pid (商品id)
    @Override
    protected void map(LongWritable key, Text value,
            Mapper<LongWritable, Text, LongWritable, TableBean>.Context context)
            throws IOException, InterruptedException {
        // 1 获取一行
        // 内容可能是:
        // t_order 表
        // 1001 01 1
        //
        // t_product 表
        // 01 小米
        String line = value.toString();
        String[] list = line.split("\\s+");
        if (fileName.contains("order")) {
            outV.setOrderID(Long.parseLong(list[0]));
            outV.setPid(Long.parseLong(list[1]));
            outK.set(Long.parseLong(list[1]));
            outV.setAmount(Long.parseLong(list[2]));
            outV.setPname("");
            outV.setTname("t_order");
        } else if (fileName.contains("pd")) {
            outV.setOrderID(0); // 尽管是0也是必须设置的
            outV.setPid(Long.parseLong(list[0]));
            outK.set(Long.parseLong(list[0]));
            outV.setAmount(0);
            outV.setPname(list[1]);
            outV.setTname("t_product");
        }
        context.write(outK, outV);
    }
}
```

```java
/* TableReducer.java */
package com.luke.mapreduce.reduceJoin;

import java.io.IOException;
import java.lang.reflect.InvocationTargetException;
import java.util.ArrayList;
import org.apache.commons.beanutils.BeanUtils;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.mapreduce.Reducer;

public class TableReducer extends Reducer<LongWritable, TableBean, TableBean, NullWritable> {
    @Override
    protected void reduce(LongWritable key, Iterable<TableBean> values,
            Reducer<LongWritable, TableBean, TableBean, NullWritable>.Context context)
            throws IOException, InterruptedException {
        // 输入数据示例:
        // 01 1001 1 order
        // 01 1004 4 order
        // 01 小米 pd
        ArrayList<TableBean> orders = new ArrayList<>();
        TableBean myPd = new TableBean();
        for (TableBean val : values) {
            if ("t_order".equals(val.getTname())) {
                // 必须有这一步,否则for循环结束后,所有orders中指向的是同一个TableBean val(也就是values中的最后一个)
                // 这是mapreduce的优化
                TableBean tmp = new TableBean();
                try {
                    BeanUtils.copyProperties(tmp, val); // 很重要
                } catch (IllegalAccessException | InvocationTargetException e) {
                    e.printStackTrace();
                }
                orders.add(tmp);
            } else if ("t_product".equals(val.getTname())) {
                try {
                    BeanUtils.copyProperties(myPd, val); // 很重要
                } catch (IllegalAccessException | InvocationTargetException e) {
                    e.printStackTrace();
                }
            }
        }
        for (TableBean bitem : orders) {
            bitem.setPname(myPd.getPname());
            context.write(bitem, NullWritable.get());
        }
    }
}
```

#### MapJoin

**使用场景: MapJoin适用于一张表很小，一张表很大的场景。**
**优点: 在Map端缓存多张表,提前处理业务逻辑，这样增加Map端业务。减少Reduce端数据压力，尽可能减少数据倾斜。**
**具体方法: 采用DistributedCache**

1. 在Mapper的Setup阶段，将文件读取到缓存集合中。**setup方法完成后，才会执行自定义的Map逻辑**;

2. 在Driver驱动类中加载缓存
   ```java
   // 缓存普通文件到 Task 运行节点
   job.addCacheFile(new URI("file:/path/fo/pd.txt"));
   // 如果是集群运行,需要设置HDFS路径
   job.addCacheFile(new Uri()"hdfs://hadoop101:8020/cache/pd.txt");
   ```

还是上面的案例，代码情况如下:
```java
/* MapJoinDriver.java main函数内容 */
        // 1. 获取job
        Configuration conf = new Configuration();
        Job job = Job.getInstance(conf);
        // 2 设置jar包路径
        job.setJarByClass(MapJoinDriver.class);
        // 3 关联 mapper, 无需设置关联的 reducer 类
        job.setMapperClass(MapJoinMapper.class);
        // 4. 设置map输出的kv 类型
        job.setMapOutputKeyClass(Text.class);
        job.setMapOutputValueClass(NullWritable.class);

        // ReduceTask是0,map阶段join已经ok,无需reduce阶段
        job.setNumReduceTasks(0);

        // 5 设置最终输出的KV类型(因为不再有redusce阶段,所以map阶段的输出类型就是最终类型)
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(NullWritable.class);

        // 加载小文件(pd.txt)缓存数据
        job.addCacheFile(new URI("file:/root/code/java/hdfs-client/input/reducerJoin/pd.txt"));

        // 6 设置输入路径和输出路径
        // 输入主要是为了读取 order 订单数据
        FileInputFormat.setInputPaths(job,
                new Path("/root/code/java/hdfs-client/input/reducerJoin/order.txt"));
        FileOutputFormat.setOutputPath(job, new Path("/root/code/java/hdfs-client/output/mapJoin"));

        // 7 提交job
        boolean ret = job.waitForCompletion(true);
        System.exit(ret ? 0 : 1);

/* MapJoinMapper.java */
package com.luke.mapreduce.mapJoin;
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.net.URI;
import java.util.HashMap;
import org.apache.commons.lang3.StringUtils;
import org.apache.hadoop.fs.FSDataInputStream;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IOUtils;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;

public class MapJoinMapper extends Mapper<LongWritable, Text, Text, NullWritable> {
    private HashMap<String, String> pdMap = new HashMap<>();
    private Text outK = new Text();

    @Override
    protected void setup(Mapper<LongWritable, Text, Text, NullWritable>.Context context)
            throws IOException, InterruptedException {
        // 获取缓存的文件 pd.txt, 并把文件内容封装到集合
        URI[] cacheFiles = context.getCacheFiles();

        FileSystem fs = FileSystem.get(context.getConfiguration());
        FSDataInputStream fis = fs.open(new Path(cacheFiles[0]));

        // 从流中读取数据
        BufferedReader reader = new BufferedReader(new InputStreamReader(fis, "UTF-8"));
        String line;
        while (StringUtils.isNotEmpty(line = reader.readLine())) {
            String[] fields = line.split("\\s+");
            pdMap.put(fields[0], fields[1]);
        }
        // 关流
        IOUtils.closeStream(reader);
    }

    @Override
    protected void map(LongWritable key, Text value, Mapper<LongWritable, Text, Text, NullWritable>.Context context)
            throws IOException, InterruptedException {
        // 处理 order.txt
        String line = value.toString();
        String[] fields = line.split("\\s+");

        // 获取pid
        String pid = fields[1];
        String pname = pdMap.get(pid);
        String str = fields[0] + "\t" + pname + "\t" + fields[2];
        outK.set(str);
        context.write(outK, NullWritable.get());
    }
}
```

