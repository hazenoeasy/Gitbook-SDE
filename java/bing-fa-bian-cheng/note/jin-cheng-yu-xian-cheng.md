# 进程与线程

### 1. 概念

#### 1.1 进程 Process

程序由指令和数据组成，指令加载到CPU，数据加载至内存。进程就是用来加载指令，管理内存，管理IO的。

当程序被执行，从磁盘加载这个程序的代码至内存，这时就开启了一个进程。

进程就可以视为程序的一个实例。

#### 1.2 线程 Thread

一个进程可以分为多个线程。

一个线程就是一个指令流，将指令流的一条条指令以一定的顺序交给CPU执行。

Java中，线程作为最小调度单位，进程作为资源分配的最小单位。在windows中进程是不活动的，只是作为线程的容器。\


#### 1.3 两者对比

进程基本相互独立，而线程存在于进程内，是进程的一个子集。

进程拥有共享的资源，如内存空间等，供其内部的线程共享。

进程通信较为复杂

* 同一台计算机的进程通信称为IPC
* 不同计算机之间的进程通信，通过网络

线程通信较为简单，因为他们共享进程的内存，一个例子是多个线程可以访问同一个共享变量。

线程更轻量，线程上下文切换一般比进程上下文切换成本低。

#### 1.4 并行与并发

并发concurrent：单核CPU情况下，线程实际还是串行执行的，操作系统的任务调度器，将cpu切片，分给不同的线程使用。

并行 paralle: 多核CPU，同一时间做多个事情

### 2. 应用

#### 2.1 异步调用

方法调用者，需要等待结果返回，才能继续运行，就是同步。

方法调用者，不需要等待，就能继续运行，就是异步。

多线程可以让方法的执行变为异步。

#### 2.2 提高效率

IO操作不占用CPU，只是一般拷贝文件使用的是阻塞IO，这时相当于线程虽然不用CPU，但是需要一直等待IO结束，没能充分利用线程，所以后面会有非阻塞IO和异步IO的优化

### 3. Java 线程

#### 3.1 创建线程

 方法1  创建一个Thead子类

```
Thread t = new Thread(){
    public void run(){
     // 要执行的任务   

    }
}
t.setName("t1")
t.start() // 把代码交给任务调度器，由任务调度器负责执行

//lamda
Thread thread = new Thread(() -> System.out.println("helloworld"));
```

方法2 将任务作为参数

实例化Runnable， 然后将runnable作为参数，创建thread实例.

```
Runnable runnable = new Runnable(){
    public void run(){
        // 要执行的任务
    }
}
Thread t = new Thread( runnable );
t.start

Runnable runnable = ()->{xxx};
```

Runnable 带有 注解@FunctionalInterface 可以被lamda简化 省去类名

方法3 Future Task&#x20;

Future Task 可以接受Callable 类型的对象，用来处理有返回结果的情况

```
  FutureTask<Integer> futureTask = new FutureTask<Integer>(new Callable(){
            @Override
            public Object call() throws Exception {
                return null;
            }
        });

// lamda
FutureTask<Integer> task = new FutureTask<>(()->{}) // 传入一个callable对象

new Thread(task,"task3").start() // 交给线程去处理
Integer result = task.get()  // 主线程阻塞，等待task完成
```

#### 3.2 查看进程

Linux:

* ps -fe 查看所有进程
* ps -fT -p \<PID> 查看某一个进程的所有线程
* kill \<PID> 杀死进程
* top 按大写H切换是否显示线程
* top -H -p \<PID> 查看某个进程\<PID> 的所有线程

JAVA:

* jsp命令查看所有的Java进程
* jstack \<PID> 查看某个Java进程的所有线程状态
* jconsole 来查看某个Java进程中线程的运行情况（图形界面）

#### 3.3 线程运行原理

* Stack and Frame
  * JVM由堆，栈，方法区组成，栈内存给线程用。
    * 每个栈由多个帧Frame组成，对应着每次方法调用时所占用的内存。一个方法一个帧,
    * 每个线程只能有一个活动帧，对应着当前正在执行的方法。方法执行完毕后，根据返回地址，指回上一个帧
    * frame包括了: 局部变量表，agrs，返回地址，锁记录，操作数栈
    * 可引用的数据存储在堆heap中，frame的agrs只存储地址
*   Thread Context Switch

    * 因为以下原因，cpu执行其他线程的代码
      * cpu时间片用完
      * 垃圾回收
      * 有更高优先级的线程需要执行
      * 线程自己调用了sleep,synchronized,lock 等方法
    * 当上下文切换时，需要操作系统保存当前的线程状态，并恢复另一个线程的状态，Java中对应的概念就是程序计数器，Program Counter Register，它的作用是记住下一条jvm指令的执行地址，是线程私有的
    * 状态包括程序计数器，虚拟机栈中每个栈帧的信息，如局部变量，操作数栈，返回地址等。



3.4 常见方法

| 方法名           | 功能                       | 注意                                                         |
| ------------- | ------------------------ | ---------------------------------------------------------- |
| start         | 启动新的线程，在新的线程运行run方法中的代码. | start只是让线程进入就绪，里面的代码不一定立即执行，因为cpu的时间片还没分配到它，每个thread只能调用一次 |
| run           |                          |                                                            |
| join          | 等待线程结束                   |                                                            |
| join(long n)  | 等待线程结束后，最多等待n ms         |                                                            |
| getd          | 获取线程长整型id                |                                                            |
| getName       | 获取线程名                    |                                                            |
| setName       |                          |                                                            |
| getPriority   | 获取线程优先级                  | 1-10                                                       |
| setPriority   |                          |                                                            |
| getStatus     | 获取线程状态                   | NEW, RUNNABLE, BLOCKED, WAITING,TIMED\_WAITING, TERMINATED |
| isInterrupted | 判断是否被打断                  | 不会被清除                                                      |





