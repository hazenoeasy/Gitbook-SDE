# 共享模型之管程

### 1. 共享问题

两个线程同时读写同一个变量

* 一段代码块内如果存在对共享资源的多线程读写操作，称这段代码为临界区
* 多个线程在临界区内执行，由于代码的执行序列不同而导致的结果无法预测，称之为发生了Race Condition
*

### 2. Synchronized

阻塞式解决方案： synchronized, 对象锁，采用互斥的方式让同一时刻至多只有一个线程能持有对象锁。

非阻塞式：原子变量

* 虽然java中互斥和同步都可以采用synchronized关键字完成，但还是有区别的
  * 互斥是保证临界区的竞态条件发生，同一时刻只能有一个线程执行临界区代码
  * 同步是由于线程执行的先后，顺序不同，需要一个线程等待其他线程运行到某一个点

```
synchronized(对象)
{
 临界区
}
```

获得synchronized资源的线程，即使时间片消耗完，也不会让其他线程获取该资源，必须等待此线程任务完成，才会释放资源。

synchronized实际是用对象锁保证了临界区内代码的原子性，临界区内的代码对外是不可分割的，不会被线程打断。

* Synchronzied 还可以加在方法上面
  * 加在成员方法上（非static），等于对this加锁，对实例加锁
  * 加在类方法上（static），等于对对象.class加锁，对类加锁

### 3. 线程安全

#### 局部变量是否线程安全？

* 局部变量是线程安全的
* 但局部变量引用的对象则未必
  * 如果该对象没有逃离方法的作用范围，它是线程安全的
  * 如果该对象逃离了方法的作用范围，需要考虑线程安全
* 多个线程会共享堆，类成员变量只会创建一次，存储在堆内。所以不同的线程引用的都是同一个对象。
*   public的方法可能被子类重写，导致局部变量暴露给新的线程，导致发生线程安全问题

    * private 修饰符，可以保证子类不会重写， 不想子类重写，还可以加上final。



#### 常见线程安全类

多个线程调用他们同一个实例的某个方法时，是线程安全的。也可以理解为 他们的每个方法都是原子的，但多个方法的组合不是原子的。

* String
* Integer
* StringBuffer
* Random
* Vector
* Hashtable
* java.util.concurrent包下的所有类

```
Hashtable table = new Hashtable();
if(table.get("key")==null){
    table.put("key",value);
}
//组合后 线程不安全，因为原子性失去了
```

#### 不可变类线程安全

String,Integer 等都是不可变类，因为其内部的状态不可改变，因此他们的方法都是线程安全的

### 4. Monitor

#### Java对象头

以32位虚拟机为例

![数组对象](<../../../.gitbook/assets/Screen Shot 2022-02-03 at 9.55.28 AM.png>)

![对象头](<../../../.gitbook/assets/Screen Shot 2022-02-03 at 9.54.45 AM.png>)

Klass Word: 指向Class对象的指针

Mark Word:

![标志头](<../../../.gitbook/assets/Screen Shot 2022-02-03 at 9.55.34 AM.png>)

每一个Java对象都可以关联一个Monitor对象，如果使用synchronzied给对象上锁，该对象头的Mark Word中就被设置指向Monitor对象的指针。

![](<../../../.gitbook/assets/Screen Shot 2022-02-03 at 10.02.02 AM.png>)

synchronized本质是，将一个对象作为标示对象，指向固定的一个Monitor，由Monitor维护owner，entrylist。使用同一个对象，就会指向同一个Monitor。

Synchronized 必须是进入同一个对象的monitor才有上述效果。

不加synchronized的对象不会关联监视器，不遵从上述规则。

老王 -> 操作系统

小南 -> 线程

对象 -> 房间

Monitor 是系统锁，所以很重量级 ->防盗锁

轻量级锁 -> 门上挂书包

偏向锁 -> 房间门上刻名字

批量重刻名 -> 偏向锁撤销到达20阈值

不能刻名字 -> 批量撤销该类对象的偏向锁, 设置该类不可偏向&#x20;

#### 轻量级锁

如果一个对象虽然有多个线程访问，但是多线程访问的时间是错开的，也就是没有竞争，那么可以用轻量级锁来优化

语法对使用者是透明的，还是synchronized

* 创建锁记录 Lock Record 对象，每个线程的栈帧都会包含一个锁记录的结构，内部可以存储锁定对象的Mark Word

![](<../../../.gitbook/assets/Screen Shot 2022-02-03 at 11.31.34 PM.png>)

* 让锁记录的Object Reference 指向锁对象，并尝试用cas替换Object的Mark Word, 将Mark Word的值存入锁记录中

![](<../../../.gitbook/assets/Screen Shot 2022-02-03 at 11.33.25 PM.png>)

当后两位是01时，代表是无锁，可以交换

![](<../../../.gitbook/assets/Screen Shot 2022-02-04 at 12.35.48 AM.png>)

对象头状态为00 代表已锁

加锁失败可能性：

* 其他线程已经持有了该Object的轻量级锁，表明有竞争，进入锁膨胀过程
* 如果时自己执行了synchroized锁重入，那么再添加一条Lock Record作为重入的计数，值设为null

![](<../../../.gitbook/assets/Screen Shot 2022-02-04 at 12.39.52 AM.png>)

在解锁的时候 null的record直接删除

其他的，使用cas将Mark Word的值恢复给对象头

* 成功，则解锁成功
* 失败，则说明轻量级锁进行了锁膨胀，或者已经升级为了重量级锁，进入重量级锁解锁流程

#### 锁膨胀

![](<../../../.gitbook/assets/Screen Shot 2022-02-04 at 12.52.39 AM.png>)

Thread1 加锁失败，进入锁膨胀流程

* 为Object对象申请Monitor锁，让Object指向重量级锁地址
* 自己进入Monitor的EntryList Blocked

![](<../../../.gitbook/assets/Screen Shot 2022-02-04 at 12.56.12 AM.png>)

Thread0 退出同步块解锁时，使用cas将Mark Word的值恢复给对象头，失败。这时会进入重量级解锁流程，即按照Monitor地址找到Monitor对象，设置Owner为null, 唤醒EntryList 中 Blocked线程



#### 偏向锁

只有第一次使用CAS将线程ID设置到对象的Mark Word头，之后发现这个线程ID时自己的，就表示没有进程，不用重新CAS，只要不发生竞争，这个对象就归该线程所有。

解锁后，Mark Word中依然不会变。

![](<../../../.gitbook/assets/Screen Shot 2022-02-04 at 1.27.14 AM.png>)

* 如果开启了偏向锁，那么对象创建后，markword值为0x05， 最后三位为101,这时它的thread, epoch, age 都为0.
* 偏向锁默认是延迟的，不会在程序启动时立即生效，如果想避免延迟，可以加入VM参数 -XX: biasedLockingStartupDelay = 0 来禁用延迟。
* 如果没有开启偏向锁，那么对象创建后，markword值为001，它的hashcode，age都为0，第一次用到hashcode时才会赋值。
* 使用hashcode会禁用偏向锁 因为一旦调用hashcode，就会为对象头写入hashcode，从而没有多余的空间留给biased 头
* 当其他线程使用偏向锁时，会将偏向锁升级为轻量级锁，当轻量级锁撤销后，会恢复为001 Noraml状态。
* 使用 wait/ notify 也会撤销偏向锁。

#### 批量重偏向

如果对象虽然被多个线程访问，但没有竞争，这时偏向了线程T1的对象仍然有机会重新偏向T2，重偏向会重置对象的ThreadID

当撤销偏向锁阈值超过20次后，jvm就会觉得自己是否偏向错了，于是会在给这些对象加锁时重新偏向至加锁线程。 修改对象头的ThreadID

![](<../../../.gitbook/assets/Screen Shot 2022-02-05 at 12.38.32 AM.png>)

#### 批量撤销

当撤销偏向锁阈值超过40次后，jvm觉得，自己确实偏向错了，于是整个类所有的对象都会设为不可偏向的，新建的对象也是不可偏向的。

![](<../../../.gitbook/assets/Screen Shot 2022-02-05 at 12.46.01 AM.png>)

#### 锁消除

JIT 会对代码进行优化，自动把不会共享的锁进行消除

### 5. Wait/Notify

![](<../../../.gitbook/assets/Screen Shot 2022-02-05 at 1.22.20 AM.png>)

* obj.wait 让object 进入 monitor的waitSet
* obj.notify 在object上正在waitSet等待的线程中 **随机** 挑一个唤醒
* obj.notify 唤醒所有waitSet的线程
* 线程必须获得此对象的锁，成为onwer后才能调用这种方法 （在synchronized 代码临界区内，才能调用方法)

#### sleep 和 wait 区别

1. sleep 是Thread方法，而wait 是 Object的方法。
2. sleep 不需要强制和synchronized配合使用，但是wait需要和synchronized一起用。
3. sleep 在睡眠的同时，不会释放对象锁，但wait在等待的时候会释放对象锁。

```
synchronized(lock){
    while(条件不成立){
        lock.wait()
    }
}

synchronized(lock){
    lock.notifyAll()
}
```

#### 模式之保护性暂停

Guarded Suspension 用一个线程等待另一个另一个线程的执行结果。

* 有一个结果需要从一个线程传递到另一个线程，让他们关联同一个GuardedObject
* 如果有结果不断从一个线程到另一个线程，那么可以使用消息队列
* JDK中，join的实现，Future的实现，采用的就这种模式
* 因为需要等待另一方的结果，因此归类到同步模式

![](<../../../.gitbook/assets/Screen Shot 2022-02-05 at 10.34.50 AM.png>)

```
class GuardObject {
    private Object response;
    public Object get(){
        synchronized(this){
            while(response == null){
                try{
                    this.wait();
                } catch(InterruptedException e){
                    e.printStackTree();
                }
            }
        }  
    public void complete(Object response){
        synchronized(this){
            this.response = response;
            this.notifyAll();
        }  
    } 
}
}
```

join 等待线程结束

futures 像是信箱，一个大型的中转站，能在多个类之间使用Guarded Object对象，解耦结果等待者和结果生产者。

#### 异步之生产者/消费者模式

* 不需要产生结果和小费结果的线程一一对应。
* 消费队列可以用来平衡生产和消费的线程资源。
* 生产者仅负责产生结果数据，不关心消息如何处理。消费者专心处理结果数据。
* 消息队列是有容量限制的，满了就不会在加入新的，空时不会再消耗数据。
* JDK中各种阻塞队列，采用的就是这种模式

![](<../../../.gitbook/assets/Screen Shot 2022-02-07 at 10.46.27 PM.png>)

#### Park & Unpark

LockSupport 的方法

LockSupport.park(); 暂停线程

LockSupport.unpark(Thread t1); 恢复线程

不必配合Object Monitor 一起使用

可以提前unpart(Thread t1).

* 每个线程都有自己的Parker对象，由三个部分组成 counter, cond, mutex
* counter  0 / 1&#x20;
* 调用park， 查看counter，要是为0 就停止；要是为1 就继续，并设置counter为0。
* counter最多为1 最少为0

![](<../../../.gitbook/assets/Screen Shot 2022-02-08 at 12.15.58 AM.png>)

### 6. 线程状态转换

![](<../../../.gitbook/assets/Screen Shot 2022-02-08 at 12.21.46 AM.png>)

1. New -> Runnable  当调用t.start()方法时, 由new -> Runnable
2. Runnable -> Waiting  t调用synchronized(obj) 获得锁之后
   1. 调用obj.wait() 方法，t线程从Runnable -> Waiting
   2. 调用obj.notify()， obj.notifyAll(), t.interrupt()时
      1. 竞争锁成功，t从waiting -> Runnable
      2. 竞争锁失败， t 从 waiting -> Blocked
3. &#x20; Runnable -> Waiting 当前线程调用t.join()
   1. 当前线程在t对象的Monitor监视器上等待
   2. t线程运行结束，或者调用了当前线程的Interrupt() 时，当前线程从WAITING -> RUNNABLE
4. &#x20;Runnale -> Waiting 当前线程调用LockSupport.park()
   1. 调用LockSupport.unpark(t1) 或者调用interrupt时，会Waiting->Runnable
5.  Runnable -> Timed\_Waiting t线程用synchronized获取了对象锁后

    1. &#x20;调用obj.wait(long n ) 方法，t线程从Runnable -> Timed\_Waiting
    2. t线程等待时间超过了n秒，或者调用了notify,notifyAll,t.interrupt()时
       1. 竞争成功， -> Runnable
       2. 竞争失败 -> Blocked


6. t.join(long n)
7. Thread.sleep(n)
8. LockSupport.parkNanos(long nanos)
9. Runnable <-> Blocked
   1. t 线程用 synchronized(obj) 获取了对象锁时，如果竞争失败，从Runnable->Blocked
   2. 持有obj锁线程的代码同步块执行完毕，会唤醒该对象上所有的Blocked的线程重新竞争，如果t线程竞争成功，从Blocked -> Runnable, 其他失败的线程仍然Blocked
10. 线程结束

#### 多把锁

* 多把不相关的锁 要是一个线程请求多个锁，可能会造成死锁。

### 7. 活跃性/ 锁

#### 死锁

t1 获得了A 想获得B

t2 获得了B 想获得A

#### 定位死锁

可以用jconsole 工具

也可以用jsp 查看进程号，然后查看进程

#### 活锁

两个线程互相改变对方的结束条件，最后谁也无法结束。

#### 线程饥饿

线程优先级太低，始终得不到CPU调度，也不能够结束。读写锁时会涉及饥饿问题。

#### 哲学家问题

![](<../../../.gitbook/assets/Screen Shot 2022-02-08 at 5.08.34 PM.png>)

#### Reentrant Lock  可重入锁

相对与synchronized，它具备如下特点

* 可中断 synchronized 不可中断
* 可以设置超时时间 synchronized 会一直等待
* 可以设置为公平锁  防止线程饥饿 fifo 而不是随机强
* 设置多个条件变量  waitset 可以有多个

与synchronized一样 可以支持可重入 同一个线程 自己反复加锁

```
reentrantLock.lock()
try{

} finally {
    retrantLock.unlock();
}

```
