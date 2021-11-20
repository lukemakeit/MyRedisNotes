## RocksDB基础概念

### LSM-Tree

**背景说明**

参考文章:

- [深入理解什么是LSM-Tree](https://cloud.tencent.com/developer/article/1441835)

- [Log Structured Merge Trees](http://www.benstopford.com/2015/02/14/log-structured-merge-trees/)

- [LSM 算法的原理是什么?](https://www.zhihu.com/question/19887265/answer/78839142) 这个回答是上面文章的翻译
- [RocksDB 笔记](http://alexstocks.github.io/html/rocksdb.html)

磁盘随机读写慢，顺序读写快。无论是SATA 还是 SSD。

对于写操作吞吐敏感的系统，像日志或者堆文件，将新数据直接添加到文件，完全顺序写入，即可提供非常好的写性能。

但基于日志的策略并不利于读取，从日志文件中读取数据需要倒序扫描找到所需内容，读将比写需要更多时间。

这说明日志策略仅适用于一些简单场景:

1. 数据被整体访问，大部分数据的[WAL(write ahead log)](https://zhuanlan.zhihu.com/p/137512843);
2. 知道明确的offset，如kafka中;

我们需要更多的日志来为更复杂的读场景(如按keey 或 range)提供高效性能，一般有四个方法完成:

1. **二分查找**：将文件数据有序保存，使用二分查找完成特定key的查找；
2. **hash**：用hash将数据分割到不同的bucket;
3. **B+树**：用B+树 或 ISAM等方法，减少外部文件的读取；
4. **外部文件**：数据存为日志，并创建hash 或者 查找树映射相应的文件。

这些方法提高了读操作想性能，但是却丢失了日志策略超好的写性能。上面这些方法，都给数据加上了结构信息，数据被按照特定的方式放置，所以可以较快的找到特定数据，但我们需要更新hash 或者B+树的结构时，同时需要更新文件系统中特定部分，这就造成了随机读写。进而导致写操作性能下降。

一种常见的解决方案是使用 方法4 为日志创建一个索引，同时保存内存中保存索引信息。如一个大的hash用于映射keys 到 value的offset(日志文件中的最新值)。该方法能有效减少随机IO次数。

不过也带来了扩展性的现在，特别是如果我们有很多小值。如果这些值只是一些简单的数字，那么索引的数据量将大于数据本身。

不过这也是一个明智的方案，从Riak 到 Orachle Coherence都使用了该方案。

LSM(Log-Structured-Merge Tree) 使用了一种不同于上述四种的方法，保持了日志文件写性能，以及微小的读操作性能损失。他完全以磁盘为中心，几乎不需要内存来提高效率，同时也保留了我们与简单日志文件相关联的大部分写入性能。与B+Tree相比，一个缺点是读性能稍差。

#### LSM基础算法

概念上，LSM树非常简单。没有一个巨大的索引结构，批量的写入会被顺序保存到一系小的索引文件中。所以每个文件保存了一小段时间的更改。**每个文件都是有序的 所以后面检索会很快。文件是不能修改的。新的数据更新会写入到新文件中。读取将检查所有文件。同时文件会定期合并，以减少文件数据。**

来看一些实现细节：

- 当更新操作到达时，他们将被添加到 内存buffer中(也就是memtable)，memtable通常保存为一个Tree(红黑树等)以保证 key的有序性。

- memtable通过WAL的方式备份到磁盘，以便能恢复，防止数据丢失。

- 当memtable满了，有序的数据会被flush到一个新的磁盘文件中。当更多的写入到来时，该过程会不断重复。重要的是，系统只做顺序IO，因为没有文件被编辑，新的数据 或 修改只用简单的生成新文件(.sst文件)。

- 当更多数据到达时，会有更多无法修改、顺序的文件会被创建。每个文件代表一个小的、按时间顺序更改的、有序子集；

- 旧文件不再更新，因此新建重复项以覆盖前面的记录(或删除标记)。这会产生一些冗余；

- 系统会周期性的执行 compation操作。compation 会选择多个文件 然后将他们合并，移除 重复的更新 或 已删除的项。这对清理冗余很重要，同时更重要的是处理因文件数量大量增长 而 导致读性能下降问题。值得庆幸的是，因文件是有序的，合并文件的过程效率也很高；

- 当接收到读请求时，系统首先检查内存buffer(也就是memtable)。如果memtable中不存在，则按照文件生成时间顺序倒序检查 磁盘文件，直到key被找到。每个文件都经过排序，因此可以导航，然而随着文件数量增加，每个文件都需要检查，读取会变得越来越慢。这是 一个问题；

- 所以LSM Treee的读取比其他做in-place更新的数据库慢。幸运的是，也有一些小技巧让读取性能更好。最常用的方法是在内存中保存 page-index(页面索引)。这提供了一个查找，使你接近你的目标 key。LevelDB、RocksDB 和 BigTable通过在每个文件末尾保存块索引来执行此操作。这通常比直接二进制搜索更有效，因为它允许使用变长字段且更适合压缩数据。

- 及时每个文件都有索引，在文件数量剧增时读取操作依然很慢。只能通过定期做compaction，让文件合并，将文件数量和读取性能保持在可接受范围内。

- 通过compaction，虽然能减少文件数据量 读操作但依然需要访问大量的文件。大多数通过使用布隆过滤器来避免这种情况。布隆过滤器是一种确定文件是否包含key的高效内存方式。

- 所以从 “写”的角度来看，所有的写入都是 一批一批写入的，并且是连续的块写入。compaction 会产生额外的、周期性IO成本。然而在进行读取时，可能会读取到大量文件。这就是算法的工作方式。我们以读取时的随机IO 替换 交换 写入的随机IO。如果我们能用布隆过滤器等软件技巧 或者 大文件缓存等硬件技巧来优化读性能，那么这种权衡是明智的。

  ![image-20211119172131999](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211119172131999.png)

#### Basic Compation

为了让LSM读取相对更快，管理文件数量很重要，所以我们来看下compaction这个过程。这个过程有点像分代(generational)垃圾回收：

当创建了一定数量的文件，如5个文件，每个文件10行数据，他们合并到一个文件，这个文件就是50行数据(可能会更少点)。

该过程将继续创建更多 10行数据的文件，每次第5个文件填满时，这些文件都会合并为 一个50行的文件。

最后 这里有5个50行数据的文件。某个时间点，5个50行数据的文件会被合并成一个250行数据的文件。该过程将继续创建更多 更大的文件。

根据前面的描述，随着大量文件被创建：读取一个key，最差情况下会所有文件都被搜索。

![image-20211119160711343](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211119160711343.png)

#### Levelled Compaction

在较新的实现中，例如LevelDB、RocksDB 和 Cassandra中的实现，通过基于level 而不是基于size的compaction的压缩来解决这个问题。这减少了最坏情况下读取操作必须检索的文件数量，并减少单次compaction的影响。

基于level的compaction 对比 base compaction有两个关键的区别：

1. 每个level都包含多个文件(**某个level保存的文件数是固定的？**)，并保证作为的一个整体其中没有重复的keys。也就是说，keys 横跨多个可用文件被 partitioned 。因此在某个level查找一个key，只需查阅一个文件即可；

   level 0 是很特殊，不满足上述性质。keys可以跨域多个文件(同一个key的多次修改保存在不同文件中)。

2. 某个时间，文件会被合并到更高一层的level的一个文恶剪中。当一个level 填满时，一个文件会被提取出来并合并到更高一层的level中(这个level会创建空间以便更多数据加入)。这和 base compaction一点细微区别：base compaction是将几个相似大小的文件合并为一个更大的文件。

这些区别也意味着基于level的compaction 可以随着时间的推移分散compaction的影响，并且需要更少的空间。它还具有更好的性能。然而，对大多数工作负载来说，总IO更高，这也意味着一些更简单的大部分只是写入的工作负载将不会受益。

### RocksDB的一些概念

![image](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/20211119234624.png)

#### Column family

参考内容:

[What is the point of column families?](https://dba.stackexchange.com/questions/166159/what-is-the-point-of-column-families) 、[中文版](https://qastack.cn/dba/166159/what-is-the-point-of-column-families)

1. Column Family的用途是啥？

   - 不同的数据部分使用不同的compaction 配置、比较器、压缩类型、merge operators 或 compactions filter;
   - 通过删除column family以便删除数据；
   - 一个column family用于存储元数据(meta)、另一个column family用于存储真正的数据；

2. 将数据存储在不同的 column family 和 存储在多个rocksdb数据库中有啥不同？

   最主要的区别是备份、原子写入 和 写入性能。

   使用多个rocksdb数据库的优势是：数据库是备份 和 checkpoint的单位。拷贝一个数据库到另一台机器 比 一个column family更容易。

   使用multi column family的优势是：(1) 一个数据库中跨多个column family批量写入是原子的，而多个rocksdb无法实现这一点。(2) 如果对WAL进行同步写入，多个数据库可能会随时性能。

3. 我有不同的key spaces，我应该通过前缀来将它们进行区分 还是 使用不同的column families？

   - 如果每个key space很大，那将他们放到不同的column families中更好。
   - 如果每个key space不大，那你应该考虑将多个key spaces放到同一个column family中，以避免维护多个column family麻烦。

一些点：

- 每个KV都会关联一个column family，其中默认的column family是"default";

- 不同的column family共享WAL，不过不同的column family都有自己的 memtable 和 SST。这也意味着我们可以给不同的column family设置不同的属性，同时能快速删除对应的column family；

  