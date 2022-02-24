# 共享模型之内存

### 1 Java 内存模型

对主存，工作内存抽象。

* 原子性
* 可见性
* 有序性

主存： 共享内存

工作内存： 线程使用的内存

#### 问题：

![](<../../../.gitbook/assets/Screen Shot 2022-02-08 at 10.41.08 PM.png>)

因为t线程要频繁的从主内存中读取run的值，JIT编译器会将run的值缓存至自己工作的高速缓存中，减少对主内存run的访问，提高效率。

但是主线程对run的修改，不会被t线程监控到

### 2 可见性

#### 解决方法

为变量加上 _volatile_ 关键字，变量不会再用缓存，保证从主存调用。

可以用来修饰成员变量和静态变量。局部变量是线程私有的，不存在这个问题。

加上synchronized也可以解决问题

volatile不能保证原子性

#### 模式之两阶段终止模式 （volatile）

* 回顾： 之前是通过 Thread.interrupt() 将 标识为true 然后通过while循环 判断标识为 来退出
* volatile: 代替之前interrupt 的 方法， 而是手动设置一个volatile标志， 来判断是否退出
* 两者也可以结合使用

同步模式之Balking&#x20;

通过标识位，在一个线程中发现另一个线程或者本线程已经做了某一件相同的事，那么本线程就无需再做了，直接返回结果。 也用于实现线程单例模式。

### 3 有序性

JVM 会在不影响正确性的前提下，调整语句的执行顺序

多线程指令重排会影响正确性

为什么会发生重排？ 有利于CPU流水线安排指令

只有在大量并发 测试下才会比较明显

* 给用到的变量加 volatile 标识  会禁止变量上面的代码进行重排

#### volatile原理 内存屏障 Memory Barrier

* 对写指令后加入写屏障
* 对读指令前加入度屏障

写屏障保证 该屏障之前的，对共享变量的修改，都同步到主内存

读屏障，该屏障之后的，对共享变量的读取，加载的是主存中最新数据



写屏障，保证不会将该代码之前的代码，排在写屏障之后

读屏障，保证读平屏障之后的代码，不会排到读屏障之前

等于设置了barrier, 限制了重排的范围

#### Double Checked locking 问题

原版

```
public final class Singleton{
    private Singleton() {}
    private static Singleton INSTANCE = null
    // 夹在类上面
    public static synchronized Singleton getInstance(){
        if(INSTANCE == null){
        return INSTANCE = new Singleton();
        }
        return INSTANCE;
    }
}
```

```
public final class Singleton{
    private Singleton() {}
    private static Singleton INSTANCE = null
    // 夹在类上面
    public static Singleton getInstance(){
        if(INSTANCE == null){
            synchronized(Singleton.class){
                if(INSTANCE == null){
                return INSTANCE = new Singleton();
                }
            }
        }
        return INSTANCE;
    }
}
```

* &#x20;懒汉实例化
* 首次使用getInstance 才会使用synchronized加锁，后续使用时无需加锁
* 第一个使用了if使用了INSTANCE变量，是在同步块之外 ？ 有序性？

new 函数 内可能会重排

1. 创建对象 将对象引用入栈   (只创建了对象 没有调用构造函数)
2. 复制一份对象引用 // 引用地址&#x20;
3. 利用对象一个引用，调用构造函数 // 根据引用地址调用
4. 表示利用一个对象引用，赋值给static INSTANCE

jvm 可以会对如上四个指令 进行重排。

要是先进行了 4 那么INSTANCE 已经不为null了 要是这时其他线程调用该方法，就会拿到半成品。**有代码逃逸出临界区**

#### Double Checked locking 解决

给 INSTANCE 加上volatile

设置写屏障 保证4 是最后执行的



```
public final class Singleton{
    private Singleton() {}
    private static volatile Singleton INSTANCE = null
    // 夹在类上面
    public static Singleton getInstance(){
        if(INSTANCE == null){
            synchronized(Singleton.class){
                if(INSTANCE == null){
                return INSTANCE = new Singleton();
                }
            }
        }
        return INSTANCE;
    }
}
```

#### Happends-before&#x20;

规定了堆共享变量的写操作对其它线程的读操作可见，它是可见性与有序性的一套规则的总结。

* 线程解锁m之前对变量的写，对于接下来对m加锁的其他线程堆该变量的读可见。
* 线程对volatile变量的写，堆接下来其他线程对该变量的读可见。
* 线程start前对变量的写，对接下来其他线程对该变量的读可见。
* t1打断t2前对变量的写，对其他线程得知t2被打断后对变量的读可见
* 对变量默认值的写，对其他变量的读可见
* 具有传递性（写屏障的 蔓延性），如果 x hb -> y , y hb -> z, 那么有 x hb -> z, 配合volatile 的防指令重排。

#### 线程安全单例习题

![](<../../../.gitbook/assets/Screen Shot 2022-02-09 at 2.33.04 AM.png>)

2\.  添加 方法&#x20;

```
public Object readResolve(){
    return INSTANCE;
}
```

3\. 防止其他类使用这个构造函数; 不能防止反射创建新实例

4\. 可以， 在类加载阶段就完成实现，线程安全。

5\. 解耦 为了以后支持泛型

![](<../../../.gitbook/assets/Screen Shot 2022-02-09 at 2.38.59 AM.png>)

1. 静态成员变量
2. **没有 静态成员变量**
3. 不会
4. 不会
5. 饿汉
6. 再写构造方法

![](<../../../.gitbook/assets/Screen Shot 2022-02-09 at 2.43.00 AM.png>)

1. 直接给类加锁，范围大

![](<../../../.gitbook/assets/Screen Shot 2022-02-09 at 2.45.01 AM.png>)

1. 防止 new 操作的重排，导致第一处!=null逻辑返回半成品的实例
2. 锁范围小
3. 首次创建时，多线程同时访问，可能会pass第一个INSTANCE，但是会被synchronzied阻塞住；当他们running时，可能INSTANCE已经被创建好了，所以直接返回就好了。

![](<../../../.gitbook/assets/Screen Shot 2022-02-09 at 2.50.46 AM.png>)

1. 懒汉
2. 由jvm来保证线程安全。
