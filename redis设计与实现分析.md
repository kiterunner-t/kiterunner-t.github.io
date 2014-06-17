
    kiterunner_t
    TO THE HAPPY FEW


## 1 环境准备
从2.6.4版本为基础了解redis的设计与实现，首先搭建一个原始模型，以便根据这个模型分析其代码的设计与实现（当然，随着进一步对redis细节的了解，肯定会对该模型进行调整，以便更适合分析其设计与实现细节）。在对该版本有较深的了解后，跟随github代码库，追踪新功能添加、bug/issue等过程，更进一步的了解redis的发展，直至最新版本，以求更全面的掌握这个分布式的k-v存储数据库。

用于分析的redis网络结构下图所示。由于主要目的在于分析，故所有节点都放在单机环境，通过不同进程来模拟（真实环境绝对不会这样做）。当前主要有一主两从节点，一个监视节点（sentinel也提供了一些集群的功能，诸如failover）。每个节点的配置当前采用redis2.6.4的默认配置（修改一下代码目录下的配置文件中端口和pid文件即可，这里就不列出来了）。

![redis-topology][1]


整个系统启动过程如下：

    redis-server ./redis.6379.conf >log/6379.log 2>&1 &
    redis-server ./redis.6380.conf >log/6380.log 2>&1 &
    redis-server ./redis.6381.conf >log/6381.log 2>&1 &
    
    redis-server ./sentinel.conf --sentinel >log/sentinel.log 2>&1 &
    
    redis-cli -h 192.168.47.120 -p 6380 slaveof 192.168.47.120 6379
    redis-cli -h 192.168.47.120 -p 6381 slaveof 192.168.47.120 6379


后续分析以该模型为基础，不断修改，争取构建一个适于分析redis的模型。

## 2 数据库服务器
信号是单独的，并没有放在redisServer结构中

忽略SIGHUP、SIGPIPE
SIGTERM sigtermHandler，设置server.shutdown_asap，在事件循环中返回，从而退出服务器。
SIGSEGV、SIGBUS、SIGFPE、SIGILL，发生了极端严重的编程错误，debug模块

数据库服务器初始化流程

* 初始化服务器参数，如设置OOM处理器、字典随机数种子，初始化服务器默认配置initServerConfig。
* 命令行选项解析、服务器配置加载。命令行选项解析（非-vh、--test-memory之类的控制服务器的选项）后，读取配置文件，并将命令行选项直接附加到配置文件的sds串中，调用loadServerConfigFromString，将各选项配置到表示服务器的结构体server中。
* 若设置了daemonize选项，则服务器daemonize化。
* 初始化服务器各子模块，如下：

    * 初始化信号处理器，忽略SIGHUP、SIGPIPE，设置SIGTERM触发时置位关闭服务器标志，设置SIGSEGV、SIGBUS、SIGFPE、SIGILL为debug模块处理函数。
    * 初始化日志模块，主要是syslog，普通日志已在前面配置加载部分初始化了。
    * 初始化客户端模块。
    * 初始化slaves模块。
    * 创建服务器共享对象、调整服务器软件限制。
    * 事件循环模块初始化，打开ipfd、sofd套接字。
    * 初始化数据库模块。
    * 初始化发布订阅模块。
    * 初始化持久化rdb/aof模块。
    * 初始化脚本模块。
    * 初始化慢速日志模块。
    * 初始化bio模块。

* 从磁盘加载数据库持久化文件。
* 事件循环，服务器初始化完成，各子模块开始干活。

业务执行流程（接收请求、请求管理、请求返回）
在服务器初始化以及后续来自客户端连接过程中，添加了以下事件（beforeSleep虽然不是事件，但每次进入事件等待前会调用该函数，因此也在这里分析）。

![redis-event-table][2]


每次事件循环进入等待之前，会调用beforeSleep函数处理unblocked_clients以及将命令添加到aof中去。

serverCron
每1ms定时执行一次，定义了一个宏run_with_period表示多少ms执行一次。该函数完成了以下工作：

* 设置watchdog，产生SIGALRM信号；更新服务器时间，更新lru时间，更新内存峰值，每100ms更新一次操作统计信息。
* 查看shutdow_asap是否置位，若是准备关闭服务器。
* 每5s进行一次数据库统计信息输出。
* 无后台进程时，根据需要重设hash表大小，并在激活了rehash动作时进行1ms的rehash。
* 非sentinel模式时，每5s输出一次客户端统计信息，包括客户端数目、slaves数目以及内存使用率
* 执行异步客户端操作 clientCron。
* 没有后台进程，且开启了aof_rewrite_scheduled时，起一个后台进程进行异步aof操作；若有后台，检查是否有后台rdb或aof进程结束，结束时根据rdb或aof操作分别进行善后操作：rdb操作时，准备向slave客户端发送数据；aof时，将server.aof_filename名字修改为临时文件名字。
* 若没有后台进程，检查是否需要rdb或aof操作，若需要则进行后续相应操作。
* 将aof_buf中内容刷新到文件中。
* 激活过期数据，每个周期要么按时间、要么按照最小数目的过期键进行。
* 关闭clients_to_close链表中客户端（为了达到异步关闭的效果）。
* 每秒进行一个复制cron，判断连接、传输、客户端超时；判断是否处于REDIS_REPL_CONNECT状态，进行连接；每10s ping一下slave节点。
* 每100ms进行sentinelTimer操作，tilt模式开启关闭，failover，运行脚本。
* cronloops加1，每个cronloops值代表大约1ms。

## 3 sentinel
sentinel服务器初始化流程，与数据库服务器相比，多数流程一样，少了部分与数据库相关的初始化工作，同时也做了一些数据有关的无用初始化。

* 初始化服务器参数，如设置OOM处理器、字典随机数种子，初始化服务器默认配置initServerConfig。
* 命令行选项解析、服务器配置加载。命令行选项解析（非-vh、--test-memory之类的控制服务器的选项）后，读取配置文件，并将命令行选项直接附加到配置文件的sds串中，调用loadServerConfigFromString，将各选项配置到表示服务器的结构体server中。清空数据库服务器的命令，仅仅sentinel的命令可用。
* 若设置了daemonize选项，则服务器daemonize化。
* 初始化服务器各子模块，如下：

    * 初始化信号处理器，忽略SIGHUP、SIGPIPE，设置SIGTERM触发时置位关闭服务器标志，设置SIGSEGV、SIGBUS、SIGFPE、SIGILL为debug模块处理函数。
    * 初始化日志模块，主要是syslog，普通日志已在前面配置加载部分初始化了。
    * 初始化客户端模块。
    * 初始化slaves模块。
    * 创建服务器共享对象、调整服务器软件限制。
    * 事件循环模块初始化，打开ipfd、sofd套接字。
    * （不需要）初始化数据库模块。
    * 初始化发布订阅模块。
    * （不需要）初始化持久化rdb/aof模块。
    * 初始化脚本模块。
    * （不需要）初始化慢速日志模块。
    * （不需要）初始化bio模块。

* 事件循环，服务器初始化完成，各子模块开始干活。


## 4 服务器各子模块
### 4.1 客户端
这里客户端是指Redis服务器在处理来自客户端的请求过程中管理资源和请求处理的对象，即围绕redisClient对象所做的工作。networking.c文件中主要就是关于这部分的代码。服务器对其管理的方式是利用链表管理所有客户端，并以客户端对应的fd（非脚本类的客户端，该小节只涉及普通的套接字的客户端）为索引添加到事件处理结构中，其中aeFileEvent->clientDat就指向该结构；对于需要关闭的客户端，也组织在一个链表clients_to_close中， redisClient结构用来表示对端的请求、请求解析、请求处理中间参数以及最终的响应（其他如multi、pubsub等本小节暂不涉及）如下图所示：

![redis-client][3]


#### 4.1.1 接收请求
readQueryFromClient，该函数是客户端响应来自对端请求的入口函数，它从socket中读取数据，然后调用协议处理函数处理，处理请求，返回响应。

#### 4.1.2 协议处理
在接收完请求后函数processInputBuffer进行协议解析，请求数据放在redisClient->querybuf中，主要有两种请求类别：REDIS_REQ_INLINE和REDIS_REQ_MULTIBULK，分别调用函数processInlineBuffer和processMultibulkBuffer处理协议。

Redis协议不仅适合人读，也适合计算机解析；在这两个函数实现基本很简单，使用分隔符\r\n分隔请求，按照协议要求组织请求，具体参考图中数据结构示意，一目了然。但在处理过程中有几点需要注意：

* REDIS_REQ_INLINE单个请求输入长度不超过REDIS_INLINE_MAX_SIZE 64K，否则报协议错；此时会设置REDIS_CLOSE_AFTER_REPLY标志，将错误响应发送给对端后，尽快释放掉客户端资源。
* 对于REDIS_REQ_MULTIBULK，请求的multibulk个数不能超过1M，超过则报协议错；每个bulk长度不能超过512M。
* REDIS_REQ_MULTIBULK请求时，若bulklen超过32K，则直接分配足够大的sds内存，避免从socket读取到的数据多次进行拷贝。（这里进行了优化，参考代码）

REDIS_BLOCKED标志需要注意一下。

#### 4.1.3 请求处理
协议解析完成后，调用processCommand进行命令解析（该函数在redis.c中）。

#### 4.1.4 请求响应
处理完请求后，会调用addReply之类的函数将请求发送给对端。redisClient管理响应有一个16K的静态缓冲区和一个reply链表，首先添加到静态缓冲区，若满，再添加到reply链表中。组织结构如上图所示。（响应的协议在协议节单独表述，这里不涉及）。

replybytes字段表示所有reply链表上的数据暂用的内存大小，该大小并不太准确，因为很多小对象是使用的全局共享对象，并不实际占用多少资源。在发送响应过程中，会根据客户端类别（NORMAL、SLAVE、PUBSUB）计算其所占用的资源是否达到软硬限制，该限制值存储在server.client_obuf_limits数组中。checkClientOutputBufferLimits展现了实现限制的算法，如下所示：

* 计算客户端reply所暂用的内存大小，包括replybytes和管理reply的链表的大小，但不包括静态缓冲区大小，并根据其类别确定是否达到限制；
* 达到物理限制则设置物理限制；
* 对于达到软限制，第一次忽略；若在限制时间内clientBufferLimitsConfig->soft_limit_seconds第二次达到，则设置软限制；

* 若达到限制，则将客户端设置为REDIS_CLOSE_ASAP，并添加到链表server->cliens_to_close，进行异步释放处理。

REDIS_CLOSE_AFTER_REPLY标志置位时，不再处理新的响应请求，每次调用addReply时，会直接返回OK，以便尽快结束掉客户端；当写完所有的响应时，会调用freeClient释放掉客户端资源。

REDIS_REPLY_CHUNK_BYTES

sendReplyToClient函数用来发送数据给客户端，先发送redisClient->buf中的数据，再发送redisClient->reply链表中响应数据；注意以下几点：

* 若客户端是REDIS_MASTER，则不实际发送数据，只是直接忽略消耗数据；
* 发送过程中，若实际发送的数据超过16K(REDIS_MAX_WRITE_PER_EVENT)，则停止发送数据；但若此时reply占用的内存过大，超过了server.maxmemory限制，则尽量发送，以便尽快释放内存；
* 发送结束后，若没有更多数据（buf和reply都为空），则从事件循环删除该客户端可写事件，若设置了REDIS_CLOSE_AFTER_REPLY标志时，释放掉客户端。

#### 4.1.5 命令
list

kill <ip:port>

monitor
执行monitor命令的时候，会把该client添加到对应的server->monitors链表中；在执行命令时，会将命令相关信息发到对应的客户端。

### 4.2 复制
复制的过程如下图所示

![redis-replication-interaction][4]


每次读取时最多读取16K的数据。

### 4.3 命令处理
使用redisCommand来表述命令，其实现代码在redis.c中，入口函数时processCommand。

### 4.4 事务
redis通过multi、discard、watch、exec实现数据库事务操作。该部分代码在multi.c中实现。

### 4.5 AOF
Redis持久化有RDB和AOF两种方式，而针对AOF有appendonly和rewrite两种方式。

appendonly会严格的记录对数据库有修改的所有操作，而rewrite则是数据库快照转换成AOF格式，完成后会替换掉appendonly的文件，文件因更小。如数据库执行了以下操作

    RPUSH mylist [1, 2, 3, 4]
    RPOP mylist
    LPUSH mylist 4

那么appendonly方式会记录以上3条命令，而rewrite只会记录最终状态的一条命令，即

    RPUSH mylist [4, 1, 2, 3]


#### 4.5.1 appendonly aof
当打开appendonly标志时，数据库服务器执行的每条命令都会添加到aof_buf中，feedAppendOnlyFile函数进行该动作。该函数执行的动作如下：

* 若当前命令的数据库与aof数据不同，则先添加SELECT命令，然后再按照Redis协议写入命令。

#### 4.5.2 rewrite aof
利用aof rewrite_buf可以有效的减少数据库持久化文件大小。AOF基本流程如下所示：

* fork子进程把当前数据库状况写入AOF文件，期间禁用数据库rehash操作（防止大量的内存页写，导致数据库占用内存高，因为父进程大量写时，子进程会复制父进程的页）。
* 在子进程写AOF文件过程中，所有对父进程的操作都会添加到rewrite_buf中；该buf总大小近似为 (buf数 - 1) * 10MB + 最后一个rewrite_buf->used，填充时总是填充满最后一个buf未用完的空间，再分配下一个rewrite_buf（每个rewrite_buf都为10MB）。
* 后台子进程AOF完成后，在服务器serverCron过程中，检测到子进程结束事件，根据进程是否正常结束进行下述操作：

* 正常结束，将rewrite_buf追加到临时AOF文件中，进行AOF文件同步，打开REDIS_AOF_ON标志（这意味着后续的操作将写入aof_buf中），同时删除旧的（若有）aof_filename，将临时aof文件重命名为aof_filename；
* 非正常结束，重新调度，状态转为REDIS_AOF_WAIT_REWRITE，下次进入serverCron时重新开始AOF基本流程。

在上述动作中，文件同步、关闭文件、重命名文件都可能造成服务器阻塞，参考代码io_delay.c（地址[https://github.com/kiterunner-t/krt/blob/master/t/linux/src/io/io_delay.c][5]），对于前面两者redis使用后台bio进行异步调用，而对于重命名则通过保留一个原始aof_fd的引用，然后放到后台去关闭来解决。

### 4.6 rdb
#### 4.6.1 rdb文件格式
rdb文件格式按照下述规则进行写入：REDIS + 4字节版本号 + 数据库数据 + 结束符0xFF + 8字节的校验和（若未启用校验和，则8字节0）。

数据库数据：0xFE 数据库序号；遍历数据库每个k-v对，按照下述规则写入数据

* （若k-v设置了过期时间，且未过期，则先写入过期操作符）0xFC 8字节的毫秒过期时间；
* 写入value类型，value类型规则见后面表格；
* 写入key，key为STRING类型；
* 写入value。

写入value的类型，根据对象的类型和编码来决定类型

![redis-rdb-type][6]


value编码规则如下：

![redis-rdb-value][7]


STRING分为INT整数和RAWSTRING两种类型，根据编码规则不同分别使用对应的类型进行编码。

INT整数存储规则

![redis-rdb-int][8]


RAWSTRING存储分3种情况

* 长度小于等于11时，若能按整数规则存储，则转换成整数存储；
* 启用了压缩，且长度大于20，使用lzf压缩方式存储；首字节高2位为11，接下来6位为0x03，即首字节为1100 0011；然后依次为压缩字符串长度，压缩前字符串长度，压缩后字符串；
* 普通方式，先存长度，再保存字符串。


长度存储规则

![redis-rdb-len][9]


double类型规则如下

![redis-rdb-double][10]

### 4.7 slowlog
慢速日志记录了那些最耗时的命令及其相关信息，如下图所示，slowlog_log_lower_than小于0时，表示禁用该功能；否则执行时间该值的命令都会被记录在slowlog链表中。slowlog_max_len表示链表的最大长度，当超过该值时，旧的slowlog将会从链表中删除。slowlogEntry中参数argv最多为32，当超过32时，最后一个参数（即第32个）表示该命令后续还剩多少参数，而对于字符串对象参数来说，slowlog只复制前128字节，后续字节被抛弃，其他对象增加引用计数即可。

![redis-slowlog][11]


慢速日志从头部开始插入，丢弃是从链表末尾开始。提供了slowlog命令可以访问该子模块信息

![redis-slowlog-command][12]


### 4.8 pubsub
在redisServer存储了每个频道有哪些client订阅了，每个client订阅了哪些模式（每个client的不同模式会有不同节点，这是通过client下的pattern链表控制的）。发布消息时，先发那些直接订阅的client，然后在遍历模式列表进行模式匹配，匹配的则发送消息。如下图所示，CLIENT_A订阅了频道hello及频道模式PATTERN_A、PATTERN_B：

![redis-pubsub][13]


命令如下图所示：

![redis-pubsub-command][14]


频道模式使用glob规则，参考util.c中函数stringmatchlen。

@WHY 服务器和客户端都存有相关信息，在发布消息时，只用到了服务器端的信息，客户端为何还要保存有相同的信息？在pubsub模块没有直接的引用。

### 4.9 scripts
命令以script开始，见下表

![redis-lua-command][15]


lua函数都以f_<sha-func-body>来命名存在server.lua_scripts中（见图）。该模块只要工作是提供lua运行环境，提供一些基本的安全的lua函数，提供用户自定义函数脚本，并提供Redis C协议和lua栈之间的转换。

### 4.10 debugs

## 5 库模块
### 5.1 事件循环
这里以epoll演示事件循环的机制，不同事件底层机制不同点在于aeApiState。

![redis-eventloop][16]

提供了以下基本接口

* 事件循环的创建、删除
* 事件循环的处理函数aeMain、aeProcessEvents、aeStop
* 定时器事件的添加和删除
* 文件事件的添加和删除
* aeWait

el->stop用于事件循环处理器检测事件是否需要停止。每次事件循环进入epoll等待事件发生之前，若设置了beforeSleep函数，则会调用该函数。

#### 5.1.1 定时器事件
el->lastTime和el->timeEventNextId用于定时器事件。timeEventNextId用于标识新建定时器，内部自增。lastTime用于检测系统时间被调整到将来，然后又调整回去的情况，每次执行定时器事件时，检测当前时间是否小于该时间，若小于则说明发生了，将所有定时器事件置0，强制都被处理。

如图所示，定时器事件使用链表进行管理，每次新增时将定时器插入到链表头。

#### 5.1.2 文件描述符事件
el->setsize限制了处理的最大文件描述符，在初始化过程中，就为events、fired、aeApiState下的events分配setsize大小的数组。添加事件时，以文件描述符fd为下标，直接添加到对应的events中，发生事件的fd以及其事件则放在fired下。

### 5.2 hiredis
hiredis是一个很小巧的用于Redis数据库的客户端库代码，提供了对printf-alike的Redis协议支持，可以对Redis数据库进行同步或异步的访问。

#### 5.2.1 同步
6种reply对象：字符串（STATUS和ERROR也是字符串类型）、整数、NIL、ARRAY。

    *2 \r\n
    $5 \r\n hello \r\n
    *3 \r\n
    :42 \r\n
    +status \r\n
    -error \r\n

上面是一串响应，则结构如下图所示（忽略空格，\r\n为ascii的可读形式）。

![hiredis-sync][17]


整个流程可以简单用文字概括如下：

* 客户端连接服务器后，调用redisCommand之类的命令函数格式化命令，将其转换成Redis协议，存储在redisContext->obuf中。
* 将Redis协议发送给服务器。
* 同步等待来自服务端的响应，此时将响应复制到redisReader->buf中。
* 利用redisReader->rstack堆栈解析响应，将命令的响应以redisReply的形式返回给客户端。hiredis一次尽可能多的读取来自服务器端的响应，但一次却只返回一个响应。客户端可以使用类似管道的机制，将多个命令一次发给服务器，然后可以按照添加命令的顺序获取对应的响应。
* 客户端使用完响应后，释放redisReply资源。
* 
在解析该串响应的时候，使用栈递归解析。上图是解析代码中的服务器端响应的数据结构图。

API如下：

    redisContext *redisConnect(const char *ip, int port);
    void redisFree(redisContext *c);
    void *redisCommand(redisContext *c, const char *fmt, ...);
    void redisAppendCommand(redisContext *c, const char *fmt, ...);
    int redisGetReply(redisContext *ctx, redisReply **reply);
    void freeReplyObject(void *reply);


#### 5.2.2 异步
hiredis也提供了异步的方式进行客户服务端的沟通。如下图所示，异步方式需要与事件循环机制结合，图中所示为ae的数据结构（绿色部分，其他事件循环机制如libev、libevent有所不同）。

![hiredis-async][18]


异步解析的流程为：

* 客户端连接服务端后，将命令格式化成Redis协议形式保存在redisContext->obuf中，同时将fd可写事件注册。
* 当接收到可写事件时，将命令发送给服务端，然后立即注册可读事件，让事件循环去等待客户端的响应，而不是如同步等待一样阻塞客户端。
* 接收到来自服务端的响应，读响应数据，将这原始数据复制到redisContext->buf中，然后调用同步流解析的方式解析响应。

在异步方式下，注册了大量的回调函数，用于事件发生时进行回调。

@WHY hiredis的异步回调应用场景还需要进一步跟踪，在sentinel中有应用。

### 5.3 rio
rio提供了基于文件流和内存流的读、写、位置通告、校验和操作方法（若设置了校验和方法，读写前会进行校验和更新操作），并提供了用于写Redis协议的高层API函数。其基本结构如下图所示（橙色表示函数指针），其中rioFileIO使用标准C流式文件IO进行流式IO操作，rioBufferIO使用sds进行内存流式IO操作。

![redis-rio][19]


提供了几个API使用，如下

    void rioInitWithFile(rio *r, FILE *fp);
    void rioInitWithBuffer(rio *r, sds s);
    
    size_t rioWrite(rio *r, const void *buf, size_t len);
    size_t rioRead(rio *r, void *buf, size_t len);
    off_t rioTell(rio *r);
    void rioGenericUpdateChecksum(rio *r, const void *buf, size_t len);


提供了几个更高层次的API用于Redis二进制协议操作的函数。

![redis-rio-api][20]


### 5.4 bio
bio通过使用后台线程来执行可能阻塞服务器的操作，目前支持两个操作close和fsync。其实现是通过为每个任务创建一个线程，线程在操作的条件变量上等待任务链表中有任务可做；当调用者有任务可做时，通过bio的接口，将任务放在list中，并通知线程进行处理。线程屏蔽了SIGALRM（Redis用其作为watchdog），防止该后台任务处理线程接收到该信号。结构如下图所示（Redis实现中并没有bio结构体，bio中所有成员都是以文件静态变量的形式单独存放）：

![redis-bio][21]


接口如下，初始化过程中会根据类型创建不同的后台线程等待任务执行；创建任务时，提交相应任务到对应的链表中，增加计数器，并通知后台线程进行任务处理；bio_pending保存了后台需要处理的任务数量，可以通过接口获取该值。

    void bioInit(void);
    void bioCreateBackgroundJob(int type,
                                      void *arg1, void *arg2, void *arg3);
    unsigned long long bioPendingJobsOfType(int type);


### 5.5 adlist
adlist是一个通用双向链表的实现，其结构如下图所示。list->len保存了链表中节点的数目，图中橙色部分表示节点的一些操作方法，包括节点复制、节点匹配、节点释放。listIter是链表的迭代器实现，可以从链表头和尾两个方向分别进行迭代。链表的实现简单清晰，具体实现可以直接参考代码。

![redis-ds-adlist][22]


### 5.6 sds
sds是一个动态字符串，本身被定义为char *，但在每个分配的字符串内存前有带有一个sdshdr，如下图所示。在向sds添加数据过程中，sds内存会自动增长（sdscat等请求的大小之外的空间），其增长策略是小于1MB时，按照指数方式扩充，当大于1MB时，每次最多增长1MB。因此，对于大数据的

![redis-ds-sds][23]



    sds sdstrim(sds s, const char *cset);

删除s中开始和结束包含cset的字符，若在开始有，会进行内存移动，保证sdshdr->buf总是有效的内存。


    sds sdsrange(sds s, int start, int end);

把s从start截断到end。start和end为索引位置，如[1, -1]表示从第二个字节到s结束。若s开始位置有变动，则会进行memmove操作。


    sds *sdssplitlen(const char *s, int len, const char *sep, int seplen, int *count);
    void sdsfreesplitres(sds *tokens, int count);

将s以sep为分隔符分隔成若干个sds字符串，s和sep都是二进制安全的。


    sds sdscatrepr(sds s, const char *p, size_t len);

将s转换成人可读的形式，首先在开始结束加上双引号，除下列字符其他字符不进行处理：

* \, "：\\\, \\"
* \n, \r, \t, \a, \b：\\n, \\r, \\t, \\a, \\b
* 不可打印字符：\\x%02x的形式，如\\x0a


    sds *sdssplitargs(const char *line, int *argc);
    void sdssplitargs_free(sds *argv, int argc);

将命令行参数解析成sds数组，argc表示数组大小。


    sds sdsmapchars(sds s, const char *from, const char *to, size_t setlen);

将s中from字符集的字符映射成to中的对应字符集，setlen表示from和to中字符集的个数，二者必须严格一一对应。

### 5.7 dict
字典使用了两个散列桶，双哈希桶的设置的主要功能是将耗资源的resize和rehash两个动作分摊到每一个查询、增加、删除以及周期性等操作中。

![redis-ds-dict][24]


### 5.8 intset

![redis-ds-intset][25]


### 5.9 ziplist
ziplist是一种平坦数据结构，通过编码将整数和字符串放到一段内存中，支持以链表的方式来操作该段内存。ziplist结构如下图所示，头部信息由2个uint32_t的长度和最后一个元素偏移量，以及一个uint16_t计数节点个数，图中也表示了一个空节点对象的内存布局。其中需要注意的是，end偏移量在没有节点时为end节点的偏移量，否则为最后一个节点开始的偏移量。

![redis-ds-ziplist-zl][26]


由于使用uint16_t来表示节点的数量，因此当节点数小于UINT16_MAX时可以直接返回。当超过该值时，需要遍历节点才能知道准确数量；同时，删除、增加等操作都不再更新该域。

每个节点entry由4部分组成，如下图所示，即前一个节点的长度、节点编码、节点长度和节点内容。注意，这几个部分并不一定存在单独的内存字节空间来表示，如对于0-12的整数，编码类型、内容长度和内容三个部分用1个字节就表示了。

![redis-ds-ziplist-entry][27]


* 前一个节点长度由存储该长度所需空间和其值两部分组成，该长度是节点的完整长度，包括节点的所有4个组成部分。该部分所占大小为1或5字节。

    * 节点长度小于等于254，直接使用一个字节存储该长度。
    * 节点长度大于254，则第一个字节为标志位0xFE，后面跟四个字节的长度。

* 其他三个部分的编码情况如表格所示，有3种类型：字符串、整数和结束符。


![redis-table-ds-ziplist-entry][28]


了解了ziplist的内存布局及编码方式，那么查找、删除、范围删除、新增节点等各种操作的实现就较为简单了。需要注意的是在删除、新增节点过程中，由于关联的节点发生了变化，则某些节点的“前一个节点长度”可能容纳不下前一个节点长度，或者长度太长，那么则需要进行级联更新。在实现过程中，Redis只对长度不够的情况进行扩充处理，而对于另一种情况则强制用较大的空间存储该长度，避免更多的内存拷贝操作。具体实现参考__ziplistCascadeUpdate函数。（注意，该函数在特殊情况下可能引起较严重的不断内存拷贝操作）。



[1]: images/redis/redis-topology.png "redis-topology"
[2]: images/redis/redis-event-table.png "redis-event-table"
[3]: images/redis/redis-client.png "redis-client"
[4]: images/redis/redis-replication-interaction.png "redis-replication-interaction"
[6]: images/redis/redis-rdb-type.png "redis-rdb-type"
[7]: images/redis/redis-rdb-value.png "redis-rdb-value"
[8]: images/redis/redis-rdb-int.png "redis-rdb-int"
[9]: images/redis/redis-rdb-len.png "redis-rdb-len"
[10]: images/redis/redis-rdb-double.png "redis-rdb-double"
[11]: images/redis/redis-slowlog.png "redis-slowlog"
[12]: images/redis/redis-slowlog-command.png "redis-slowlog-command"
[13]: images/redis/redis-pubsub.png "redis-pubsub"
[14]: images/redis/redis-pubsub-command.png "redis-pubsub-command"
[15]: images/redis/redis-lua-command.png "redis-lua-command"
[16]: images/redis/redis-eventloop.png "redis-eventloop"
[17]: images/redis/hiredis-sync.png "hiredis-sync"
[18]: images/redis/hiredis-async.png "hiredis-async"
[19]: images/redis/redis-rio.png "redis-rio"
[20]: images/redis/redis-rio-api.png "redis-rio-api"
[21]: images/redis/redis-bio.png "redis-bio"
[22]: images/redis/redis-ds-adlist.png "redis-ds-adlist"
[23]: images/redis/redis-ds-sds.png "redis-ds-sds"
[24]: images/redis/redis-ds-dict.png "redis-ds-dict"
[25]: images/redis/redis-ds-intset.png "redis-ds-intset"
[26]: images/redis/redis-ds-ziplist-zl.png "redis-ds-ziplist-zl"
[27]: images/redis/redis-ds-ziplist-entry.png "redis-ds-ziplist-entry"
[28]: images/redis/redis-table-ds-ziplist-entry.png "redis-table-ds-ziplist-entry"
[5]: https://github.com/kiterunner-t/krt/blob/master/t/linux/src/io/io_delay.c
