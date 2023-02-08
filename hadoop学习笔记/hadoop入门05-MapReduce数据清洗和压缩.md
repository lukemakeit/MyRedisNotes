### 数据清洗 (ETL)

"ETL", 是英文`Extract-Transform-Load`的缩写，用于描述将数据从源端经过抽取(Extract)、转换(Transform)、加载(Load)至目的端的过程。ETL一词较常用在数据仓库，但其对象不不限于数据仓库。
在运行核心业务MapReduce程序之前，往往需要先对数据进行清洗，清理掉不符合用户要求的数据。**清理的过程往往只需要运行Map程序，而不需要运行Reduce程序。**
一个简要示例:

```java
/*   */
public class etlDemoMapper extends Mapper<LongWritable, Text, Text, NullWritable> {
    @Override
    protected void map(LongWritable key, Text value, Mapper<LongWritable, Text, Text, NullWritable>.Context context)
            throws IOException, InterruptedException {
        // 1 读取一行
        String line = value.toString();

        // 2 ETL
        boolean result = parseLog(line, context);
        if (!result) {
            return;
        }

        // 3 写出
        context.write(value, NullWritable.get());
    }

    protected boolean parseLog(String line, Mapper<LongWritable, Text, Text, NullWritable>.Context context) {
        return line.contains("error");
    }
}
```

### 数据压缩

优点: 减少磁盘IO，减少磁盘存储空间;
缺点: 增加CPU开销;

**压缩原则:**

1. 运算密集型Job，少用压缩;
2. IO密集型Job, 多用压缩;

![image-20221005131444211](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221005131444211.png)

![image-20221005140348107](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221005140348107.png)

Snappy 压缩能达到每秒钟 250MB/s 或者更多，解压缩能达到500MB/s 或者更多。

**压缩方式选择**
压缩方式选择时重点考虑: **压缩/解压缩速度、压缩率(压缩后存储大小)、压缩后是否支持切片。**

**MapReduce数据压缩**

![image-20221005141119092](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20221005141119092.png)

**Hadoop中启用压缩，可以配置如下参数:**

| 参数                                                         | 默认值                                              | 阶段        | 建议                                            |
| ------------------------------------------------------------ | --------------------------------------------------- | ----------- | ----------------------------------------------- |
| <mark style="color:red">`io.compress.codecs`(在`core-site.xml`中配置)</mark> | 无,这个需要在命令行通过输入`hadoop checknative`查看 | 输入压缩    | Haoop使用文件扩展名判断是否支持某种编解码器     |
| <mark style="color:blue">`mapreduce.map.output.compress`(在`mapred-site.xml`中配置)</mark> | `false`                                             | mapper输出  | 这个参数设置为`true`启用压缩                    |
| <mark style="color:blue">`mapreduce.map.output.compress.codec`(在`mapred-site.xml`中配置)</mark> | `org.apache.hadoop.io.compress.DefaultCodec`        | mapper输出  | 企业多使用`LZO`或`Snappy`编码器在此阶段压缩数据 |
| <mark style="color:purple">`mapreduce.output.fileoutputformat.compress`(在`mapred-site.xml`中配置)</mark> | `false`                                             | reducer输出 | 这个参数设置喂`true`启用压缩                    |
| <mark style="color:purple">`mapreduce.output.fileoutputformat.compress.codec`(在`mapred-site.xml`中配置)</mark> | `org.apache.hadoop.io.compress.DefaultCodec`        | reducer输出 | 使用标准工具或者编解码器,如gzip 和 bzip2        |

**Map输出端采用压缩**
即使MapReduce的输入输出文件都是未压缩文件，你仍然**可以对Map任务的中间结果输出做压缩,因为它要写在磁盘并通过网络传输到Reduce节点，对其压缩可提高很多性能，这些工作只需要设置连个属性即可。**

Hadoop源码支持的压缩格式有: `Bzip2Codec`、`DefaultCodec`

```java
/* Driver.java 文件中 */

// 1. 获取job
Configuration conf = new Configuration();
// 开启map端输出压缩
conf.setBoolean("mapreduce.map.output.compress", true);
// 设置map端输出压缩方式
conf.setClass("mapreduce.map.output.compress.codec", BZip2Codec.class, CompressionCodec.class);
```

**Reduce输出端采用压缩**

```java
// 设置Reduce端输出 压缩开启
FileOutputFormat.setCompressOutput(job, true);
// 设置压缩的方式
FileOutputFormat.setOutputCompressorClass(job, GzipCodec.class);
// FileOutputFormat.setOutputCompressorClass(job,DefaultCodec.class);
```
