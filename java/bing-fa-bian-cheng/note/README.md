# Note

### Java 并发编程，JUC并发编程

Java平台上的，进程，线程，并发，并行，以及Java并发工具，并发问题以及解决方案

#### Java并发

* 进程
* 线程
* 并发 共享模型
  * 管程 悲观锁
  * JMM
    * 原子性
    * 可见性
    * 有序性
  * 无锁 乐观锁
  * 不可变
  * 并发工具
    * 线程池
    * JUC
      * Lock
      * Semaphore
      * CountdownLatch
      * CyclicBarrier
      * ConcurrentHashMap
      * ConcurrentLinkedQueue
      * BlockingQueue
      * CopyOnWriteArrayList
    * disruptor
    * guava
  * 异步编程
    * CompletableFuture
    * 反应式
      * Project-reactor
      * spring webflux
* 非共享
  * 私有
  * Actor
    * akka
* 并行
  * 函数编程
  * 并行编程
    * 映射
    * 规约

#### 应用

* 效率
* 限流
* 同步
* 异步
* 缓存
* 队列
* 分治
* 统筹

#### 原理

#### 模式

* Balking
* Guarded Suspension
* 控制变量
* 两阶段退出
* WorkThread
* Thread per Message
* 生产者/消费者



