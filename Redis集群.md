
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
* 为每个节点设置epoch，可选。
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

* 集群节点管理（节点自动发现）
* gossip消息分发
* 主从容错（选举、从变主、自动容错、手动容错）
* 数据分片，每个slot负责一部分键
* slot迁移，进和出


[1]: images/redis/redis-cluster-topology.png "redis-cluster-topology"
[3]: images/redis/redis-cluster.png "redis-cluster"
[2]: https://github.com/kiterunner-t/krt/blob/master/t/redis/create-cluster-test.sh
