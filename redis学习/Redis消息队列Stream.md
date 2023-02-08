## Redis消息队列: stream

原文地址:  
[基于Redis的Stream类型的完美消息队列解决方案](https://zhuanlan.zhihu.com/p/60501638)  

消息队列功能包括但不限于:  
- 消息ID的序列化生成;
- 消息遍历;
- 消息的阻塞和非阻塞读取;
- 消息的分组消费;
- 未完成消息的处理;
- 消息队列监控;

#### 追加新消息,XADD,生产消息
XADD，命令用于在某个stream(流数据)中追加消息，演示如下:
```cpp
127.0.0.1:9001> XADD memberMessage * user kang msg Hello
"1652910665791-0"
127.0.0.1:9001> XADD memberMessage * user zhong  msg nihao
"1652910673025-0"
```
语法格式: `XADD key ID field string [field string ...]`
- key 没啥好说的;
- 消息ID 最常用的是`*`,表示Redis生成消息ID,这也是强烈建议的方案;
- `field string [field string]`: 当前消息的内容,由1个或多个key-value构成;  
- 返回值就是 新增消息的 消息ID。消息ID由两个64位的数字组成,ID第一个数字代表unix毫秒时间,第二个数字是相同毫秒时间能消息的序列号;
- xadd中支持限制消息队列的大小: `XADD mystream MAXLEN ~ 1000 * ... entry fields here ...`,限制消息队列大概在1000大小左右,如果要精确控制则替换`~`为`=`;  

#### 从消息队列中 阻塞/非阻塞 方式获取消息:XREAD
```cpp
127.0.0.1:9001> XREAD streams  memberMessage  0
1) 1) "memberMessage"
   2) 1) 1) "1652910665791-0"
         2) 1) "user"
            2) "kang"
            3) "msg"
            4) "Hello"
      2) 1) "1652910673025-0"
         2) 1) "user"
            2) "zhong"
            3) "msg"
            4) "nihao"
```
上面的命令是从消息队列memberMessage中读取所有消息。  
语法格式: `XREAD [COUNT count] [BLOCK milliseconds] STREAMS key [key ...] ID [ID ...]`
其中:
- `[COUNT count]`: 限定获取的消息数量;  
- `[BLOCK milliseconds]`: 用于设置XREAD为阻塞模式,默认为非阻塞模式;  
- `ID`: 用于设置由哪个消息ID开始读取。使用0表示从第一条消息开始。(本例中就是使用0）此处需要注意，消息队列ID是单调递增的，所以通过设置起点，可以向后读取。在阻塞模式中，可以使用`$`，表示当前最大消息后的消息(也就是最新的)消息`ID`,类似于`tail -f`。（在非阻塞模式下`$`无意义）。  

XRED读消息时分为阻塞和非阻塞模式，使用`BLOCK`选项可以表示阻塞模式，需要设置阻塞时长。
非阻塞模式下，读取完毕（即使没有任何消息）立即返回，而在阻塞模式下，若读取不到内容，则阻塞等待。  
典型的阻塞模式用法为:  
```cpp
127.0.0.1:9001> XREAD block 1000 streams memberMessage $
(nil)
(1.07s)
```
因此，典型的队列就是 XADD 配合 XREAD Block 完成。XADD负责生成消息，XREAD负责消费消息。  

#### 消息ID说明
`XADD`生成的`1652910665791-0`,就是Redis生成的消息ID。它由两部分组成: 时间戳-序号。时间戳是毫秒级的单位,生成消息的Redis服务器时间，它是个64位整型(int64)。  
序号为这个毫秒时间内的消息序号,也是个int64。  
通过multi批处理,验证序号递增:
```cpp
127.0.0.1:9001> MULTI
OK
127.0.0.1:9001> XADD memberMessage * msg one
QUEUED
127.0.0.1:9001> XADD memberMessage * msg two
QUEUED
127.0.0.1:9001> XADD memberMessage * msg three
QUEUED
127.0.0.1:9001> XADD memberMessage * msg four
QUEUED
127.0.0.1:9001> XADD memberMessage * msg five
QUEUED
127.0.0.1:9001> EXEC
1) "1652912173729-0"
2) "1652912173729-1"
3) "1652912173729-2"
4) "1652912173729-3"
5) "1652912173729-4"
```
由于一个redis命令的执行很快，所以可以看到在同一时间戳内，是通过序号递增来表示消息的。  
为了保证消息是有序的，因此Redis生成的ID是单调递增有序的。由于ID中包含时间戳部分，为了避免服务器时间错误而带来的问题（例如服务器时间延后了），Redis的每个Stream类型数据都维护一个`latest_generated_id`属性，用于记录最后一个消息的ID。若发现当前时间戳退后（小于`latest_generated_id`所记录的），则采用时间戳不变而序号递增的方案来作为新消息ID（这也是序号为什么使用int64的原因，保证有足够多的的序号），从而保证ID的单调递增性质。  

强烈建议使用Redis的方案生成消息ID，因为这种时间戳+序号的单调递增的ID方案，几乎可以满足你全部的需求。但同时，记住ID是支持自定义的，别忘了！  

#### 获取消息队列元素个数: XLEN
```cpp
127.0.0.1:9001> xlen memberMessage
(integer) 7
```
#### XRANGE 和 XREVRANGE
XRANGE命令返回stream的全部消息.
```cpp
127.0.0.1:9000> xrange memberMessage - +
1) 1) "1654399915425-0"
   2) 1) "user"
      2) "kang"
      3) "msg"
      4) "Hello"
2) 1) "1654399934474-0"
   2) 1) "user"
      2) "zhong"
      3) "msg"
      4) "nihao"
3) 1) "1654670729984-0"
   2) 1) "zhangsan"
      2) "19"
      3) "lisi"
      4) "20"
...
```
返回**某个时间段内**的消息:
```cpp
127.0.0.1:9000> xrange memberMessage 1654670747952 1654670834414
1) 1) "1654670747952-0"
   2) 1) "luxi"
      2) "19"
      3) "wangwu"
      4) "25"
2) 1) "1654670834414-0"
   2) 1) "a"
      2) "1"
      3) "b"
      4) "2"
      5) "c"
      6) "3"
      7) "d"
      8) "4"
```
XRANGE支持指定每次返回的消息条数,并从上一次读取的位置继续读取:
```cpp
127.0.0.1:9000> xrange memberMessage - + count 1 //获取一条消息 以及 这条消息的ID
1) 1) "1654399915425-0"
   2) 1) "user"
      2) "kang"
      3) "msg"
      4) "Hello"
127.0.0.1:9000> 
127.0.0.1:9000> xrange memberMessage  (1654399915425-0 + count 2 //获取消息ID之后的额外两条消息
1) 1) "1654399934474-0"
   2) 1) "user"
      2) "zhong"
      3) "msg"
      4) "nihao"
2) 1) "1654670729984-0"
   2) 1) "zhangsan"
      2) "19"
      3) "lisi"
      4) "20"
```
XREVRANGE和XRANGE命令类似,不过返回的元素的倒序的. 注意:`XREVRANGE`命令以相反的顺序使用`start`、`stop`参数.  
```cpp
127.0.0.1:9000> xrange memberMessage - +
1) 1) "1654399915425-0"
   2) 1) "user"
      2) "kang"
      3) "msg"
      4) "Hello"
2) 1) "1654399934474-0"
   2) 1) "user"
      2) "zhong"
      3) "msg"
      4) "nihao"
... //省略掉部分消息
6) 1) "1654671267997-0"
   2) 1) "aa"
      2) "11"
      3) "bb"
      4) "22"
7) 1) "1654671275530-0"
   2) 1) "cc"
      2) "33"
8) 1) "1654671280688-0"
   2) 1) "dd"
      2) "44"
      3) "ee"
      4) "55"
9) 1) "1654671288306-0"
   2) 1) "ff"
      2) "66"
      3) "gg"
      4) "77"
127.0.0.1:9000> xrevrange memberMessage + - count 1 //注意是 + - 位置
1) 1) "1654671288306-0"
   2) 1) "ff"
      2) "66"
      3) "gg"
      4) "77"
127.0.0.1:9000> xrevrange memberMessage (1654671288306-0 - count 2 //利用上个命令的消息ID,继续获取两条消息
1) 1) "1654671280688-0"
   2) 1) "dd"
      2) "44"
      3) "ee"
      4) "55"
2) 1) "1654671275530-0"
   2) 1) "cc"
      2) "33"
```

#### 消费者组模式,consumer group
当多个消费者（consumer）同时消费一个消息队列时，可以重复的消费相同的消息，就是消息队列中有10条消息，三个消费者都可以消费到这10条消息。  
但有时，我们需要多个消费者配合协作来消费同一个消息队列，就是消息队列中有10条消息，三个消费者分别消费其中的某些消息，比如消费者A消费消息1、2、5、8，消费者B消费消息4、9、10，而消费者C消费消息3、6、7。也就是三个消费者配合完成消息的消费，可以在消费能力不足，也就是消息处理程序效率不高时，使用该模式。该模式就是消费者组模式。
- 消费者组是对应 消息队列产生的,所以创建消费者组时,必须指定 目标消息队列;
- 每个消费者组 在 某个消息队列中都有唯一确定的名字;
- 一个消费者组创建后, 还能消费其他队列不？感觉是可以的,因为 XREADGROUP命令后面都可以跟多个key。  
根据[官网文档](https://redis.io/docs/manual/data-types/streams/)解释,是可以在一个XREADGROUP中同时消费多个 消息队列的。但条件是每个消息队列上都创建了同名的 消费者组;  
> Even with XREADGROUP you can read from multiple keys at the same time, however for this to work, you need to create a consumer group with the same name in every stream. This is not a common need, but it is worth mentioning that the feature is technically available.
- 一个消息队列 可以 对应着 多个 consumerGroup, 同时也能服务 通过XREAD读取数据的client;  

如下图所示:
![20220519070547](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/20220519070547.png)

消费者组模式的支持主要由两个命令实现：  
- `XGROUP`:用于管理消费者组，提供创建组，销毁组，更新组起始消息ID等操作;
- `XREADGROUP`，分组消费消息操作;
```cpp
> XREADGROUP GROUP group consumer [COUNT count] [BLOCK milliseconds] [NOACK] STREAMS key [key ...] id [id ...]
```

演示如下:  
```cpp
# 生产者生成5条消息
127.0.0.1:9001> multi
OK
127.0.0.1:9001> XADD mq * msg 1
QUEUED
127.0.0.1:9001> XADD mq * msg 2
QUEUED
127.0.0.1:9001> XADD mq * msg 3
QUEUED
127.0.0.1:9001> XADD mq * msg 4
QUEUED
127.0.0.1:9001> XADD mq * msg 5
QUEUED
127.0.0.1:9001> exec
1) "1652913903153-0"
2) "1652913903153-1"
3) "1652913903153-2"
4) "1652913903153-3"
5) "1652913903153-4"

# 该命令为mq这个消息队列创建了一个消费者组 consumGroup,
# 0代表从消息队列第一条消息开始消费,将0替换成$,代表从消息队列最新一条消息开始消费
# 默认情况下, XGROUP CREATE命令默认目标消息队列已经存在,如果不存在则会报错. 也可以使用MKSTEAM子命令在目标消息队列不存在时 创建
127.0.0.1:9001> XGROUP create mq consumGroup 0
OK

# 消息队列mystream不存在时创建
> XGROUP CREATE mystream mygroup $ MKSTREAM

# 消费者A,消费第1条消息
127.0.0.1:9001> XREADGROUP group consumGroup consumerA count 1 streams mq >
1) 1) "mq"
   2) 1) 1) "1652913903153-0"
         2) 1) "msg"
            2) "1"
# 消费者A,消费第2条消息
127.0.0.1:9001> XREADGROUP group consumGroup consumerA count 1 streams mq >
1) 1) "mq"
   2) 1) 1) "1652913903153-1"
         2) 1) "msg"
            2) "2"
# 消费者B,消费第3条消息
127.0.0.1:9001> XREADGROUP group consumGroup consumerB count 1 streams mq >
1) 1) "mq"
   2) 1) 1) "1652913903153-2"
         2) 1) "msg"
            2) "3"
# 消费者A,消费第4条消息
127.0.0.1:9001> XREADGROUP group consumGroup consumerA count 1 streams mq >
1) 1) "mq"
   2) 1) 1) "1652913903153-3"
         2) 1) "msg"
            2) "4"
# 消费者C,消费第5条消息
127.0.0.1:9001> XREADGROUP group consumGroup consumerB count 1 streams mq >
1) 1) "mq"
   2) 1) 1) "1652913903153-4"
         2) 1) "msg"
            2) "5"
```
上面的例子中，三个在同一组 mpGroup 消费者A、B、C在消费消息时（消费者在消费时指定即可，不用预先创建），有着互斥原则，消费方案为`A->1`, `A->2`, `B->3`, `A->4`, `C->5`。语法说明为:  
- `XGROUP CREATE mq consumGroup 0`: 用于在消息队列mq上创建消费组 consumGroup，最后一个参数0，表示该组从第一条消息开始消费。（意义与`XREAD`的0一致）。除了支持`CREATE`外,还支持`SETID`设置起始ID,`DESTROY`销毁组,`DELCONSUMER`删除组内消费者等操作;  
- `XREADGROUP GROUP consumGroup consumerA COUNT 1 STREAMS mq >`,用于组`consumGroup`内消费者`consumerA`在队列mq中消费,  
参数`>`表示未被组内消费的起始消息(messages never delivered to other consumers so far);  
如果我们提供的不是参数`>`而是一个有效的ID,如`0`。那么`XREADGROUP`返回的是group pending的历史消息(let us access our history of pending messages)。也就是哪些已经传递给具体consumer,但还没收到XACK确认的消息。如下:  
参数`count 1`表示获取一条。语法与`XREAD`基本一致，不过是增加了组的概念;  
```cpp
127.0.0.1:9000> xinfo consumers memberMessage consumGroup
1) 1) "name"
   2) "consumerA" //consumerA 没有pending的消息
   3) "pending"
   4) (integer) 0
   5) "idle"
   6) (integer) 189234278
2) 1) "name"
   2) "consumerB"
   3) "pending" //consumerB 有2条没收到ACK的消息
   4) (integer) 2
   5) "idle"
   6) (integer) 5694
// 下面来看下这2条消息的具体ID
127.0.0.1:9000> xpending memberMessage consumGroup - + 10  consumerB
1) 1) "1654399934474-0"
   2) "consumerB"
   3) (integer) 188995577
   4) (integer) 1
2) 1) "1654670729984-0"
   2) "consumerB"
   3) (integer) 241253
   4) (integer) 1
// consumerB 再次从ID为0开始读取,则将再次看到已经处于pending状态的消息
127.0.0.1:9000> xreadgroup group consumGroup consumerB count 1 streams memberMessage 0
1) 1) "memberMessage"
   2) 1) 1) "1654399934474-0" // 这个ID原本就是pending状态
         2) 1) "user"
            2) "zhong"
            3) "msg"
            4) "nihao"
//consumerA 再次从ID为0开始读取,因为 consumerA 没有处于pending状态的消息,所以返回结果为空
127.0.0.1:9000> xreadgroup group consumGroup consumerA count 1 streams memberMessage 0
1) 1) "memberMessage"
   2) (empty list or set)
```
- 消费者组中的consumer不用人为创建,他们会在第一次提到时自动创建;
> Consumers are auto-created the first time they are mentioned, no need for explicit creation.
- 可以进行组内消费的基本原理是,`STREAM`类型会为每个组记录一个最后处理(交付)的消息`ID(last_delivered_id)`，这样在组内消费时，就可以从这个值后面开始读取，保证不重复消费;
- 以上就是消费组的基础操作。除此之外，消费组消费时,还有一个必须要考虑的问题,就是若某个消费者,消费了某条消息,但是并没有处理成功时(例如消费者进程宕机),这条消息可能会丢失,因为组内其他消费者不能再次消费到该消息了(某个消费者程序永远挂掉);  
- `XREADGROUP`是一个写命令,尽管看起来是读取 但他也会对 GROUP的情况造成修改。所以只能在master上执行;  

#### pending 等待列表
为了解决组内消息读取但处理期间消费者崩溃带来的消息丢失问题,`STREAM`设计了`Pending`列表,用于记录读取但并未处理完毕的消息。命令`XPENDIING`用来获消费组或消费内消费者的未处理完毕的消息。演示如下:
```cpp
127.0.0.1:9001> XPENDING mq consumGroup
1) (integer) 5 # 5个已读取但未处理的消息
2) "1652913903153-0" # 起始ID
3) "1652913903153-4" # 结束ID
4) 1) 1) "consumerA" # 消费者A有3个
      2) "3"
   2) 1) "consumerB" # 消费者B有1个
      2) "1"
   3) 1) "consumerC" # 消费者C有1个
      2) "1"

127.0.0.1:9001> XPENDING mq consumGroup - + 10 # 使用 start end count 选项可以获取详细信息
1) 1) "1652913903153-0" # 消息ID
   2) "consumerA" # 消费者
   3) (integer) 1478469 # 从读取到现在经历了1478469ms，IDLE
   4) (integer) 1   #消息被读取了1次，delivery counter
2) 1) "1652913903153-1" 
   2) "consumerA"
   3) (integer) 1460658
   4) (integer) 1
...
5) 1) "1652913903153-4"
   2) "consumerB"
   3) (integer) 1403761
   4) (integer) 1
省略一些内容

127.0.0.1:9001> XPENDING mq consumGroup - + 10 consumerA # 在加上消费者参数，获取具体某个消费者的Pending列表
1) 1) "1652913903153-0"
   2) "consumerA"
   3) (integer) 1631955
   4) (integer) 1
2) 1) "1652913903153-1"
   2) "consumerA"
   3) (integer) 1614144
   4) (integer) 1
...
```
每个pending的消息有4个属性:  
a. 消息ID;  
b. 所属消费者;  
c. IDLE, 已读取时长;  
e. delivery counter,消息被读取次数;  

上面的结果我们可以看到,我们之前读取的消息,都被记录在Pending列表中,说明全部读到的消息都没有处理，仅仅是读取了。那如何表示消费者处理完毕了消息呢？使用命令`XACK`完成告知消息处理完成，演示如下:
```cpp
127.0.0.1:9001> XPENDING mq consumGroup - + 10 consumerA # consumerA有3条已读未处理消息
1) 1) "1652913903153-0"
   2) "consumerA"
   3) (integer) 1631955
   4) (integer) 1
2) 1) "1652913903153-1"
   2) "consumerA"
   3) (integer) 1614144
   4) (integer) 1
3) 1) "1652913903153-3"
   2) "consumerA"
   3) (integer) 1577039
   4) (integer) 1
127.0.0.1:9001> 
127.0.0.1:9001> XACK mq consumGroup 1652913903153-0
(integer) 1
127.0.0.1:9001> XPENDING mq consumGroup
1) (integer) 4
2) "1652913903153-1"
3) "1652913903153-4"
4) 1) 1) "consumerA" #已读未处理的消息,消费者A只剩2条了
      2) "2"
   2) 1) "consumerB"
      2) "1"
```
有了这样一个Pending机制，就意味着在某个消费者读取消息但未处理后，消息是不会丢失的。等待消费者再次上线后，可以读取该Pending列表，就可以继续处理该消息了，保证消息的有序和不丢失。  
此时还有一个问题，就是若某个消费者宕机之后，没有办法再上线了，那么就需要将该消费者Pending的消息，转义给其他的消费者处理，就是消息转移。请继续。

#### 消息转移
消息转移的操作时将某个消息转移到自己的`Pending`列表中。使用语法`XCLAIM`来实现，需要设置组、转移的目标消费者和消息ID，同时需要提供`IDLE`(已被读取时长)，只有超过这个时长，才能被转移。演示如下:
```cpp
# 当前属于consumerA的消息 1652913903153-1,已经有 30983,173ms没被处理了
127.0.0.1:9001> XPENDING mq consumGroup - + 10 
1) 1) "1652913903153-1"
   2) "consumerA"
   3) (integer) 30983173
   4) (integer) 1
...

# 转移超过3600s的消息 1652913903153-1 到consumerB的Pending列表
127.0.0.1:9001> XCLAIM mq consumGroup  consumerB 3600000 1652913903153-1
1) 1) "1652913903153-1"
   2) 1) "msg"
      2) "2"

# 消息 1652913903153-1 已经转移到consumerB的Pending中
127.0.0.1:9001> XPENDING mq consumGroup - + 10 
1) 1) "1652913903153-1"
   2) "consumerB"
   3) (integer) 24017
   4) (integer) 2
...
```
以上代码，完成了一次消息转移。转移除了要指定ID外，还需要指定IDLE，保证是长时间未处理的才被转移。被转移的消息的IDLE会被重置，用以保证不会被重复转移，以为可能会出现将过期的消息同时转移给多个消费者的并发操作，设置了IDLE，则可以避免后面的转移不会成功，因为IDLE不满足条件。例如下面的连续两条转移，第二条不会成功。
```cpp
127.0.0.1:9001> XCLAIM mq consumGroup  consumerB 3600000000 1652913903153-3
127.0.0.1:9001> XCLAIM mq consumGroup  consumerC 3600000000 1652913903153-3
```
这就是消息转移。至此我们使用了一个Pending消息的ID，所属消费者和IDLE的属性，还有一个属性就是消息被读取次数，delivery counter，该属性的作用由于统计消息被读取的次数，包括被转移也算。这个属性主要用在判定是否为错误数据上。  

**检查pending的消息 和 claiming 消息可以由不同的程序来完成。**  

#### 自动化 claim:XAUTOCLAIM
`XAUTOCLAIM`是6.2版本加入的,可以自动化claim一个长时间pending状态的消息。  
语法:  
`XAUTOCLAIM <key> <group> <consumer> <min-idle-time> <start> [COUNT count] [JUSTID]`
下面是示例:  
```cpp
127.0.0.1:9000> xinfo consumers memberMessage consumGroup
1) 1) "name"
   2) "consumerA"
   3) "pending"  //consumerA 没有pending的消息
   4) (integer) 0
   5) "idle"
   6) (integer) 4160318
2) 1) "name"
   2) "consumerB"  //consumerB 有2条pending的消息
   3) "pending"
   4) (integer) 2
   5) "idle"
   6) (integer) 4443354
//consumerA 认领了一条 
//就像XCLAIM命令, XAUTOCAIM 会回复一系列的已经claimed的消息, 但是同时会返回一个 stream ID用于轮训pending的消息. 
127.0.0.1:9000> xautoclaim memberMessage consumGroup consumerA 100000 0-0 count 1
1) "1654399934474-0" //cursor, 我们可以用这个ID 做下一个命令的参数
2) 1) 1) "1654399934474-0"
      2) 1) "user"
         2) "zhong"
         3) "msg"
         4) "nihao"
127.0.0.1:9000> xinfo consumers memberMessage consumGroup
1) 1) "name"
   2) "consumerA"
   3) "pending"
   4) (integer) 1
   5) "idle"
   6) (integer) 6338
2) 1) "name"
   2) "consumerB"
   3) "pending"
   4) (integer) 1
   5) "idle"
   6) (integer) 4538052
127.0.0.1:9000> xack memberMessage consumGroup 1654399934474-0 // 确认已处理
127.0.0.1:9000> xautoclaim memberMessage consumGroup consumerA 100000 1654399934474-0 count 1 // consumerA再处理一条
127.0.0.1:9000> xautoclaim memberMessage consumGroup consumerA 100000 1654670729984-0 count 1 //没有pending的消息啦
1) "0-0"
2) (empty list or set)
```

#### 坏消息问题,Dead Letter,死信问题
正如上面所说，如果某个消息，不能被消费者处理，也就是不能被XACK，这是要长时间处于Pending列表中，即使被反复的转移给各个消费者也是如此。此时该消息的`delivery counter`就会累加（上一节的例子可以看到），当累加到某个我们预设的临界值时，我们就认为是坏消息（也叫死信，DeadLetter，无法投递的消息），由于有了判定条件，我们将坏消息处理掉即可，删除即可。删除一个消息，使用XDEL语法，演示如下：
```cpp
# 删除队列中的消息
127.0.0.1:9001> XDEL mq 1652913903153-1
(integer) 1

# 查看队列中再无此消息
127.0.0.1:9001> XRANGE mq - +
1) 1) "1652913903153-0"
   2) 1) "msg"
      2) "1"
2) 1) "1652913903153-2"
   2) 1) "msg"
      2) "3"
3) 1) "1652913903153-3"
   2) 1) "msg"
      2) "4"
4) 1) "1652913903153-4"
   2) 1) "msg"
      2) "5"

# Pending 队列中依然有这个消息
127.0.0.1:9001> XPENDING mq consumGroup - + 10 
1) 1) "1652913903153-1"
   2) "consumerB"
   3) (integer) 909005
   4) (integer) 2
...

# 只有XACK确认处理后,Pending中的这条消息才会删除
127.0.0.1:9001> XACK mq consumGroup 1652913903153-1
(integer) 1
```
#### 信息监控, XINFO

查看队列信息:
```cpp
127.0.0.1:9001> XINFO stream mq
 1) "length"
 2) (integer) 4
 3) "radix-tree-keys"
 4) (integer) 1
 5) "radix-tree-nodes"
 6) (integer) 2
 7) "groups"
 8) (integer) 1
 9) "last-generated-id"
10) "1652913903153-4"
11) "first-entry"
12) 1) "1652913903153-0"
    2) 1) "msg"
       2) "1"
13) "last-entry"
14) 1) "1652913903153-4"
    2) 1) "msg"
       2) "5"
```

消费组信息:
```cpp
127.0.0.1:9001> XINFO groups mq
1) 1) "name"
   2) "consumGroup"
   3) "consumers"
   4) (integer) 2
   5) "pending"
   6) (integer) 2
   7) "last-delivered-id"
   8) "1652913903153-4"
```

消费组成员信息:
```cpp
127.0.0.1:9001> XINFO consumers mq consumGroup
1) 1) "name"
   2) "consumerA"
   3) "pending"
   4) (integer) 0
   5) "idle"
   6) (integer) 32400855
2) 1) "name"
   2) "consumerB"
   3) "pending"
   4) (integer) 2
   5) "idle"
   6) (integer) 1313844
```

#### 命令概览
|命令|说明| 
|:---|:---| 
|XACK|结束Pending| 
|XADD|生成消息| 
|XCLAIM|消息转移| 
|XDEL|删除消息| 
|XGROUP|消费组管理| 
|XINFO|得到消费组信息| 
|XLEN|消息队列长度| 
|XPENDING|Pending列表| 
|XRANGE|获取消息队列中消息| 
|XREAD|消费消息| 
|XREADGROUP|分组消费消息| 
|XREVRANGE|逆序获取消息队列中消息| 
|XTRIM|消息队列容量|

