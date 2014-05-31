
    kiterunner_t
    TO THE HAPPY FEW


## 环境准备
从2.6.4版本为基础了解redis的设计与实现，首先搭建一个原始模型，以便根据这个模型分析其代码的设计与实现（当然，随着进一步对redis细节的了解，肯定会对该模型进行调整，以便更适合分析其设计与实现细节）。在对该版本有较深的了解后，跟随github代码库，追踪新功能添加、bug/issue等过程，更进一步的了解redis的发展，直至最新版本，以求更全面的掌握这个分布式的k-v存储数据库。

用于分析的redis网络结构下图所示。由于主要目的在于分析，故所有节点都放在单机环境，通过不同进程来模拟（真实环境绝对不会这样做）。当前主要有一主两从节点，一个监视节点（sentinel也提供了一些集群的功能，诸如failover）。每个节点的配置当前采用redis2.6.4的默认配置（修改一下代码目录下的配置文件中端口和pid文件即可，这里就不列出来了）。

![redis-topology][1]


整个系统启动过程如下：

    redis-server ./redis.6379.conf >log/6379.log 2>&1 &
    redis-server ./redis.6380.conf >log/6380.log 2>&1 &
    redis-server ./redis.6381.conf >log/6381.log 2>&1 &
    
    redis-server ./sentinel.conf >log/sentinel.log 2>&1 &
    
    redis-cli -h 192.168.47.120 -p 6380 slaveof 192.168.47.120 6379
    redis-cli -h 192.168.47.120 -p 6381 slaveof 192.168.47.120 6379


后续分析以该模型为基础，不断修改，争取构建一个适于分析redis的模型。


[1]: images/redis-topology.png "redis-topology"
