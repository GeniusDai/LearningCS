# Redis
*From Source Code*

Event-Driven
------------

redis的事件驱动分为三类：aeFileEvent、aeTimeEvent、aeFiredEvent

Background I/O Service
----------------------

bio分为两种：删除文件与AOF持久化

后台服务实现过程：
* 使用POSIX线程实现，信号量与互斥锁实现控制
* bioInit创建两个线程，线程运行函数为bioCreateBackgroundJob，是一个死循环函数。该函数会检查对应类型的链表，如果有元素就执行close或aof_fsync，如果没有就投入睡眠。这里使用pthread_cond_wait与pthread_cond_signal实现信号量，pthread_cond_wait内部会将mutex解锁并等待信号，此时可以对链表加锁并添加元素。
* bio还实现了bioKillThreads用以崩溃时结束线程