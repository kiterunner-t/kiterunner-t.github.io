
    kiterunner_t
    TO THE HAPPY FEW


## 1 集群环境搭建
按照官方推荐的测试集群拓扑，在测试单机上搭建一个最小集群，3个主节点，3个从节点，每个主节点一个从节点。拓扑如下图所示。

![redis-cluster-topology][1]


### 1.1 redis-trib.rb
搭建该集群可以使用代码中redis-trib.rb工具。首先以下列配置文件为模板创建6个以集群模式启动的独立节点（需要修改端口号以及日志文件名）。

    port 7001
    cluster-enabled yes
    cluster-config-file nodes.conf
    cluster-node-timeout 5000
    appendonly yes
    daemonize yes
    logfile cluster.7001.log


启动集群模式配置的单个节点命令是

    redis-server redis-cluster.7001.conf


然后使用redis-trib.rb下列命令激活集群

    ruby redis-trib.rb create --replicas 1 127.0.0.1:7001 127.0.0.1:7003 127.0.0.1:7005 127.0.0.1:7002 127.0.0.1:7004 127.0.0.1:7006


### 1.2 create-cluster-test.sh
使用redis-trib.rb工具可以方便的搭建集群，但对于想要分析集群实现来说就不是那么友好了。为了分析集群，使用redis-cli作为客户端，通过shell搭建了同样的集群，脚本参考这里[https://github.com/kiterunner-t/krt/blob/master/t/redis/create-cluster-test.sh][2]。

使用该脚本，首先会在脚本当前目录下创建分别为7001-7006的6个子文件夹，每个文件夹作为集群一个节点的运行目录，目录下会自动生成节点配置文件、集群配置文件（由redis动态生成和修改，以反映集群的当前拓扑）、日志文件、aof和rdb文件等。

创建好集群节点后，通过以下步骤就可以搭建好上图所示的集群拓扑。

* 为主节点分配槽位（shell脚本中槽位分配与redis-trib.rb中不同）。
* 为每个节点设置configEpoch。
* 将节点加入集群。
* 设置主从复制。

参考命令如下所示，详细步骤参见脚本：

    redis-cli -p 7001 cluster addslots 0 1
    redis-cli -p 7001 cluster set-config-epoch 1
    redis-cli -p 7002 cluster meet 127.0.0.1 7001
    redis-cli -p 7002 cluster replicate <nodeid-7001>


## 2 集群实现
在图示的集群拓扑中，每个节点中都有一个节点实例对象clusterNode来表示该节点，以端口号为7001的节点为例，其内部实现的数据结构如下图所示（注意，图中字典分配按端口为序显示，并不表示真实字典的布局如此）。

![redis-cluster][3]


通过上述数据结构，集群实现了以下基本功能：

* 集群节点管理（节点自动发现），集群节点迁移
* gossip消息分发
* 容错（选举、从变主、自动容错、手动容错）
* 数据分片，每个slot负责一部分键
* slot迁移，进和出

初始化完成后，集群管理可以通过以下渠道来改变集群的状态：

* clusterCron，集群之间进行ping、节点加入等
* server.port+4000，消息处理
* 集群管理命令

集群配置文件格式，每行表示一个节点的信息，最后一行为epoch。每个节点的信息如下所示：

    <name> <ip>:<port> <flags> <slaveof> <ping-sent-time> <pong-received-time> <config-epoch> <connect-status> [<slot-list>] [<slot-migrate>]
    vars currentEpoch <epoch> lastVoteEpoch <epoch>


注意以下几点：

* 状态之间以逗号分隔，没有flag时为noflags，noflags,myself,masterslave,fail?,handshake,noaddr。
* 没有slaveof时为-。
* 当前节点服务的slot：单节点直接用槽位号表示，连续槽位号使用\<start-slot>-\<end-slot>，不能服务时为空。
* 槽位迁移的信息，配置文件中只会为当前节点生成槽位迁移的信息

      to:   [slotnum->-nodename]
      from: [slotnum-<-nodename]


### 2.1 节点管理
如上图所示，集群节点使用clusterState来表示当前节点对整个集群的认识，并通过这些数据结构来管理整个集群。集群中每个节点都通过一个clusterNode实例来表示该节点，我称之为节点实例，表示了在整个集群中

nodes_black_list字典中存储了节点名字、该节点可以重新加入集群的时间。主要防止当cluster forget命令从在某个节点中被删除后，但又收到来自其他节点（没有对该节点执行forget操作）的gossip消息，于是又重新被加入到当前节点的集群数据结构中。

clusterNode->fail_reports记录有哪些节点报告了该节点实例失败。

节点握手，节点之间的连接分为两种情况：

* clusterCron中会主动去连接对方，此时会在node->link下创建一个link结构体。
* clusterAcceptHandler中会被动的接受对方连接，此时创建一个孤立的link，但是还没有注意到在什么地方将该link附加到结构上。

监听端口服务端口+4000（不能超过65535），来自其他节点的事件回调函数为clusterAcceptHandler，处理tcp accept事件；accept后，设置读网络事件回调函数为clusterReadHandler。

在clusterAcceptHandler中，会为每个连接分配一个clusterLink实例，但此时并不知道该连接对应的节点实例，直到其他节点的数据到来时，通过消息才会知道。消息处理的主体函数是clusterProcessPacket。

节点类型

* SELF
* MASTER
* SLAVE

节点状态

* PFAIL
* FAIL
* HADNSHAKE
* NOADDR

### 2.2 消息
Redis集群的消息是通过二进制的结构体进行传递的，虽然定义了9种类型的消息，实际上只有4种消息体，消息的结构如下图所示。

![redis-cluster-msg][4]


集群消息的封装结构 clusterMsg，共有9种消息类型：

* PING
* PONG
* MEET
* FAIL
* PUBLISH
* FAILOVER_AUTH_REQUEST
* FAILOVER_AUTH_ACK
* UPDATE
* MFSTART

ping消息（pong和meet内容上与ping一致，消息类型不一样）由2部分组成，一是消息头构成的节点自身状态信息，当节点是从节点时，会将其复制节点负责的槽位信息放在消息中；二是随机选择的最多3个其他活动节点实例的信息（当前节点所感知到的），包括节点名字、ip/port、ping/pong发送接收时间、节点状态等。

publish消息包含节点自身状态和需要发布的频道和消息。

fail消息还包括当前节点感知到的失败的节点名字。某个节点发现一个节点失效的时候，该消息就会在集群中进行广播。

update消息用于关注的节点的槽位发生了变化，需要通知其他节点。因此，消息中主要信息是变化的节点名字、configEpoch（用于标识消息的是否是最新的）、槽位位图。

消息的发送方式有单播和广播两种。

meet消息，只有处于handshake状态的节点会向对方节点发送meet消息。以下情况可能进行handshake，

* 集群初始化，需要将节点彼此连接起来，此时会通过cluster meet命令来进行handshake操作。
* ...

每次接收到pong后会将ping_sent置为0

clusterProcessGossipSection
遍历这些消息，对每一个消息进行如下处理：

* 在日志中记录该节点状态。
* 在当前节点中查找该节点实例，如没有且不在黑名单中，则新建一个处于REDIS_NODE_HANDLESHAKE的节点实例，并添加到实例字典中，在下次clusterCron中再建立连接。
* 如果该节点实例存在，发送gossip消息的为主节点，且该节点实例不是指向当前节点，查看节点是否被报告失败

    * 被报告失败，则添加到失败报告列表中，计算报告该节点失败的总数，如果达到quorum，若失败节点为主节点，则向其他节点广播该节点失败的消息。
    * 报告没有失败，则从报告失败节点中删除失败报告。

* 若节点被报告失败，且ip或port改变了，则重新开始握手。

消息处理函数clusterProcessPacket分析

* 若发送节点已知，则更新当前节点对发送节点的认知，包括以下内容

    * 发送节点的currentEpoch比当前节点的大，则更新集群中当前节点的currentEpoch。
    * 当前节点对发送节点的configEpoch认知过时，更新，并设置CLUSTER_TODO_SAVE_CONFIG标志，下次进入事件循环前，更新集群配置文件。
    * MF时，从节点接收到来自其master节点在停止向客户服务时的复制偏移量，更新该值，等待当前从节点完成该偏移量的复制，然后就可以开始MF了（非强制性MF）。

* 根据不同消息类型进行处理，ping/pong/meet消息处理占据了大量代码量。

CLUSTERMSG_TYPE_MEET

* sender未知，在当前节点中为sender新增一个节点实例（REDIS_NODE_HANDSHAKE状态），设置事件循环前进行集群配置保存。
* 处理gossip消息部分，并向sender发送一个pong消息（gossip消息部分包含了最多3个随机的当前节点认知的状态良好的节点信息）。

### 2.3 容错
Redis集群在容错方面包括手动容错、发现节点不可用时进行容错、节点迁移。

#### 2.3.1 manual failover
从节点动作：

* 接收到manual failover命令；初始化manual failover信息，设置mf_end（当前时间+5s）；如果是强制failover，则设置mf_can_start（跳过步骤2和3）；否则向master发送一个CLUSTERMSG_TYPE_MFSTART消息
* 从节点收到来自主节点的带CLUSTERMSG_FLAG0_PAUSED的（ping）消息，更新mf_master_offset标志。
* 主从完成复制，从节点的复制偏移量达到mf_master_offset，设置mf_can_start标志，开始进行failover
* clusterHandleSlaveFailover进行failover，先计算候选人的开始选举的延迟，并向当前节点的拥有共同复制节点的从节点广播一个PONG消息，更新复制偏移量。计算选举延迟时，(500 + random()%500 + auth_rank*1000)ms，复制偏移量最大的能够最先发起选举，其他候选人拥有更多的惩罚。
* 等待选举开始的延迟时间。
* 开始选举，向其他节点发起投票请求（发送带有FORCEACK的消息CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST）。
* 从节点收到来自主节点的投票，更新票数统计，查看是否获得足够的票数，如果没有获得，继续等待投票（可能超时，这就需要进行failover重试等动作）。选举若超时，则会重试。
* 获取到足够的票数（没有超时），开始进行failover；进行以下动作

    * 将当前节点转变为主节点，取消复制。
    * 将原先主节点的槽位分配给自己，接管这些槽位。
    * 更新configEpoch为failover_auth_epoch
    * 更新集群状态，并保存集群新的配置。
    * 向其他所有节点发送pong消息，通告自己成为主节点。
    * 重置manual failover状态，完成failover，进行服务。

主节点动作：

* 接收到来自客户端的MFSTART消息，设置mf_end时间（当前时间+5s）；暂停向客户端服务，停止时间为当前时间+10s。
* 停止向客户端服务，在clusterCron中，向从节点发送一个带标志CLUSTERMSG_FLAG0_PAUSED的ping消息，向客户端通告其停止服务时的复制偏移量。
* 等待主从复制完成。
* 收到来自从节点的投票请求（包括其他所有主节点，都会进行看是否投票给该从节点）clusterSendFailoverAuthIfNeeded，若投票通过，则发送CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK给发起投票的从节点。

投票规则clusterSendFailoverAuthIfNeeded

* 分配了槽位的主节点才有投票权。
* 当前投票轮次中，若已经投票了，不再投票；若上次选举时间没超过节点超时时间的2倍，不投票。
* 候选人要么是手动failover，要么是从当前节点来看，其主节点已经挂了。
* 候选人的currentEpoch小于当前节点，不投票。
* 从当前节点的角度来看，若正在工作的槽位的configEpoch大于候选节点的configEpoch，不投票。

对于非手动failover时，从节点跳过步骤1-3（此时其主节点已经down了）；而其他主节点参与的机会只是进行投票。

#### 2.3.2 节点迁移
在节点clusterCron中，会统计每个分配了槽位的主节点状态良好的从节点数目，若发现有主节点没有从节点，即孤儿主节点，则会从拥有最多状态良好的主节点中挑选一个从节点（挑选规则是节点名字最小的那个从节点），将该从节点分配给孤儿主节点。

@WHY 在实现节点迁移的函数clusterHandleSlaveMigration中，再一次的遍历了所有节点，在集群的当前实现中，由于单进程加事件机制，在clusterCron中已经遍历取到了这些数据，在迁移时应该可以直接利用。不知这里是否还有没有注意到的地方。

### 2.4 数据分片
所有的key被划分成16384个槽位，函数keyHashSlot返回key所属的槽位。在该函数中，若key中包含"{}"对，则只有花括号中的那部分会用于计算槽位；若花括号中没有字符，则整个key会被用作槽位计算；若有嵌套花括号，则只有第一个花括号对之间的内容会被用做计算槽位。

### 2.5 槽位迁移
@WHY 每个节点实例的slot位图不是很必要，对于该位图提供的功能，在clusterState中的slots数组完全能够同样高效的提供。

* 查看某个槽位是否由指定节点负责：位图中，通过查看该节点实例位图，看是否置位；其实通过slots中节点实例指针比较即可知道是否由该节点负责。O(1)
* 遍历某个节点负责的槽位：位图中，遍历位图中每个点即可；同样，slots遍历，比较指针即可。O(n)

不使用位图的时候，发送消息就比较麻烦了。


[1]: images/redis/redis-cluster-topology.png "redis-cluster-topology"
[3]: images/redis/redis-cluster.png "redis-cluster"
[4]: images/redis/redis-cluster-msg.png "redis-cluster-msg"
[2]: https://github.com/kiterunner-t/krt/blob/master/t/redis/create-cluster-test.sh
