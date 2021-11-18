# Redis集群

### Redis集群

#### Cluster相关函数

#### cluster manual failover大致流程

<mark style="color:red;">**在manual failover期间**</mark>, 这里slave会执行几步很重要的步骤:

1. 用户发送 CLUSTER FAILOVER 命令。初始化failover状态, mf\_end 将被设置为我们将终止尝试的毫秒时间;
2. slave发送一条 MFSTART 信息给 master, 请求将客户端暂停(执行两次,manual failover超时时间是 REDIS\_CLUSTER\_MF\_TIMEOUT)。 当master因 manual failover暂停(接收客户端)时,master会被设置上 CLUSTERMSG\_FLAG0\_PAUSED 标志位;
3. slave 等待master 发送标记为 PAUSED的复制偏移量(replication offset);
4. 如果slave接收到来自于master的replication offset, 和自己的offset 相等, mf\_can\_start 设置为 1; clusterHandleSlaveFailover() 将执行正常的failover流程; 不同之处在于投票请求将被修改 强迫master们投票给一个 slave.

从master的视角看,这个问题比较简单: 当接收到一个 PAUSE\_CLIENTS 包, master将设置 mf\_end, mf\_slave设置为发送者. 在manual failover期间, master会更频繁的发送 PING 给这个salve, PING包被用PAUSED标记为标记, 所以slave在接收到 master带 PAUSED标记位的包后,将设置 mf\_master\_offset.

manual failover的目标是在没有数据丢失的情况下，执行快速故障迁移，不会因为主从异步复制而导致数据丢失.

**resetManualFailover()**

* 如果有正在执行中manual failover, 则终止; 同时将所有paused的client 不再 paused(server.clients加入到server.unblocked\_clients中);
* 同时将所有 manual failover相关状态重置: server.cluster->mf\_end = 0; server.cluster->mf\_can\_start = 0; server.cluster->mf\_slave = NULL; server.cluster->mf\_master\_offset = 0;

**clusterNode\* createClusterNode(char \*nodename, int flags)**

目标: 创建一个带有指定flags标记的集群节点;

执行流程:

* 如果nodename == NULL, 那么表示我们是第一次和该node握手; 这里会为该node设置一个随机的nodename. nodename在之后接收到node的第一个PONG回复之后就会被更新;
* 返回会返回新创建的Node, 但不会自动将它添加到当前节点的`cluster->nodes`哈希表中;

**int clusterHandshakeInProgress(char \*ip, int port)**

目的: 如果当前节点已经向`ip`和`port`所指定的节点进行了握手，返回1。(该函数用于防止对同一节点进行多次握手)

执行流程:

* 遍历`server.cluster->nodes`,跳过不是处于handshake`状态的`node\`;
* 剩下的都是正在处于`handshake`状态的节点，如果这些节点的ip和`port` 与 node的相等, 则跳出循环;
* 如果得到的节点非空，则代表`ip`和`port`对应的节点正在进行握手;

**int clusterAddNode(clusterNode \*node)**

目标: 将参数中的node添加到`server.cluster->nodes`哈希表中, 这样接下来当前节点就会创建连接node的Link;

**int clusterStartHandshake(char \*ip, int port)**

目的: 如果还没和指定的地址进行握手，那么进行握手。返回1 表示握手开始；返回 0 并将 errno 设置为一下值来表示意外情况:

* `EAGAIN` 与该地址已经有握手在进行中了;
* `EINVAL` ip 或 port 参数不合法;

执行流程:

* 检查`ip`和`port`是否合法, 不合法则返回0；
* 调用函数`clusterHandshakeInProgres()`检查节点是否已经发送了握手请求，如果是则直接返回;
* 调用`createClusterNode()`创建一个带有`REDIS_NODE_HANDSHAKE|REDIS_NODE_MEET`标记的node(带有一个随机nodename);
* 调用`clusterAddNode()`将节点添加到`server.cluster->nodes`中;

**void clusterBuildMessageHdr(clusterMsg \*hdr, int type)**

目标: 构建clusterMsg; 执行流程:

* 如果当前节点是一个master节点, 我们将其 slots bitmap和配置纪元(configEpoch)信息包含在 clusterMsg中; 如果当前节点是一个slave节点, 我们将发送其master的信息(slots bitmap、configEpoch等),当前节点被标记为slave, 所以接受者知道并不真实负责这些slots;
*   大致内容:

    > clusterMsg->sig\[4] = "RCmb";\
    > clusterMsg->type 由传入参数决定;\
    > clusterMsg->sender = myself->name;\
    > clusterMsg->port = server->port;\
    > clusterMsg->flags = myself->flags;\
    > clusterMsg->state = server.cluster->state; 集群的状态(上线还是下线);\
    > clusterMsg->currentEpoch = server.cluster-> currentEpoch; 集群配置纪元\
    > clusterMsg->offset = 如果是slave,则是slave复制偏移; 如果是master,则是master\_repl\_offset(master每执行一次更新操作,offset都会加上相关的值);\
    > 如果当前节点是master节点,且正在manual failover中.则 clusterMsg->mflags\[0] |= CLUSTERMSG\_FLAG0\_PAUSED; 代表当前master节点处于 PAUSED\_CLIENT状态;\
    > clusterMsg->totlen 如果TYPE是 `CLUSTERMSG_TYPE_FAIL`|`CLUSTERMSG_TYPE_UPDATE`,则该函数会自己计算;
    >
    > 如果TYPE是**PING** **PONG** **MEET**, 则由调用者自己计算` clusterMsg->totlen`;
    >
    > clusterMsg->myslots = master->slots;\
    > clusterMsg->slaveof = master->name;\
    > clusterMsg->configEpoch = master->configEpoch;

**void clusterBroadcastMessage(void \*buf, size\_t len)**

目标: 向有连接的所有其他节点发送message。

执行流程:

* 遍历`server.cluster->nodes`, 排除`node->link==NULL` 和 本节点、`REDIS_NODE_HANDSHAKE`状态的节点;
* 调用`clusterSendMessage()` 发送信息;

**void clusterSendFail(char\* nodename)**

目标: 向本节点已知的所有节点发送 `FAIL`消息(`server.cluster->nodes`);

本节点探测到`nodename`下线后(`REDIS_NODE_PFAIL`)发送。同时我们也得到足够数量节点的 `nodename`已下线的支持。那么本节点就会将 `nodename`标记为`FAIL`(其实也就是markNodeAsFailingIfNeeded()函数所做的工作)。

并执行这个函数，向其他node发送`FAIL`消息，要求它们也将nodename标记为`FAIL`。

执行流程:

* 调用`type=CLUSTERMSG_TYPE_FAIL`的`clusterMessage`;
* `clusterMessage`中包含`nodename`;
* 调用`clusterBroadcastMessage()`广播消息;

**void clusterWriteHandler(aeEventLoop \*el, int fd, void \*privdata, int mask)**

目标: 写事件处理器, 向集群节点发送信息;

执行流程:

* `void *pridata` 转换为 `clusterLink`;
* 将`link->sndbuf`中的内容写入到 fd中;
* 从`link->sndbuf`中删除已写入部分;
* 如果`link->sndbuf`中所有内容都已写入完毕(缓冲区为空) , 那么删除`link->fd`相关的写事件处理器;

**void clusterReadHandler(aeEventLoop \*el, int fd, void \*privdata, int mask)**

目标: 读事件处理器, 尝试读取header的第一个field(8字节), 进而检查数据包(data packet)的大小. 当整个数据包(a whole packet)在内存中，该函数将调用函数处理数据包.

执行流程:

* `void *pridata` 转换为 `cluserLink`;
* `link->rcvbuf`代表已经读取到的数据，如果`link->rcvbuf`中已读取的数据8个字节都没有，那就优先从fd中读取8个字节。(**注意: clusterMsg的前8字节包含: char sig\[4]四个字节 + uint32\_t totlen 四个字节**)
* 已读前8字节,我们从这8字节中可以得到clusterMsg的真实大小(`clusterMsg->totlen`),后续将读入完整的`clusterMsg`;
* 如果`clusterMsg`过大，一个`buf`装不下，则每次最多只能读取`sizeof(buf)`大小的内容;
* 当读取的数量`rcvbuflen` 等于 `clusterMsg->totlen`大小时，调用`clusterProcessPacket()` 处理读取到的数据;
* 释放`clusterMsg->rcvbuf`的空间;

**int clusterNodeAddFailureReport(clusterNode \*failing, clusterNode \*sender)**

目标: 该函数会在 当前节点 接收到 来自某个节点(`sender`)的下线报告时调用。

执行流程:

* `failing->fail_reports`指向下线报告的链表;
* 如果`sender`在下线报告中, 则只更新该报告的时间戳;
* 否则,创建一个新的报告, 并添加到`failing->fail_repots`中;

**void clusterNodeCleanupFailureReports(clusterNode \*node)**

目标: 移除node节点太旧的`failure reports`,太旧意味着比global node timeout还老( 多长时间为过期是根据node timeout 选项的值来决定的)。

注意: 将node标记为`FAIL`状态，我们都需要本地`PFAIL state`至少比global node timeout更旧(older)。因此我们不仅仅想想来自于其他节点的`failure reports`。

执行流程:

* `failure report`的最大保质期: `maxtime= server.cluster_node_timeout * REDIS_CLUSTER_FAIL_REPORT_VALIDTIY_MULT`;
  * `REDIS_CLUSTER_FAIL_REPORT_VALIDTIY_MULT` 值为2;
* 遍历`node->fail_reports`,删除`now - item->time > maxtime`的元素;

**int clusterNodeDelFailureReport(clusterNode \*node, clusterNode \*sender)**

目标: 从node的下线报告(`failure_reports`)中移除`sender`对node的下线报告;

该函数在以下情况下使用: 本节点仍为node已下线(`FAIL`或`PFAIL`)，但`sender`却向本节点发来报告，说它认为node节点没有下线。

那么当前节点就要移除`sender`对node的下线报告——如果`sender`曾报告过node下线的话。

即使 在节点没有下线的情况下，这个函数也会被调用，并且调用的次数还比较频繁。在一般情况下，这个函数的复杂度为`O(N)`。不过在不存在下线报告的情况下，这个函数复杂度仅为常数。

执行流程:

* 遍历`node->fail_reports`，找到`sender`节点的下线报告;
* 如果没找到，直接返回;
* 如果找到，则从node->`fail_reports`中删除。
* 同时调用`clusterNodeCleanupFailureReports()`删除`node->failure_reports`中的过期报告;

**int clusterNodeFailureReportsCount(clusterNode \*node)**

目标: 计算不包括本节点在内的，将node标记为`PFAIL` 或 `FAIL`的节点的数量(`node->fail_reports`的长度);

执行流程:

* 调用`clusterNodeCleanupFailureReports(node)` 从`node->fail_reports`中移除过期的元素;
* 返回`node->fail_reports`的长度;

**void markNodeAsFailingIfNeeded(clusterNode \*node)**

目的: 该函数用于判断是否需要将node标记为 FAIL;

注释: 将node标记为FAIL需要满足以下两个条件.

* 通过gossip接收到半数以上的master 将该node添加到`failure reports`中。也就是`PFAIL`状态;
* 当前节点也将node标记为`PFAIL`状态;

如果确认Node已进入FAIL状态，那么当前节点还会向其他节点发送`FAIL`消息，让其他节点也将node标记为`FAIL`。

注意：集群判断一个node进入`FAIL`所需的条件是`weak`的，因为节点对node的状态报告不是实时的，而是经过一段时间间隔(这段时间内node的状态可能已经发生改变)。

并且尽管当前节点会向其他节点发送`FAIL`消息，但是因为网络分裂(net partition)的问题，有一部分node可能还是不会将node标记为`FAIL`。

不过:

* 只要我们成功将 node 标记为 FAIL，那么这个`FAIL`状态最终总会传播至整个集群的所有节点;
* 抑或，因为半数节点的支持，当前节点不能够将node标记为`FAIL`，所以对 `FAIL`节点的故障迁移将无法进行，`fail`标识可能会在之后被移除;

执行流程:

* 标记(参数)node为`FAIL`所需的节点数量: `server.cluster->size/2 +1`, `server.cluster->size`代表集群中至少负责一个slot的master的个数;
* 如果当前node还没将 参数node 标记为 PFAIL 状态 或 参数node已被标记为 `FAIL`状态，则直接返回;
* 调用`failures=clusterNodeFailureReportsCount()` ，统计将参数node标记为`PFAIL`或`FAIL`状态的 节点数量, 也就是返回`node->fail_reports`的长度;
* 如果本node是一个master节点，在`failures++`。因为本node已经将参数node标记为`PFAIL`;
* 如果`failures`小于节点总数的一半，则不能判断节点为`FAIL`。直接返回;
* 否则，将node标记为`REDIS_NODE_FAIL`,更新`node->fail_time`为当前时间。
* 如果本node是master的话，那么向其他node发送参数node的`FAIL`信息(`clusterSendFail()`)，让其他node将参数node标记为`FAIL`;

**void clusterProcessGossipSection(clusterMsg \*hdr, clusterLink \*link)**

目的: 处理MEET、PING 或 PONG消息中`gossip`协议相关的信息。

确定gossip section中的node是否被sender标记为PFAIL`或`FAIL`, 如果`FAIL`了，则广播；如果node没被sender标记为PFAIL/FAIL，说明node状态正常，那么更新`node->failure\_repots`; 如果 node是一个新节点，则调用`clusterStartHandshake()`与其进行没收，并将其保存到`server.cluster->nodes\`中.

注意：这个函数假设调用者已经根据消息长度，对消息进行过合法的检查。

参数`clusterLink *link`保存着`sender`的信息，`link->rcvbuf`保存着发送过来的`clusterMsg`，其实就是参数`hdr`;

执行流程:

* 遍历`hdr->data.ping.gossip`, `hdr->data.ping.gossip`有多少个node元素，就代表该clusterMsg中包含了多少个node的信息;
* 打印node的一些`GOSSIP`信息;
* 从`server.cluster->nodes`中查找node:
  * 如果找到了，则:
    * 如果sender是master，且node不是本节点，则确定sender是否将node标记为PFAIL`或`FAIL\`状态:
      *   如果sender将node标记为`PFAIL`或`FAIL`状态，则调用`clusterNodeAddFailureReport()`函数，添加 sender 对 node 的下线报告(将sender保存在node->fail\_reports链表中,代表sender报告Node已下线);

          继续调用`markNodeAsFailingIfNeeded()`尝试将node标记为`FAIL`状态也就是已下线状态，如果标记成功，则广播已下线信息;
      * 如果sender没将node标记为`PFAIL`或`FAIL`状态，则调用`clusterNodeDelFailureReport()` : 如果 sender 曾经发送过对 node 的下线报告，此时 清除该报告(也就是从node->failure\_reports中删除sender的下线报告);
    * 如果node之前处于 PFAIL 或者 FAIL 状态，并且该node的 IP 或者port已经发生变化，那么可能是节点换了新地址，调用`clusterStartHandshake()`对它进行握手;
  * 如果没找到，代表本节点不认识node:
    * 如果 node 不是 NOADDR 状态，并且node不在 server.cluster->nodes\_black\_list 黑名单中，那么调用`clusterStartHandshake()`对它进行握手(新建带`REDIS_NODE_HANDSHAKE|REDIS_NODE_MEET`标记的node，并将其加入`server.cluster->nodes`中);

**void clusterSendMessage(clusterLink \*link, unsigned char \*msg, size\_t msglen)**

目标: 将msg放入link的发送缓冲区中, 并给 link->fd 绑定写事件处理函数(clusterWriteHandler);

执行流程:

* 为 `link->fd` 绑定写事件处理函数(clusterWriteHandler());
* 将msg信息追加到`link->sndbuf`中;
* `server.cluster->status_bus_messages_sent++`;

**void clusterSendPing(clusterLink \*link, int type)**

目的: 向指定节点发送一条 MEET 、PING 或 PONG消息(消息中`clusterMsg->data.ping.gossip`部分会携带2个其他node的信息一并发送给receiver)。

执行流程:

* 初始化变量`int freshnodes=dictSize(server.cluster->nodes)-2`: freshnodes 表示在ping packet中, 我们能在 gossip section中添加的node的计数器。freshnodes 初始值是server.cluster->nodes 中的节点数量减去 2 (`server.cluster->nodes -2`), 这里的 2 指两个节点，一个是 myself 节点（也即是发送信息的这个节点),另一个是接受 gossip 信息的节点。每将一个node添加(到gossip section)，程序将 freshnodes 的值减一, 当 freshnodes 的数值小于等于 0 时，我们知道没有更多gossip 信息需要我们发送了。
* 调用`clusterBuildMessageHdr(hdr,type)`构建一个type类型的`clusterMsg`: clusterMsg中将携带本node的`configEpoch`、`slots bitmap`等很多信息;
* 循环两次，每次从`server.cluster->nodes`中随机选择一个node:
  * 如果node 是如下情况之一，则直接跳过(同时freshnodes-- ):
    * node == myself: node是节点本身;
    * node处于`HANDSHAKE`状态;
    * node带有`NOADDR`标识;
    * node部分负责任何slot;
  * 将node的信息(`ping_sent`、`pong_received`、`ip`、`flags`等)添加到`hdr->data.ping.gossip[i]`中;
*   重新计算`clusterMsg`信息的长度:

    ```
    totlen = sizeof(clusterMsg)-sizeof(union clusterMsgData);
    totlen += (sizeof(clusterMsgDataGossip)*gossipcount);
    ```
* 调用`clusterSendMessage(link,buf,totlen)` 向`link->fd`所代表的node发送消息;

**void clusterSendMFStart(clusterNode \*node)**

目标: 向给定node发送一条**MFSTART**消息. 该消息目标是: 在slave上执行`cluster failover`命令后, slave向master发送`MFSTART`消息,代表`manual failover`开始, master进入`PAUSED_CLIENT`状态(不再接受client的请求)? 同时 master 更频繁的向slave 发送 PING命令，以便将自己的`master_repl_offset`发送给slave。slave以便与自己的`repl_offset`进行对比，进而执行 `manual failover`;

执行流程:

* 如果`node->link`为空, 则直接返回;
* 调用`clusterBuildMessageHdr()`构建一个`type=CLUSTERMSG_TYPE_MFSTART`消息;
* 调用`clusterSendMessage(node->link,buf,totlen)` 发送;

**int nodeUpdateAddressIfNeeded(clusterNode \*node, clusterLink \*link, int port)**

目的: 更新node节点的地址(IP和port)，IP和端口可以从 `link->fd`中获得。断开`node->link`，并根据新的地址创建新连接。

如果ip 和 端口和现在的连接相同，不执行任何动作。

函数返回0表示地址不变，地址已被更新则返回1。

执行流程:

* 如果`link == node->link`，直接返回;
* 获取字符串格式的ip地址 和 端口号 并与 `node->ip`、`node->port`比较，没有变化返回 0;
* 释放旧的`node->link`, 新连接会在之后clusterCron中自动建立;
* 如果node是本node(本node为slave)的master, 那么根据最新地址 设置复制对象;

**void clusterRenameNode(clusterNode \*node, char \*newname)**

目标: 第一次向节点发送cluster meet命令时，因为发送命令的节点还不知道目标节点的名字，所以它会给目标节点分配一个随机的名字。

当目标节点向发送节点返回PONG回复时，发送节点就知道了目标节点的IP 和 port，此时 发送节点就可以通过调用这个函数为目标节点改名。

执行流程:

* 根据`node->name`，先将其从`server.cluster->nodes`中删除;
* 将`newname` 拷贝给 `node->name`;
* 再调用`clusterAddNode()` 将 `node`加入到`server.cluster->nodes`中: `node->name`是key, `node`是value;

**void clearNodeFailureIfNeeded(clusterNode \*node)**

目的: 该函数在 本节点 接收到 **一个被标记为 FAIL 状态** 的节点那里接收到消息时使用。该函数将确认是否能将该node的 FAIL 状态移除。

执行流程:

* 如果node不是`FAIL`状态，直接返回;
* 如果node是slave 或 node不负责任何slots(`node->numslots==0`), 那么直接移除该node的`FAIL`标记;
* 如果同时满足下面条件也可以直接清理 node的 FAIL 标记:
  * node是master;
  * 从本节点的视角，node负责处理的slots不为空(`node->numslots>0`), 也就意味着还没执行 `failover`;
* 此时说明 node节点仍然有slot没迁移完，那么当前节点移除node的FAIL标识;

**void clusterUpdateSlotsConfigWith(clusterNode \*sender, uint64\_t senderConfigEpoch, unsigned char \*slots)**

目的: 该函数在节点通过 PING、PONG、UPDATE消息接收到一个 master 的配置时调用，函数以a node、a configEpoch of the node、以及节点在 configEpoch 纪元下的slot作为参数。

(注意三个参数都属于 sender)

该函数要做的就是: 将参数slosts最新配置 和 本节点slots进行对比，并更新本节点的slots布局。如果有需要，函数还会将本节点转换成 sender 的slave。

'sender'可以是我们接收到的 "最新配置" 的发送者，有时 他可能不是消息的'sender'，如我们接收到一个'UPDATE'消息时。

执行流程:

*   `dirty_count` 变量是一个slot列表, 这些slot中仍然存在key的情况下，我们丢失了这些slot的所有权。这通常发生在`failover`之后 或 管理员将集群重新配置后;

    如果一个update消息不能将master降级为slave，那么我们需要删除丢失所有权的slot中的所有key;

    如果将master降级为slave，那么我们将与new master重新同步，更新整个key空间。
* `curmaster` 代表
  * 如果当前节点是master，那么将`curmaster`设置为master;
  * 如果当前节点是slave, 那么将 `curmaster`设置为当前节点那正在复制的master节点;
  * 稍后在for循环中 我们将使用 `curmaster`检查与当前节点有关的slot是否发生了变动。
* 如果`sender` 其实就是当前节点，则直接返回(`sender`==`myself`)，忽略自己的Update消息;
* for循环所有slot，看这些slot是否发生改变(`for (j = 0; j < REDIS_CLUSTER_SLOTS; j++) `)
  * 如果 参数slots中包含 slot\[j], 代表 slot\[j] 已经被 sender接管，那么执行如下代码:
    * 如果本地记录 slot\[j] 就是 sender 在负责，则continue: `server.cluster->slots[j]==sender`;
    * slot\[j] 在本地依然处于正在`importing`状态，则continue: 应该正通过`redis-trib`对`slot[j]` 进行手动修改，正在进行迁移，migrating 的node 的slot已经关闭且正在发布配置，但我们这里依然希望(migrating 的node)通过手动关闭该slot，而不是程序自动完成;
    *   以下情况下，我们将slot重新绑定到他的新节点(sender)上:

        * slot未被分配(`server.cluster->slots[j]`==NULL) 或 新的node使用更大的configEpoch 声明了该slot (`server.cluster->slots[j]->configEpoch < senderConfigEpoch`);
        * 我们当前没有在`importing` slot;

        如果slot\[j]原本归属与我们 且 slot中包含了 key，那么将其标记为 `dirty_slot`

        如果以前负责slot\[j]的是curmaster，(但是现在是sender)，这说明故障迁移了，所以当前节点将复制 new master(想一想一主多备 manual failover的场景);

        调用`clusterAddSlot(sender,j)`将slot\[j]指派给sender;
* 如果当前节点（或者当前节点的主节点）有至少一个槽被指派到了 sender，且 sender 的 configEpoch 比当前节点的configEpoch要大，那么可能发生了:
  * 当前节点是一个不再处理任何槽的master, 这意味着我们failed over(故障迁移了,如manual failover后, old master的状态); 这时应该将当前节点设置为new master(sender)的slave。
  * 当前节点是一个slave, 并且当前节点的curmaster已经不再处理任何槽,这时应该将当前节点设置为new master的从节点;
  * 反正: newmaster 不为NULL， 且curmaster->numslots==0, 那么就将new(master) 设置为当前节点的master;
* 如果 `dirty_slots_count >0`
  *   我们收到一个`update message`, 该message 删除了我们对某些slot的所有权, 这些slot中我们还有key在里面;

      (但是我们依然持有一些slot的所有权此时该master 不会 降级为 slave); 为了在key 和 slots之间保持一致性状态, 我们需要将 时区所有权的slot中的key 删除;

**void clusterSendFailoverAuthIfNeeded(clusterNode \*node, clusterMsg \*request)**

目的: 条件满足的情况下，为请求进行故障转移的node进行投票，支持它进行failover.

执行流程:

* 如果本节点是slave，或 是一个没有处理任何slot的master，则没权限投票;
* request->currentEpoch 必须大于等于 `server.cluster->currentEpoch`, 否则直接返回;
* 如果本节点已经投过票了，不继续投票; `server.cluster->lastVoteEpoch == server.cluster.currentEpoch`;
* 参数node必须是`slave` 且 其master必须是`FAIL`状态 或者 request中使用 `CLUSTERMSG_FLAG0_FORCEACK标记`标记(force manual failover)，则master可以处于非`FAIL`状态;
* 本节点在 `2*node_timeout` 时间内不会再次为该slave(也就是参数node)投票。这对算法的正确性不是必须的，但是会使得基本情况更加线性;
* 遍历所有slots:`for (j = 0; j < REDIS_CLUSTER_SLOTS; j++)`
  * 跳过 request 没有负责的slot;
  * request负责的slot\[j]，本节点中记录着其应该是nodeA负责(`nodeA=server.cluster->slots[j]`)，那么`nodeA->configEpoch` 必须小于 `requestConfigEpoch`;
  * 如果 `nodeA->configEpoch` 大于 `requestConfigEpoch`，那么说明`request的configEpoch`已经过期，直接返回;
* 调用`clusterSendFailoverAuth(node)` 函数为 `node`投票: 向node发送`CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK`类型的消息;
* `node->slaveof->voted_time`更新为当前时间;

**`Gossip`几种消息类型**

```
// 注意，PING 、 PONG 和 MEET 实际上是同一种消息，都是 clusterMsg - clusterMsgData + clusterMsgDataGossip
// PONG 是对 PING 的回复，它的实际格式也为 PING 消息，
// 而 MEET 则是一种特殊的 PING 消息，用于强制消息的接收者将消息的发送者添加到集群中
// （如果节点尚未在节点列表中的话）
// PING
#define CLUSTERMSG_TYPE_PING 0          /* Ping */
// PONG （回复 PING）
#define CLUSTERMSG_TYPE_PONG 1          /* Pong (reply to Ping) */
// 请求将某个节点添加到集群中
#define CLUSTERMSG_TYPE_MEET 2          /* Meet "let's join" message */
// 将某个节点标记为 FAIL, 类型是: clusterMsg - clusterMsgData + clusterMsgDataFail
#define CLUSTERMSG_TYPE_FAIL 3          /* Mark node xxx as failing */
// 通过发布与订阅功能广播消息，类型是: clusterMsg - clusterMsgData + clusterMsgDataPublish + channel_len + message_len
#define CLUSTERMSG_TYPE_PUBLISH 4       /* Pub/Sub Publish propagation */

// CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK CLUSTERMSG_TYPE_MFSTART 几种消息类型一样，都是 clusterMsg - clusterMsgData
// 请求进行故障转移操作，要求消息的接收者通过投票来支持消息的发送者
#define CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST 5 /* May I failover? */
// 消息的接收者同意向消息的发送者投票
#define CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK 6     /* Yes, you have my vote */

// CLUSTERMSG_TYPE_UPDATE 类型的消息，长度: clusterMsg - clusterMsgData + clusterMsgDataUpdate
// 槽布局已经发生变化，消息发送者要求消息接收者进行相应的更新
#define CLUSTERMSG_TYPE_UPDATE 7        /* Another node slots configuration */
// 为了进行手动故障转移，暂停各个客户端
#define CLUSTERMSG_TYPE_MFSTART 8       /* Pause clients for manual failover */
```

**int clusterProcessPacket(clusterLink \*link)**

目的: 当这个函数被调用时，说明`node->rcvbuf`中有一条待处理的信息。信息处理完毕后，buffer的释放工作由调用者完成，所以这个函数只需要负责处理信息即可。

* 如果函数返回1, 那么说明处理信息时没有遇到问题，连接依然可用;
* 如果函数返回0，那么说明信息处理时遇到问题(比如PONG消息的sender ID不对)

执行流程:

* 如果消息长度小于 16字节，返回 1;
* 如果`totlen 大于 sdslen(link->rcvbuf)`, 返回 1;
* 如果消息类型是 `CLUSTERMSG_TYPE_PING` 或者 `CLUSTERMSG_TYPE_PONG` 或者 `CLUSTERMSG_TYPE_MEET`
  * 消息的长度: `explen=sizeof(clusterMsg)-sizeof(union clusterMsgData) + sizeof(clusterMsgDataGossip)*count `;
* 如果消息类型是 `CLUSTERMSG_TYPE_FAIL`
  * 消息长度: `explen=sizeof(clusterMsg)-sizeof(union clusterMsgData) + sizeof(clusterMsgDataFail)`
* 如果消息类型是 `CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST` `CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK` `CLUSTERMSG_TYPE_MFSTART`
  * 消息长度: `explen=sizeof(clusterMsg)-sizeof(union clusterMsgData)`
*   如果消息类型是 `CLUSTERMSG_TYPE_UPDATE`

    * 消息长度:`explen=sizeof(clusterMsg)-sizeof(union clusterMsgData) + sizeof(clusterMsgDataUpdate)`

    如果消息`explen`不等于`totlen`, 然后1;
* 查找消息发送者, 本地是否已经记录了该node: `sender = clusterLookupNode(hdr->sender)`， `sender`就代表本node中记录的 消息发送者node 的信息
  * 如果本地 已记录 消息发送者node， 同时该node不是 `HANDSHAKE`状态
    * 如果发送者的`senderCurrentEpoch > server.cluster->currentEpoch`, 则 `server.cluster->currentEpoch=senderCurrentEpoch`;
    * 如果发送者的`senderConfigEpoch > sender->configEpoch`, 则 `sender->configEpoch= senderConfigEpoch`;
    * 更新`sender->repl_offset = hdr->offset`; 更新`sender->repl_offset_time`为当前时间;
      * `repl_offset`在node是master时代表master的复制偏移量，在node是slave时代表复制master的复制偏移量;
    * 如果我们是一个slave, 正在执行manual failover(cluster->mf\_end不为0, mf\_end代表manual failover的最长超时时间), 我们的master就是sender; master已经处于paused\_client状态, 当前将其`rerpl_offset`发送给我们。
      * `server.cluster->mf_master_offset = sender->repl_offset`;
* 如果消息类型是 `CLUSTERMSG_TYPE_PING` 或者 `CLUSTERMSG_TYPE_MEET`
  * 如果我们是第一次遇到消息发送者(`sender==NULL`)，且消息类型是 MEET
    *   新建一个`HANDSHAKE`状态的node，从接收到的消息中得到发送者的ip + port 复制给 node, 将node添加到 `server.clsuter->nodes`中;

        注意我们不会拷贝 消息发送者的`flags`、`slaveof`等信息，等当前节点(clusterCron函数)向对方发送 `PING` 并接受到对方 `PONG`回复后，我们从`PONG`回复中得到;
    * 调用`clusterProcessGossipSection(hdr,link)`分析消息中`gossip section`部分的信息: 处理这些node是否`FAIL`、`PFAIL`等flags，如果本地从没记录过这些node，则调`clusterStartHandshake`尝试与他们进行握手，并将其添加到`server.cluster->nodes`中;
    * 调用 `clusterSendPing()`给sender回复一个`PONG`消息;
*   如果消息类型是 `CLUSTERMSG_TYPE_PING` 或者 `CLUSTERMSG_TYPE_PONG` 或者 `CLUSTERMSG_TYPE_MEET`

    * 如果`link->node`不为空，`link`中保存了发送消息者的所有信息，node、flag、ip、port等
      *   如果`link->node`处于`HANDSHAKE`状态

          (一个典型的场景: nodeB上执行 cluster meet nodeA, 此时nodeB会与nodeA握手，并且在nodeB中, nodeA就是处于HANDSHAKE状态, nodeA收到MEET消息后，将向nodeB回复PONG消息，nodeB根据PONG消息中的内容跟新自己记录的nodeA的相关信息)

          *   如果本地`server.cluster->nodes`中记录了发送者的信息（`sender!=NULL`）, 则利用`link`中的信息，更新`sender`的ip port 等。返回0。

              我能想到的就是上面说的: NodeB 与NodeA建立连接后，将NodeA记录为 HANDSHAKE状态，NodeA返回PONG回复之后，在NodeB中执行的就是这一部分吧;
          *   如果本地`server.cluster->nodes`中没有记录发送者的信息(sender == NULL), 关闭`link->node->flags`的`HANDSHAKE`状态，并确定`link->node`是`MASTER`还是`slave`

              我能想到的就是上面说的: NodeB 与 Node建立联系，朝NodeA发送了一个 MEET消息，此时nodeA的`server.cluster->nodes`中肯定没有NodeB。

              所以NodeB第一次MEET NodeA，NodeA就为 `link->node`去掉flags的 HANDSHAKE状态了？？
      * 如果link->node不是处于HANDSHAKE`状态，且`link->node->name`和`hdr->sender\`不相等，此时代表本地link中记录的发送者id 与 实际发送者id不相等，断开link返回 0;
    * 如果`sender!=NULL`且sender部署处于`handshake`状态，发过来的是PING消息， 调用`nodeUpdateAddressIfNeeded()` 更新sender的ip port,如果ip port被更新，则断开`node->link`，等着下次重连;
    * 如果`link->node!=NULL`且发过来的是PONG消息，则更新`link->node`的PONG接收时间`pong_received`、清零最近一次等待ping命令的时间`ping_sent`;
      * 如果`link->node`处于`PFAIL`状态，则将`PFAIL`状态取消;
      * 如果`link->node`处于`FAIL`状态，则调用`clearNodeFailureIfNeeded()`看`link->node`的`FAIL`状态是否能取消;
    * 如果`sender!=NULL`,我们这里讲更新他的角色信息:`master->slave or slave -> master`
      * 如果`hdr->slaveof`为`REDIS_NODE_NULL_NAME`,说明sender是一个master，调用`clusterSetNodeAsMaster(sender)`将其设置为master;
      * 否则，说明`sender`是一个slave，下面我们就要看他的master信息是否更新了
        * `hdr->slaveof`中就是最新的master信息；
        * 如果以往`sender`是一个master，现在`hdr->slaveof`不为空，代表sender有了master，此时我们删除sender负责的所有slot信息，并将sender设置为slave; 下一步配置`sender->slaveof`;
        * 如果`sender->slaveof`不等于`hdr->slaveof`
          * 如果`sender->slaveof`不为空，则调用`clusterNodeRemoveSlave()`从旧的`sender->slaveof`这个master中将`sender`这个slave移除;
          * 更新`sender->slaveof`为`hdr->slaveof`;
    * 更新当前节点对`sender`所处理`slot`的认识，这里必须在上一步更新`sender`的master/slave信息后，因为这里用到了`REDIS_NODE_MASTER`标识:
      * 将`sender_master->slot`(sender是master就是`sender->slot`), 和 `hdr->myslot`做对比，两者是否一致记录为`dirty_slots`;
      * 如果`sender`是一个master，且sender的slot信息出现了变动, 则调用 `clusterUpdateSlotsConfigWith(sender,senderConfigEpoch,hdr->myslots)`函数: 该函数的目的就是 将参数slosts最新配置 和 本节点slots进行对比，并更新本节点的slots布局。如果有需要，函数还会将本节点转换成 sender 的slave;
      * 如果 sender说 slot\[j] 是自己负责, 实际上我们知道有另一个节点 configEpoch比sender的大, 同时也说自己负责处理 slot\[j], 此时我们就需要调用`clusterSendUpdate()`告诉 sender;
    * 如果我们的configEpoch 和 sender的configEpoch 相等, 那么尝试调用`clusterHandleConfigEpochCollision(sender)`解决他:

    ```
    server.cluster->currentEpoch++;
    myself->configEpoch = server.cluster->currentEpoch;
    ```

    * 调用`clusterProcessGossipSection(hdr,link)`提取并分析`gossip`中的信息: 确定gossip section中的node是否被sender标记为PFAIL`或`FAIL`, 如果`FAIL`了，则广播；如果node没被sender标记为PFAIL/FAIL，说明node状态正常，那么更新`node->failure\_repots`; 如果 node是一个新节点，则调用`clusterStartHandshake()`与其进行没收，并将其保存到`server.cluster->nodes\`中;
* 如果消息类型是`CLUSTERMSG_TYPE_FAIL`: sender告诉当前节点，某个节点进入了`FAIL`状态;
* 如果消息类型是`CLUSTERMSG_TYPE_PUBLISH`: 这是一条publish消息，调用`pubsubPublishMessage()`处理;
* 如果消息类型是 `CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST`: sender 请求当前节点为它进行failover投票，通过调用`clusterSendFailoverAuthIfNeeded(sender,hdr)`完成，如果投票完成，就向`sender`回复`CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK`类型的消息;
*   如果消息类型是 `CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK`: sender 支持当前节点执行故障迁移操作，只有符合一下以下几个条件，sender的投票才有效:

    * sender是主节点;
    * sender至少负责一个slot;
    * sender的currentEpoch 大于 当前节点的 currentEpoch (`server.cluster->failover_auth_epoch`);

    满足条件后，`server.cluster->failover_auth_count++`;
* 如果消息类型是`CLUSTERMSG_TYPE_MFSTART`：这条消息只可能来自于本节点自己的slave, 请求自己这个master 暂停客户端 的请求, 要执行 `manual failover`了
  * 此时本节点(master), 调用`resetManualFailover()`重置failover的各种状态;
  * 记录manual failover的超时时间: `server.cluster->mf_end = mstime() + REDIS_CLUSTER_MF_TIMEOUT`;
  * 记录manual failover是哪个slave发起的: `server.cluster->mf_slave = sender`;
  * 本节点进入`PAUSED_CLIENT`状态，让服务器在指定的时间内(当前时间+`REDIS_CLUSTER_MF_TIMEOUT*2`)不再接受被客户端发来的请求(slave正常);
* 如果消息类型是`CLUSTERMSG_TYPE_UPDATE`: sender告诉本节点slot信息已经发生变化，消息发送者要求接受者进行相应的更新
  * 消息中的`MsgUpdate->configEpoch`必须大于`node->configEpoch`(node是根据`MsgUpdate->nodename`从`server.cluster->nodes`中找到的);
  * 如果node在本节点中以前记录为一个slave，此时(发送slot信息)代表其已变成一个master，所以将node角色切换成master;
  * 调用`clusterUpdateSlotsConfigWith(node,reportedConfigEpoch,hdr->...slots)`更新slot布局;

**void clusterUpdateState(void)**

目标: 更新节点状态(`server.cluster->state`)

执行流程:

* 如果这是一个master节点, 在将其状态转为OK前等待一段时间(`REDIS_CLUSTER_WRITABLE_DELAY=2s`)。因为在一个node重启后，不给cluster重新配置这个node的机会，就将这个可写的master加入到cluster中，并不明智;
* 检查是否所有slot都已经有某个节点负责了，也就是该集群所有slot都覆盖了。如果没有被覆盖，则`new_sate=REDIS_CLUSTER_FAIL`;
* 统计在线且正在处理(至少)一个slot的master的数量(`server.cluster->size`); 以及下线`master`的数量(`unreachable_masters`,这些master也负责着相关slot);
* 如果`unreachable_masters`超过半数，那么将我们自己的状态设置为`new_state=REDIS_CLUSTER_FAIL`;
* 如果`new_state != server.cluster->state`
  * 如果当前节点是一个master 同时已经和集群分隔变成了少部分(partitioned away),当前集群状态为`FAIL`;
  * 那么在重连上cluster后(partition heals) 一段时间内, 不要让它接收请求(不要将其状态变成OK), 以确保足够的时间接收配置更新;
  * 否则，更新节点状态为 `new_state`;

**void clusterHandleSlaveFailover(void)**

目的: 执行 slave failover, 如果当前节点是一个slave节点，其master负责非零个slot 但处于 `FAIL`状态，那么执行这个函数.

该函数有三个目标:

1. 检查是否可以对master执行一次failover, 节点的关于master的信息是否准确和最新(updated);
2. 选举一个 new master;
3. 执行failover, 并通知其他节点;

执行流程:

* 无论是automatic 还是 manual failover检查必须同时满足下面几个条件，否则返回:
  * 我们是一个slave;
  * 我们的master被标记为`FAIL` 或 这是一个 `manual_failover`;
  * 我们的master负责了一部分slot;
*   获取本节点和master的断开秒数:`data_age`

    如果`data_aget > server.repl_ping_slave_period * 1000 + server.cluster_node_timeout * REDIS_CLUSTER_SLAVE_VALIDITY_MULT `, 则直接返回;

    * `repl_ping_slave_period` 代表 master给slave发送PING的频率;
    * `REDIS_CLUSTER_SLAVE_VALIDITY_MULT`当前是10;
* 如果上一次`failover`超时 或 重试时间已过，我们会发起一个新的:
  * 下一次`failover`的开始时间是: `mstime()+ 500ms + random() % 500ms + failover_auth_rank * 1000ms `;
  * `server.cluster->failover_auth_rank`: 当前slave的排名(rank), 也就是在相同的master下，和其他slave相比，当前slave的排名，排名由每个slave的`replication offset`决定。`rank=0`的slave replication offset更大。注意,可能多个slave具有相同的rank，这是因为他们的offset 相同;
  * 如果`failover`是`manual failover`, 则不需要发起，也就是说起开始时间是:`mstime()`, 其`failover_auth_rank=0`。 (但并不意味着马上会执行`failover`，而是在`clusterCron()`下一次执行时会发起)
  * 广播一条PONG消息给所有slave(和本节点一个master的slave)， 以便获得他们的offset，进行排名;
  * 返回;
* 如果不是`manual failover`且还没向其他节点发送投票请求，则计算slave的排名(`new rank`)，排名有变化，则更新响应的`failover`开始时间;
* 如果当前时间 还没到 `failover`开始时间，则直接返回(我们本地还没到`failover`开始时间，但不代表其他slave没到`failover`开始时间，所以排名靠前的slave会优先执行`failover`);
* 如果已经过了故障迁移的超时时间，直接返回;
* 向其他节点发送failover请求:
  * 增加本节点的配置纪元, `server.cluster->currentEpoch++`;
  * 记录发起failover的配置纪元:`server.cluster->failover_auth_epoch = server.cluster->currentEpoch`;
  * 向其他所有节点广播`CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST`消息，寻求其他节点投票支持本节点执行`failover`;
  * 打开`failover`已开始标识:`server.cluster->failover_auth_sent = 1`;
  * `return`;
* 如果当前节点获得足够多的投票，那么对FAIL状态的master进行failover:
  * 当前节点身份由slave变成master，函数: `clusterSetNodeAsMaster()`, 将自己从master的`slaves`列表中移除，本节点`node->flags`添加`REDIS_NODE_MASTER`;
  * 让slave去掉复制，成为新的master，函数: `replicationUnsetMaster()`，`free` master客户端、`backlog`等等;
  * 接手所有master负责的slot: `server.cluster->slots`、`node->slots`;
  * 更新当前节点配置纪元 为 投票时的配置纪元:` myself->configEpoch = server.cluster->failover_auth_epoch`;
  * 调用`clusterUpdateState()`更新节点状态;
  * 调用`clusterSaveConfigOrDie()` 保存配置文件;
  * 向所有节点发送`PONG`消息，让其他人知道当前节点已经升级为master 并负责相应slots;
  * 调用`resetManualFailover()`重置manual failover相关状态(`mf_end`、`mf_slave`等)，关闭`PAUSED CLIENT`状态, 继续接受客户端的请求;

**void clusterCron(void)**

目的: 周期函数，默认每秒执行10次(每隔100毫秒执行一次);

执行流程:

* 遍历所有nodes, `server.cluster->nodes`:
  * 如果node是本节点自身(`REDIS_NODE_MYSELF`) 或 没有地址(`REDIS_NODE_NOADDR`)的node, 则继续;
  * 如果node处于`HANDSHAKE`状态时间超过`handshake_timeout`(一般和`node_timeout`相等)，则直接剔除该node; `continue`;
  * 如果`node->link==NULL`, 则为该未创建连接的节点创建连接(如本节点\`cluster meet $some\_node):
    * 调用`anetTcpNonBlockBindConnect()`、`createClusterLink()`等函数创建连接;
    * 向该节点发送`MEET/PING`消息，如果处于`MEET`状态则发送`MEET`消息，如果处于`PING`状态则发送`PING`消息;
    *   node去除`REDIS_NODE_MEET`标记(该节点后续可能处于`HANDSHAKE`状态)

        **如果当前节点(发送者) 没能收到`MEET`信息的`PONG`回复，那么它将不再向目标节点发送命令(在函数`clusterSendPing()`中实现, `clusterSednPing()`会跳过处于`NOADDR/HANDSHAKE`状态的节点);**

        如果当前节点(发送者)收到MEET信息的PONG回复，那么节点将不再处于`MEET/HANDSHAKE`状态，将继续向目标节点发送普通的`PING`命令;
* `clusterCron()`函数每轮询10次(至少间隔一秒)， 就随机选择一个node向其发送`GOSSIP`信息
  * 随机遍历 5 个node(跳过`node->link==MULL`、`MYSELF`、`HANDSHAKE`)，选出其中 oldest `pong_received` time的节点(接收到它的`PONG`回复时间距离当前最久)， 向其发送`PING`消息;
* 遍历所有nodes :`server.cluster->nodes`:
  * 依然跳过`MYSELF|NOADDR|HANDSHAKE`状态的节点;
  * 获得`orphaned_master`、`max_slaves`、`this_slaves`等信息:
    * `orphaned_master` : master负责slots, 但是其没有状态ok的slave，这类master的个数;
    * `max_slaves`: 集群中master拥有状态ok的`slave`的最大个数，这里说的是单个master;
    * `this_slaves`: 如果我们是slave, 我们的master具有状态ok的slave的个数;
  * 如果(`PING`消息已经发出),等到 `PONG` 回复的时间 超过了 node timeout 的一半,释放该`node->link`,等待下一次`clusterCron()`自动重连: 因为尽管node依然正常，但`link`可能已经出问题，所以重连一下;
  * **如果我们目前对该node所有`PING`都已收到回复(当前没有`PING`已发出但没收到回复), 同时最后一次收到来自该node PONG回复时间已经超过 `node_timeout/2`；那么向node继续发送一个`PING`，确保所有node不会延迟太久才被`PING`到**;
  *   如果当前节点是一个主节点，node是其slave, node请求进行`manual failover`( `server.cluster->mf_end` 就代表接收到了slave的manual failover请求), 那么向node发送 PING 回复，并continue;

      (这遵循了处于 manual failover状态的master 更频繁的向slave发送 PING 消息的原则)
  * 如果向节点发送`PING`消息，等待`PONG`回复的时间超过`node_timeout`，那么将node标记为`REDIS_NODE_PFAIL`;
* 如果我们是一个slave节点,但是复制关系当前依然是关闭状态。如果我们已知master的地址,此时我们将重新开启复制;
* 如果`manual failover`超时，则终止他: `manualFailoverCheckTimeout()`;
* 如果当前节点是slave:
  *   判断manual failover 是否可以开始了(如果ok则`server.cluster->mf_can_start=1`)

      最主要的判断: `server.cluster->mf_master_offset == server.master->reploff`;
  * 调用`clusterHandleSlaveFailover()` 执行`failover`操作;
* 调用:`clusterUpdateState()`更新集群状态;

**一些关键时间的计算**

* failover超时时间: `auth_timeout=server.cluster_node_timeout*2`, 最少 2s;
* failover重试时间: `auth_retrytime=auth_timeout*2`(超时时间的两倍);
* handshake超时时间: `handshake_timeout=server.cluster_node_timeout`, 最少 1秒;
* slave进入`manual failover`的超时时间(`mf_end`): `mstime() + REDIS_CLUSTER_MF_TIMEOUT`, `REDIS_CLUSTER_MF_TIMEOUT`在Redis 3.0的版本中值为 5秒;
* master进入`manual failover`的超时时间(`mf_end`): `mstime()+REDIS_CLUSTER_MF_TIMEOUT*2`, `REDIS_CLUSTER_MF_TIMEOUT`在Redis 3.0的版本中值为 5秒;
* 本节点将一个node标记为`PFAIL`的时间: `server.cluster_node_timeout`;

**阅读源码中的问题:**

1.  `server.cluster->nodes[j]`和`link->node`到底有啥区别? 特别是`clusterProcessPacket()`函数中一会儿检测的是`link->node!=NULL` 一会儿是`sender!=NULL`，搞不懂;

    比如NodeAA 给 NodeBB 发送MEET消息，NodeBB新接收到这个消息时，`link->node`是否是NULL;

    答:

    > 什么时候 link->node == NULL, 什么时候 link->node != NULL?
    >
    > `clusterProcessPacket()`只会被`clusterReadHandler()`函数调用，而在`clusterReadHandler()`中`link`也是通过`void *privdata`参数转换而来, 所以`link->node`是否为NULL不是`clusterReadHandler()`决定的，`clusterReadHandler()`函数只是简单的从`link->fd`中读取内容到`link->rcvbuf`而已;
    >
    > `clusterReadHandler()`函数主要有两个地方调用:
    >
    > * `clusterAcceptHandler()`，该函数在cluster启动时绑定相关`fd`，主要用于接收cluster的连接请求。在该函数中, `link->node`始终为NULL，而后`clusterReadHandler()`函数可以从`link->fd`中读取内容填充`link->rcvbuf`; 继而`clusterProcessPacket()`可以从`link->fd`中读取发送者 ip 和 port;
    > * `clusterCron()`, 该函数被cluster node定期调用，传入`link->node`不是`NULL`。
    >
    > 根据上面这两个函数的调用情况，我们可以得出 "NodeAA 给 NodeBB 发送MEET消息" 的大致执行流程。
2. **NodeAA 给 NodeBB 发送MEET消息 的大致执行流程 是啥(也是一个集群新增一个节点流程)**？
   *   NodeAA上执行`cluster meet NodeBB_ip NodeBB_port`

       NodeAA将调用`clusterStartHandshake()`: 检查ip port的合法性，并创建一个nodeBB(nodename随机)保存ip port，并将该nodeBB保存到 `server.cluster->nodes`中，注意: 此时nodeBB被标记为`REDIS_NODE_HANDSHAKE|REDIS_NODE_MEET`，同时`node->link==NULL`;
   * NodeAA在`clusterCron()`函数中，遍历`server.cluster->nodes`,刚好遍历到nodeBB:
     *   此时`nodeBB->link==NULL`，则根据nodeBB `ip port`等创建link => `nodeBB->link!=NULL`。

         且`nodeBB->fd`读取事件和`clusterReadHandler()`绑定，也就是说 `nodeBB->fd有数据可读，则server调用`clusterReadHandler()\`处理;

         NodeAA调用`clusterSendPing()`向NodeBB发送`MEET`消息;

         `nodeBB->flags`去掉`REDIS_NODE_MEET`标记，`nodeBB->flags`等于`REDIS_NODE_HANDSHAKE`;
   * NodeBB接收到来自NodeAA的`MEET`消息，调用` clusterAcceptHandler()`处理，在该函数中`link->node==NULL`;
     * `clusterAcceptHandler()`继续调用`clusterReadHandler()`处理，`clusterReadHandler()`函数简单地从`link->fd`中读取内容到`link->rcvbuf`，`clusterReadHandler()`继续调用`clusterProcessPacket()`处理;
     * `clusterProcessPacket()`中，NodeBB因为是第一次收到MEET消息:
       * `server.cluster->nodes`根据nodename找不到发送者nodeAA，因此`sender==NULL`;
       * 同时`link->node`也是NULL;
       * nodeBB只能新建一个nodeAA(nodename随机)，从`link->fd`中得到`nodeAA_ip`、`nodeAA_port`保存到`nodeAA`中，并将nodeAA添加`server.cluster->nodes`中。
       * nodeBB继续调用`clusterProcessGossipSection()`函数解析`gossip section`的内容，因为
         * `gossip section`中的node对于nodeBB来说依然是一个也不认识(`node=clusterLookupNode(g->nodename),node==NULL`);
         * 同时nodeBB也不知道发送者是谁: `link->node==NULL` 同时 `clusterLookupNode(hdr->sender)==NULL`;
         * 所以`clusterProcessGossipSection()`什么也做不了;
       * nodeBB向nodeAA回复`PONG`消息(注意PONG消息中带有nodeBB真实的nodename);
   * NodeAA接收到来自NodeBB的`PONG`消息，并调用`clusterProcessPacket()`处理:
     * 此时NodeAA虽然保存了nodeBB节点信息，但是`nodeBB->name`还是随机生成的，所以`sender=NULL`;
     * 然后调用`clusterRenameNode(link->node, hdr->sender)`将发送者(NodeBB)的name赋值给`link->node->name`,当然 `server.cluster->nodes`中的信息也会被更新;
     * NodeAA去掉NodeBB的`REDIS_NODE_HANDSHAKE`标记， 且确定NodeBB是master还是slave。
     * 继续更新NodeBB的`pong_received`、`ping_sent`等信息;
     * 至此，NodeAA 中 NodeBB属于一个普通正常的节点了，NodeAA向NodeBB握手完成;
   * NodeBB中此时记录的nodeAA，依然是`HANDSHAKE`，而后在NodeBB的`clusterCron()`函数中，将重复上面nodeAA的步骤:
     *   根据NodeAA `ip port`等创建link => `nodeAA->link!=NULL`。

         且`nodeAA->fd`读取事件和`clusterReadHandler()`绑定，也就是说 `nodeAA->fd`有数据可读，则server调用`clusterReadHandler()`处理;

         NodeBB调用`clusterSendPing()`向NodeAA发送`MEET`消息;

         NodeAA回复NodeBB PONG消息后，NodeBB将调用`clusterProcessPacket()`处理，并从中得到发送者(NodeAA)的name等信息，去掉本地NodeAA的 `HANDSHAKE`标记。至此，NodeBB向NodeAA也握手完成;
3. **`manual failover`的执行流程**
   * 用户在`redis_slave`上执行:`cluster failover [force]`命令;
     * 如果当前节点不是slave，返回错误;
     * 如果当前节点的master为空或处于FAIL状态(`myself->slaveof==NULL`、`nodeFailed(myself->slaveof)`),则必须处于带上`force`参数;
     * 重置`manual failover`有关的状态属性，如`server.cluster->mf_end`等;
     *   设置`manual failover`的最大执行时间(超过这个时间失败 or 终止): `server.cluster->mf_end= mstime() + REDIS_CLUSTER_MF_TIMEOUT`;

         `REDIS_CLUSTER_MF_TIMEOUT`当前是 5秒;
     * 如果是强制manual failover(`force==true`), 那么设置`mf_can_start=1`, 代表可以立即开始 failover;
     * 如果不是强制manual failover(`force==true`), 那么向本节点的`master`发送`CLUSTERMSG_TYPE_MFSTART`消息;
     * 回复客户端;
   * master接收到`CLUSTERMSG_TYPE_MFSTART`消息后:
     * 如果消息发送者`sender`的master不是我，则直接返回;
     * 设置`manual failover`的最大执行时间(超过这个时间失败 or 终止): `server.cluster->mf_end= mstime() + REDIS_CLUSTER_MF_TIMEOUT`;
     * 记录请求`manual failover`的节点: `server.cluster->mf_slave=sender`;
     *   调用`pauseClients()`让master节点进入`PAUSED_CLIENT`状态:`server.clients_paused = 1`;

         让master在指定时间内(`mstime()+REDIS_CLUSTER_MF_TIMEOUT*2`)不再接收客户端的请求(slave的请求正常);
     * 后续`master`在`clusterCron()`函数中向slave发送`PING`消息的过程中, `clusterBuildMessageHdr()`函数将执行:
       * `PING`消息将带有`CLUSTERMSG_FLAG0_PAUSED`标识: `clusterMsg->mflags[0] |= CLUSTERMSG_FLAG0_PAUSED`;
       * 同时带有自己的`offset`: `clusterMsg->offset=server.master_repl_offset`;
   * slave接收到master的`PING`消息后:
     * 更新自己本地的`master->repl_offset`: `sender->repl_offset=hdr->offset`;
     *   如果slave 处于`manual failover`状态，其发送过来的`clusterMsg`带有`CLUSTERMSG_FLAG0_PAUSED`标识

         则更新本节点的`mf_master_offset`: `server.cluster->mf_master_offset= sender->repl_offset`;
     * 设置`clusterDoBeforeSleep(CLUSTER_TODO_HANDLE_MANUALFAILOVER)`
   * slave在`clusterBeforeSleep()`函数中:
     * 调用`clusterHandleManualFailover()`确定`manual failover`是否可以开始了, 如果可以开始了: `server.cluster->mf_can_start=1`;
     * 调用`clusterHandleSlaveFailover()`执行`failover`:
       * 做一些前置检查，如本节点是一个slave，我们的master被标记为`FAIL` 或 这是一个`manual failover`;
       *   确定本节点在左右slave中的排名，进而确定本节点`failover`的开始时间，如果开始时间没到，则直接返回。

           (我们本地还没到`failover`开始时间，但不代表其他slave没到`failover`开始时间，所以排名靠前的slave会优先执行`failover`);
       * 如果是`manual failover`，上一步不存在，会马上开始`failover`;
       * 增加本节点的配置纪元;
       * 向其他所有节点广播`CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST`消息，寻求其他节点投票支持本节点执行`failover`;
       * 等待下一次`clusterCron()`再次调用`clusterHandleSlaveFailover()`;
     * 下一次`clusterCron()`调用`clusterHandleSlaveFailover()`:
       * 如果本节点获得了足够多的选票，则:
         * 当前节点身份由slave变成master,让本节点去掉复制，成为新的master;
         * 接手所有master负责的slot;
         * 更新当前节点配置纪元 为 投票时的配置纪元;
         * 向所有节点发送`PONG`消息，让其他人知道当前节点已经升级为master 并负责相应slots;
4.  **故障检测流程**

    比如具有8节点的集群，包含nodeA、nodeB、nodeC..., 其中nodeB故障

    * 集群中其他节点在`clusterCron()`中都会不断发送PING消息给 nodeB，而nodeB 超过`server.cluster_node_timeout`时间没有回复，其他节点将把`nodeB`标记为`REDIS_NODE_PFAIL`:`node->flags |= REDIS_NODE_PFAIL`
      * 通过每个节点本地的`nodeB->ping_sent`实现
      * `nodeB->ping_sent`: 代表本节点最后一次发送PING消息给`nodeB`的时间，如果本节点接收到nodeB的PONG回复，则将`nodeB->ping_sent=0`;
      * `nodeB->pong_received`: 代表本节点最后一次从nodeB接收到`PONG`回复的时间;
    * 比如此时`nodeA`将`nodeB`标记为`PFAIL`状态，那么`nodeA`向其他节点发送PING消息的`gossip section`中，如果选到`nodeB`就会带上`nodeB`的`flags`信息:`gissip->flags=this->flags`;
    * 比如此时`nodeC`收到来自于`nodeA`的PING消息，PING消息`gossip section`中带有`nodeB->flags==REDIS_NODE_PFAIL`
      * 调用`clusterProcessGossipSection()` 处理PING消息的`gossip section`部分;
      *   则添加sender(nodeA)对nodeB的下线报告(将nodeA保存在`nodeB->fail_reports`链表中，代表nodeA报告nodeB已下线);

          调用`clusterNodeAddFailureReport(node,sender)`完成;
      * 调用`markNodeAsFailingIfNeed(nodeB)`尝试将nodeB标记为 FAIL: **如果我们检测到超过半数的其他节点将nodeB都标记为`PFAIL`, 那么我们将把`nodeB`标记为`FAIL`。同时广播`nodeB`进入`FAIL`状态这个消息**;
    * **其他节点接收到`nodeC`发送过来的`nodeB`进入`CLUSTERMSG_TYPE_FAIL`类型的消息后，纷纷将其标记为`CLUSTERMSG_TYPE_FAIL`，并记录下nodeC进入`CLUSTERMSG_TYPE_FAIL`的时间**`nodeC->fail_time=mstime()`;
    * 上面的`CLUSTERMSG_TYPE_FAIL`消息`nodeB`的slave也会收到，比如`slave01_of_nodeB`收到消息后，将记录本地`master->flags=CLUSTERMSG_TYPE_FAIL`
      * 在`slave01_of_nodeB`的`clusterCron()`函数中，只要本节点是slave，都会执行`clusterHandleSlaveFailover()`函数:
        * 做一些前置检查: 我们是一个slave，我们的master被标记为`FAIL`，我们的master负责了一部分`slots`，检查主从断开时间是否超过`10*node_timeout`;
        *   确定本节点在左右slave中的排名，进而确定本节点`failover`的开始时间，如果开始时间没到，则直接返回。

            (我们本地还没到`failover`开始时间，但不代表其他slave没到`failover`开始时间，所以排名靠前的slave会优先执行`failover`);
        * 如果达到`failover`的开始时间，则:
          * 增加本节点的配置纪元;
          * 向其他所有节点广播`CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST`消息，寻求其他节点投票支持本节点执行`failover`;
          * 等待下一次`clusterCron()`再次调用`clusterHandleSlaveFailover()`;
        * 下一次`clusterCron()`调用`clusterHandleSlaveFailover()`:
          * 如果本节点获得了足够多的选票，则:
          * 当前节点身份由slave变成master,让本节点去掉复制，成为新的master;
          * 接手所有master负责的slot: 调用`clusterAddSlot()`、`clusterDelSlot()`,`server.cluster->slots、node->slots、node->numslots`都会被更新;
          * 更新当前节点配置纪元 为 投票时的配置纪元;
          * 向所有节点发送`PONG`消息，让其他人知道当前节点已经升级为master 并负责相应slots;
    * 此时nodeA、nodeC接收到来自于`slave01_of_nodeB`的PONG消息，调用`clusterProcessPacket()`处理
      * 如nodeA发现本地`server.cluster->nodes`中`slave01_of_nodeB`还是一个slave，但发送过来的消息中`hdr->slaveof`为空，则更新本地关于`slave01_of_nodeB`的记录，将其更新为一个maseter;
      * 对比`sender_master->slots`和`hdr->myslots`是否有变化，如果有变化且`sender`是一个master，则调用`clusterUpdateSlotsConfigWith()`处理:
        * 更新本地`sender`的slots信息: `server.cluster->slots、node->slots、node->numslots`都会被更新;
    * 经过上面的过程，`nodeB`负责的slots全部迁移为`slave01_of_nodeB`负责，此时`nodeB->numslots==0`:
      * 而后，`nodB`处于`CLUSTEER_NODE_FAIL`状态，`nodeB->numslots==0`, `nodeB`虽然在集群每个节点的`server.cluster->nodes`中有记录，但是并不计入`server.cluster->size`中;
      * 通过函数`clusterSendFailoverAuthIfNeeded()`可以知道, 如果节点是`slave`或`node->numslots==0`, 则该节点不参与投票;
      * 如果`nodeB` 在`cluster_node_timeout/2`时间内没有回复`PONG`消息，则我们将会把`nodeB->link`释放。并在下一次`clusterCron()`中继续尝试与其建立连接;
    * 如果FAIL nodeB在一段时间后复活:
      * 集群其他节点在`clusterCron()`中与其重新建立连接成功:`node->link!=NULL`， 并向其发送一个PING消息;
      * FAIL nodeB接收到`PING`消息:
        * 更新`server.cluster->currentEpoch`、`sender->configEpoch`。注意FAIL nodeB自己的configEpoch还是旧的;
        * 向发送者回复`PONG`消息, `PONG`消息中将携带上`nodeB->ip`、`nodeB->port`、`node->flags`、`node->configEpoch`、`nodeB->myslots`等信息;
        * 比如此时`nodeA`接收到`FAIL nodeB`回复的PONG消息，调用`clusterProcessPacket()`处理:
          * 去掉`FAIL nodeB`的`REDIS_NODE_PFAIL`标识;
          *   PONG消息中`FAIL nodeB`宣称对`slot[j]`负责的，但实际上这部分slot已经是`slave01_of_nodeB`负责

              且`PONG`消息中`PFAIL nodeB`的configEpoch 小于 `slaveof_01_nodeB`的configEpoch;

              所以这个`PONG`消息不会产生任何作用;
          * nodeA将调用`clusterSendUpdate()`向`FAIL nodeB`发送`CLUSTERMSG_TYPE_UPDATE`消息。更新`FAIL nodeB`的slot信息;
        * `FAIL nodeB`接收到来自于其他节点的更新slots的信息后，调用`clusterUpdateSlotsConfigWith()`处理:
          * 更新本地slots情况，此时`FAIL nodeB`不再负责任何slots;
          * 将`FAIL nodeB`做某个master node的slave，哪个 master node呢？这个node就是接手`FAIL nodeB`负责对应`slot[j]`的节点，在我们这里就是节点`slave01_of_nodeB`;
5. 如果产生脑裂, 节点数少的那一部分如何确保自己不再接收client的命令？
   * 这一部分内容在函数`clusterUpdateState()`中完成:
     *   遍历`server.cluster->nodes`, 确定`unreachable_masters`的个数，如果`unreachable_master`的个数超过`server.cluster->size/2+1`, 则代表产生了脑裂，且我们是脑裂后节点数较少的那一部分。

         而后将`server.cluster->state`置为`REDIS_CLUSTER_FAIL`, 该节点不再接受任何请求;
   * `clusterCron()`每执行一次几乎都会执行`clusterUpdateState()`;
6. node挂掉，被探测到并广播，这个行为难道是在`clusterProcessGossipSection()`中完成的？
7. failover后，将old master变成 new master的slave，难道就是在`clusterUpdateSlotsConfigWith()`函数中完成的?
8. `server.cluster->importing_slots_from`、`server.cluster->migrating_slots_to`这两个dict在迁移完成后到底会不会被清空，什么时候清空？
9. 避免重复投票啥的，应该是在`clusterSendFailoverAuthIfNeeded()`完成的;
10. 什么情况下会调用`clusterDelNode()`删除node?

    a. 调用`cluster reset`命令，那么执行该命令的node将调用`clusterDelNode()`删除所有`server.cluster->nodes`;

    b. `clusterProcessPacket()`中如果`link->node`处于`HANDSHAKE`状态，同时又找到了`sender`， 则删除node;

    c. `clusterCron()` 总如果某个节点处于`handshake`状态超过`handshake_timeout`，则不会删除;

    d. 调用`cluster forget`命令, 会将该node加入到`server.cluster->blacklist`中，同时`clusterDelNode()`删除该node;
11. Redis cluster中`slave`是否参与投票?
    * 通过函数`clusterSendFailoverAuthIfNeeded()`可以知道, 如果节点是`slave`或`node->numslots==0`, 则该节点不参与投票;
    * 但是通过函数`clusterRequestFailoverAuth()`可以知道，请求投票的`CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST`类型消息会发送给所有`node->link!=NULL`的节点，包括slave、包括`node->numslots==0`的正常节点;
    * 同时在`slave`接收到`CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK`类型的消息，如果消息发送者是一个slave，也会忽略这张票；
    * `server.cluster->size`的计算也只包含 负责slot的master节点个数;
12. Redis cluster中nodeB被标记为`FAIL`状态，并已完成`failover`; 但是nodeB并不会在`server.cluster->nodes`中被删除。

    集群中nodeA等节点还会在`cronCluster()`中不断重连: `free(nodeB->link)`, 又不断尝试`create(nodeB->link)`，这个过程永不停歇。

    现在考虑一种场景: `2.2.2.2#30000`以前是clusterA的节点，后来因为机器故障挂了，这过程中已经完成了failover。机器修复ok后，我们在`2.2.2.2`上重新上了一个`30000`的redis实例，也是`cluster_abled yes`。但我这里的目的是新建一个集群，而不是将`2.2.2.2#30000`加入到以前的集群中。

    此时，nodeA、nodeC等老集群的节点还会尝试与`2.2.2.2#30000`建立连接，建立连接成功后，则发送`PING`消息，看代码里`2.2.2.2#30000`似乎还会接受nodeB的给 nodeA返回`PONG`消息，并尝试与其建立连接。
13. nodeA 执行cluster meet nodeB\_IP nodeB\_Port，两者建立关系后，集群已有的节点是怎么知道nodeA这个新节点的?
    * nodeB会将nodeA的信息保存在 `server.cluster->nodes`中，且在`clusterCron()`中会给集群其他节点发送PING消息。PING消息的gossip 部分会带上nodeA的信息;
    * 比如集群中的nodeC收到nodeB的PING消息后，会更新本地对nodeB的认识，同时处理gossip部分，函数:`clusterProcessGossipSection()`;
    * `clusterProcessGossipSection()`发现gossip中保存的node自己不认识，就会调用`createClusterNode()`为这个node创建一个节点，并保存到`server.cluster->nodes`中。这个node->link=NULL;
    * 后续`nodeB`的`clusterCron()`会尝试与其建立连接;

#### 改造rediscluster，添加只投票，不负责数据的节点

* clusterSendFailoverAuthIfNeeded() 必须修改;
* 新加入的节点要算到 `server.cluster->size`中吗? 这个参数甚至影响了投票时 `needed_quorum`的判断;

### Redis 普通函数与机制

1.  服务器中的数据库:`struct redisServer`

    普通情况下，我们redisServer有16个数据库(通过`databases 16`参数控制)，Redis服务器将所有数据库都保存在`struct redisServer`结构的`db`数组中。

    `db`数组的每个项都是一个`struct redisDb`结构，每个`redisDb`结构代表一个数据库:

    ```c
    struct redisServer {
    	// 数据库
      redisDb *db;
      // 数据库数量
      int dbnum;
      ...
    }
    struct redisDb {
      // 数据库键空间，保存着数据库中的所有键值对
      dict *dict;
      // 键的过期时间, 字典的键为键，字典的值为过期事件 UNIX 时间戳
      dict *expires;
    }
    ```

    ![image-20200921113513758](Users/lukexwang/Library/Application%20Support/typora-user-images/image-20200921113513758.png)
