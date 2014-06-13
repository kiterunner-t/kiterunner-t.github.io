
    kiterunner_t
    TO THE HAPPY FEW


本文简要介绍了并发编程的概念及基本方法，随后重点放在POSIX线程上。这里我介绍了线程的基本概念，同步，然后介绍了多线程编程的常用模型，顺带介绍了UNP上的网络程序设计的基本手段。随后，列出了线程编程的困难，以及本文未完成接下来需要继续的工作。最后列出了所用到的参考资料。

以前接触到线程的机会并不多，除了自己写的toy code之外，更是没有机会在实际工作中使用，因此也就免不了花拳绣腿之嫌。文中大部分内容相当于后文所列参考资料的书摘，极少部分自己思考所得。

## 1 并发
若逻辑控制流在时间上重叠，那么它们就是并发的，这种现象被称为并发。换句话说就是让实际上可能串行发生的事情好像是同时发生的一样。并发出现在计算机系统的不同层面上，硬件异常处理程序，进程，线程，信号处理，I/O多路复用（事件驱动）等等都是并发的例子。并发描述了单处理器系统中线程或进程的行为特点。

异步表明事情是相互独立发生的，除非有强加的依赖性。现实世界是异步的，依赖性是大自然补充的，彼此互不相干的事件可以同时发生。传统的计算机系统让所有事件依次发生，而进程线程概念使计算机也能是异步的，也是计算机系统从现实中模拟的开始。

并发给我们带来了很大的好处，如访问慢速I/O设备，人机交互，服务于多个网络客户端，多核机器上并行计算等。现代操作系统为开发应用级并发程序提供了三种最基本的方法：进程，I/O多路复用，线程。

并发应具有的基本功能是：

* 执行环境，它是并发实体的状态。并发系统必须提供建立、删除执行环境和独立维护它们状态的方式。现代计算机系统的体系结构决定了内存在执行环境中的重要位置，并发系统内存可视范围的重要性也就不言而喻了。
* 调度，决定在某个给定时刻该执行哪个环境，并在不同的环境中切换。
* 同步，为并发执行的环境提供协调访问共享资源的一种机制。

考虑这三种最基本的并发机制，下表进行了简易对比。

![pthread-concurrent-mechanism][1]


线程是进程的租借者。

在这三种并发手段中，缺点，数据共享困难，当并发量上升时，进程上下文切换的开销将会成为巨大的性能瓶颈；而线程切换比进程快速，编程模型也简单，容易理解；I/O多路复用编码复杂，随着并发粒度的减小，复杂性会上升，同时也不能充分利用多核处理器。在网上有一系列文章 “Why Threads Are a Bad Idea?”, “Why Events Are a Bad Idea?”, 比较详细的罗列了事件驱动和线程的缺点。

## 2 POSIX线程
线程API的国际标准时POSIX 1003.1c-1995, 于1995年6月通过。POSIX 1003.1的一个新版本，即ISO/IEC 9945-1:1996(ANSI/IEEE标准1003.1, 1996版)也由IEEE通过。该新版本包含了1003.16-1993 (实时), 1003.1c-1995 (线程)和1003.1i-1995 (对1003.1b-1993的改进)。该标准列举了下列不透明的线程数据类型，可移植性的代码不能对这些数据类型的实现做任何假设。

![pthread-object][2]


注意线程的错误检查机制，Pthreads的函数出错时不会设置errno变量，而是通过函数的返回值来表示错误状态。

针对pthread_t的API: 

    int pthread_create(pthread_t *thread, const pthread_attr_t *attr, 
    						void *(*start)(void *), void *arg);
    int pthread_exit(void *value_ptr);
    int pthread_equal(pthread_t t1, pthread_t t2);
    pthread_t pthread_self(void);
    int sched_yield(void);
    int pthread_detach(pthread_t thread);
    int pthread_join(pthread_t thread, void **value_ptr);


## 3 同步
同步是关于线程如何看待计算机内存的问题。Pthreads提供了一些有关内存可视性的规则：

* 当线程调用pthread_create时，它所能看到的内存值也是它建立的线程能够看到的。任何在调用pthread_create之后向内存写入的数据，可能不会被建立的线程看到，即使写操作发生在启动新线程之前。
* 当线程解锁互斥量时看到内存中的数据，同样也能被后来直接锁住（或通过等待条件变量锁住）相同互斥量的线程看到。同样，在解锁互斥量之后写入的数据不必被其他线程看见，即使写操作发生在其他线程锁互斥量之前。
* 线程终止时看到的内存数据，同样能够被join该线程的其他线程看到。终止后写入的数据不会被join线程看到，即使写操作发生在连接之前。
* 线程发信号或广播条件变量时看到的内存数据，同样可以被唤醒的其他线程看到。而在发信号或广播之后写入的数据不会被唤醒的线程看到，即使写操作发生在线程被唤醒之前。

现代计算机复杂的存储层次结构（CSAPP第6章有详细介绍）使得线程的同步问题变得更加复杂。局部性原理降低了CPU访问主存的开销，但是由此带来了读写/回写排序，内存屏障，内存粒度操作等引起的同步问题。在编写多线程代码时，首先就必须确保线程那些只有一个线程能访问的某段数据，在这里线程是与自己同步的。如线程寄存器变量，线程堆栈，register/auto变量等。其次，在任何超过一个线程需要访问相同数据时，就必须利用其中一条内存可视性规则进行同步操作。

pthread提供了互斥量，条件变量，读写锁等同步方法，“POSIX多线程程序设计”中扩展了一种同步方法，barrier, 也用信号量和互斥量实现了一个读写锁(UNPv2第8章也实现了一个读写锁)。

几个同步的关键性概念：
不变量是指由程序作出的假设，特别是有关变量组间关系的假设。这里重要的并不完全是数据，还有数据之间的关系。如队列头或者为空，或者包含一个指向队首元素的指针，这个关系就是队列包中的不变量。

临界区是指影响共享数据的代码段。由于大部分程序员习惯于思考程序功能而非程序数据，所以认识临界区比认识数据不变量更容易。不过，临界区总能够对应到一个数据不变量，反之亦然。

谓词是描述代码所需不变量的状态的语句。谓词可以是一个布尔变量，也可以是测试指针是否为空的测试结果，还可以是更复杂的表达式（如判断计数器是否大于其下限），甚至可以是某些函数的返回值（如用select返回值来判断输入文件是否可用）。

### 3.1 互斥量
互斥量是一种特殊的信号量。

    pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
    int pthread_mutex_init(pthread_mutex_t *mutex, pthread_mutexattr_t *attr);
    int pthread_mutex_destroy(pthread_mutex_t *mutex);
    
    int pthread_mutex_lock(pthread_mutex_t *mutex);
    int pthread_mutex_unlock(pthread_Mutex_t *mutex);
    
    // 非阻塞式互斥锁
    int pthread_mutex_trylock(pthread_mutex_t *mutex);


使用互斥量可以实现操作的原子性。任何对不变量感兴趣的线程在修改或检测不变量状态时都必须使用同一个互斥量。当对多个共享变量进行保护时，锁的粒度就显得很重要了，基本策略有两种，为每个变量指派一个小的互斥量，或者为两个变量指派一个大的互斥量，设计时的主要考虑因数有：

* 互斥量不是免费得，需要时间来加解锁。锁住较少互斥量的程序运行得更快，所以互斥量应尽量少，够用即可，每个互斥量保护的区域应尽量打。
* 互斥量的本质串行执行的。如果很多线程需要频繁加锁同一个互斥量，则线程的大部分时间就会在等待，这有害性能。如果互斥量保护的数据或代码包含彼此无关的片段，则可以将大的互斥量分解为几个小的互斥量来提高性能。所以，互斥量应该足够多，每个互斥量保护的区域则应尽量的少。
* 上两个方面互相矛盾，通常需要一些实验来获得正确的平衡。

当多个互斥量加锁事，复杂度会增加，最坏的情况就是死锁。避免死锁的常用策略是固定加锁层次和试加锁/回退算法。

固定加锁层次伪代码（顺序本身并不重要，重要的是按照固定的顺序加解锁）

    // a线程
    pthread_mutex_lock(&mutex_a);
    pthread_mutex_lock(&mutex_b);
    
    // b线程
    pthread_mutex_lock(&mutex_b);
    pthread_mutex_lock(&mutex_a);


回退意味着以正常方式锁住集合中的第一个互斥量，而调用pthread_mutex_trylock函数有条件的加锁集合中其他互斥量。若pthread_mutex_trylock返回EBUSY, 则释放已经拥有的所有属于该集合的互斥量并重新开始。

### 3.2 条件变量
条件变量是用来通知共享数据状态信息的。条件变量的作用是发信号，而不是互斥，所以条件变量本身也时常需要互斥量来提供互斥。

在线程被唤醒时再次检测谓词是否为真是个不错的主意，理由来自被拦截的唤醒，松散的谓词，假唤醒等。

    pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
    int pthread_cond_init(pthread_cond_t *cond, 
                               pthread_condattr_t *condattr);
    int pthread_cond_destroy(pthread_cond_t *cond);
    
    int pthread_cond_wait(pthread_cond_t *cond, 
    pthread_mutex_t *mutex);
    int pthread_cond_timedwait(pthread_cond_t *cond, 
    pthread_mutex_t *mutex, 
    struct timespec *expiration);
    
    int pthread_cond_signal(pthread_cond_t *cond);
    int pthread_cond_broadcast(pthread_cond_t *cond);


### 3.3 读写锁

### 3.4 barrier
barrier是将一组成员保持在一起的一种方式，它通常被用来确保某些并行算法中的所有合作线程在任何线程可以继续运行之前到达算法中的一个特定点。


    typedef struct barrier_tag {
    	pthread_mutex_t mutex;
    	pthread_cond_t  cv;     // wait for barrier
    	int                valid; // set when valid
    	int                threshold; // number of threads required
    	int                counter;   // current number of threads
    	unsigned long    cycle;      // count cycles
    } barrier_t;
    
    int barrier_init(barrier_t *barrier, int count);
    int barrier_destroy(barrier_t *barrier);
    int barrier_wait(barrier_t *barrier);


## 4 多线程编程模型
下表是我所了解到的多线程编程常见模型，限于没有更多的实际使用经验，该小节涉及到的知识仅仅是那些参考资料的一些书摘，掺杂少量自己对这些模型的理解，以备在编程设计时参考，那就自不免有生涩之感。我画出了每个模型的图形表示，同时罗列出书上所附源码的数据结构的设计和接口函数的声明，只作简要说明。

![pthread-program-model][3]


### 4.1 流水线
如下图所示，流水线类似于生产车间的装配线，在创建流水线并启动它后，依靠传送带从线程1依次向下传递直到最后一个线程3, 最后从流水线尾端取出成品。

![pthread-pipeline][4]


stage_t代表了流水线中的每一个步骤。avail用来通知某步要处理的数据已经准备好，而每一个步骤准备好处理新数据时也会发信号给自己的ready条件变量。

pipe_t则描述了一条流水线，head表示流水线中的第一个线程，tail为最后一步，但这是一个特殊的stage_t, 因为此时产品已经完成，所以没有对应的线程，只是最终保存结果的地方。

    typedef struct pipe_tage *pipe_p;
    typedef data_t            int;
    
    typedef struct stage_tag {
        pthread_mutex_t   mutex;
        pthread_cond_t    avail;
        pthread_cond_t    ready;
        
        pthread_t         thread;
        int               data_ready;
        data_t            data;
        
        struct state_tag *next;
    } stage_t;
    
    typedef struct pipe_tag {
        pthread_mutex_t  mutex;
        stage_t         *head;
        stage_t         *tail;
        int              stages;
        int              active;
    } pipe_t;
    
    int   pipe_create(pipe_t *pipe, int stages); // 创建流水线
    int   pipe_start(pipe_t *pipe, data_t value); // 启动流水线
    int   pipe_result(pipe_t *pipe, data_t *result); // 成品
    
    void *pipe_linebelt(void *arg); // 传送带
    // 在传送带上完成递送动作
    int   pipe_send(stage_t *stage, data_t data);


这里是最开始的想法，这几天重新实现了一次，代码见这里[https://github.com/kiterunner-t/krt/tree/master/t/linux/][5]（thread/pipeline.c）。

### 4.2 工作组

![pthread-crew][6]


### 4.3 工作队列
工作队列是一组线程从同一个队列中接受由主线程分配的工作任务，并且并行的处理它们。从另一个角度看也可以认为工作队列管理器是一个工作组成员管理器。

![pthread-workqueue][7]



    typedef struct workq_ele_tag {
        struct workq_ele_tag *next;
        void                 *data;
    } workq_ele_t;
    
    typedef struct workq_tag {
        pthread_mutex_t  mutex;
        pthread_cond_t   cv;          /* wait for work */
        pthread_attr_t   attr;        /* create detached threads */
        
        int              valid;       /* set when valid */
        int              quit;        /* set when workq should quit */
        int              parallelism; /* number of threads required */
        int              counter;     /* current number of threads */
        int              idle;        /* number of idle threads */
        
        workq_ele_t     *first;
        workq_ele_t     *last;
        
        void (*engine)(void *arg);
    } workq_t;
    
    int workq_init(workq_t *wq, int threads, void (*engine)(void *));
    int workq_destroy(workq_t *wq);
    int workq_add(workq_t *wq, void *data);


### 4.4 客户/服务器

![pthread-server-client][8]


### 4.5 生产/消费

![pthread-producer-cosumer][9]


### 4.6 网络服务器多线程的设计方式
在今天网络大行其道的世界里面，同时进程间通信很多时候也采用socket方式，因此网络服务器的设计方式就显得举足轻重了。在UNPv1, 3rd的第6.2小节总结了UNIX的I/O模型，第30章总结了并发服务器的设计方式。陈硕在“多线程服务器的常用编程模型”中推荐event loop per thread + thread pool模式。而更好的事件机制，epoll/kqueue等的出现，使得C10K问题早已不是问题。

## 5 Why threads are a bad idea?
按照John Qusterhout的观点，线程对于大多数程序员来说过于复杂，即使对那些专家而言，基于线程的应用开发也是一件痛苦的事情。这种复杂和痛苦来自于同步和死锁，难于调试 (because data dependencies, timing dependencies), 破坏了抽象不能独立的设计模块/模块高度耦合，要达到高性能是困难的，标准库不是线程安全的，等等不一而足。然后他列举了事件驱动的并发设计。最后的结论是
并发本质上是困难的，只要可能，避免使用。
线程比事件驱动更强大，但是这种强大却很少被需要。
线程编码比事件驱动编码要复杂得多，应当仅由高手去编码。
使用事件驱动作为主要的开发工具，在GUI和分布式系统里面都应如此。
仅仅当性能必须时才使用线程。 （原文 Use threads for performance-critical kernels, 貌似翻译得不恰当。）

线程编程是困难的，但是在多核时代，线程却是无处不在了。今天线程的工具以及开发经验都不是当年可以比拟的了。只要我们时时小心这些前辈的经验和陷阱，相信线程会成为我们手中强大的工具的。很多时候，我都将进程比作一个人，进行着自己的使命，一直以来却没有给线程找到一个合适的定位。今日偶尔所感，或许将线程比作个体之内的人格或某一方面的行为能力之集合是恰当的。

## 6 未完成的工作
以前接触到线程的机会并不多，除了自己写的toy code之外，更是没有机会在实际工作中使用，因此也就免不了花拳绣腿之嫌。文中大部分内容相当于后文所列参考资料的书摘，极少部分自己思考所得。

线程属性和POSIX关于线程的一些语义本文没有过多涉及，一则线程的默认属性在大部分场景下够用，二则这里主要抓住线程的基本使用模式，细节不过分追究。而那些令人蛋疼的多线程下的信号以及多进程多线程的东西，就更没去考虑了。

UNPv2给出了一个读写锁的实现，《C语言接口与实现——创建可重用软件的技术》最后一章给出了一个用户级线程的实现，这些我都跳过去了。第4小节提到的多线程编程模型的一些理解并不十分到位，对于程序员来说，除非自己去实现一次，哪怕是相对更简单一点的模型，否则代码就不是自己的。

## 7 参考资料

* [Butenhof 1995] POSIX多线程程序设计。David R. Butenhof （于磊，曾刚 译），中国电力出版社，2003. 原书写得很好，详尽介绍了POSIX线程的语义，介绍了常见的多线程编程模型，多线程编程的若干技术性提醒。但是翻译得很烂，推荐直接看英文原版。
* [Bryant, O’Hallaron 2011] 深入理解计算机系统，第2版（简称CSAPP）。Randal E. Bryant, David R. O’Hallaron （龚奕利，雷迎春 译），机械工业出版社，2011. 第8章介绍了进程和信号，第12章介绍了并发编程的常见方法，并相应的给出了例子。
* [Stevens 1998] UNIX环境高级编程，第2版。适合当作手册常备案头，不适合入门之用。相反，CSAPP的第8章，第10到12章详细介绍了UNIX环境下编程的key points和API, 而使我们不致于在APUE复杂的细节里面迷失，把握住关键点。第11, 12章详细罗列了POSIX线程库的API。
* [Stevens 1998] UNIX Network Programming (vol 2): Interprocess Communications, second edition
* [Qusterhout 1996] Why Threads are a Bad Idea (For Most Purpses)
* [陈硕 2009] 多线程服务器的常用编程模型
* [极光] 炉边夜话——多核多线程杂谈


[1]: images/pthread/pthread-concurrent-mechanism.png "pthread-concurrent-mechanism"
[2]: images/pthread/pthread-object.png "pthread-object"
[3]: images/pthread/pthread-program-model.png "pthread-program-model"
[4]: images/pthread/pthread-pipeline.png "pthread-pipeline"
[6]: images/pthread/pthread-crew.png "pthread-crew"
[7]: images/pthread/pthread-workqueue.png "pthread-workqueue"
[8]: images/pthread/pthread-server-client.png "pthread-server-client"
[9]: images/pthread/pthread-producer-cosumer.png "pthread-producer-cosumer"
[5]: https://github.com/kiterunner-t/krt/tree/master/t/linux/
