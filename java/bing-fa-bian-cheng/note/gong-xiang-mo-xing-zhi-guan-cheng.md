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
* 当其他线程使用偏向锁时，会将偏向锁升级为轻量级锁。



### 5. Wait/Notify

### 6. 线程状态转换

### 7. 活跃性

### 8. Lock
