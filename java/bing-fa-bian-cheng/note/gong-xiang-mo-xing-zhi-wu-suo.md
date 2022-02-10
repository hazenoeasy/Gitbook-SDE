---
description: 管程即monitor是阻塞式的悲观锁实现并发控制，这章我们将通过非阻塞式的乐观锁的来实现并发控制
---

# 共享模型之无锁

### 6.1 问题提出

有如下需求，保证account.withdraw取款方法的线程安全 Test5.java

```java
public class Test5 {

    public static void main(String[] args) {
        Account.demo(new AccountUnsafe(10000));
    }
}

class AccountUnsafe implements Account {
    private Integer balance;
    public AccountUnsafe(Integer balance) {
        this.balance = balance;
    }
    @Override
    public Integer getBalance() {
        return balance;
    }
    @Override
    
    public void  withdraw(Integer amount) {
        // 通过这里加锁就可以实现线程安全，不加就会导致结果异常
        synchronized (this){
            balance -= amount;
        }
    }
}


interface Account {
    // 获取余额
    Integer getBalance();
    // 取款
    void withdraw(Integer amount);
    /**
     * 方法内会启动 1000 个线程，每个线程做 -10 元 的操作
     * 如果初始余额为 10000 那么正确的结果应当是 0
     */
    static void demo(Account account) {
        List<Thread> ts = new ArrayList<>();
        long start = System.nanoTime();
        for (int i = 0; i < 1000; i++) {
            ts.add(new Thread(() -> {
                account.withdraw(10);
            }));
        }
        ts.forEach(Thread::start);
        ts.forEach(t -> {
            try {
                t.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        long end = System.nanoTime();
        System.out.println(account.getBalance()
                + " cost: " + (end-start)/1000_000 + " ms");
    }
}
```

#### 解决思路-无锁

上面的代码中可以使用synchronized加锁操作来实现线程安全，但是synchronized加锁操作太耗费资源，这里我们使用无锁来解决此问题：  依托于AtomicInteger.

```java
class AccountSafe implements Account{

    AtomicInteger atomicInteger ;
    
    public AccountSafe(Integer balance){
        this.atomicInteger =  new AtomicInteger(balance);
    }
    
    @Override
    public Integer getBalance() {
        return atomicInteger.get();
    }

    @Override
    public void withdraw(Integer amount) {
        // 核心代码
        while (true){
            int pre = getBalance();
            int next = pre - amount;
            if (atomicInteger.compareAndSet(pre,next)){
                break;
            }
        }
        // 可以简化为下面的方法
        // balance.addAndGet(-1 * amount);
    }
}
```

### 6.2 CAS 与 volatile

#### cas compare and swap  cpu层面 原子操作

前面看到的AtomicInteger的解决方法，内部并没有用锁来保护共享变量的线程安全。那么它是如何实现的呢？

```java
    @Override
    public void withdraw(Integer amount) {
        // 核心代码
        // 需要不断尝试，直到成功为止
        while (true){
            // 比如拿到了旧值 1000
            int pre = getBalance();
            // 在这个基础上 1000-10 = 990
            int next = pre - amount;
            /*
             compareAndSet 正是做这个检查，在 set 前，先比较 prev 与当前值
             - 不一致了，next 作废，返回 false 表示失败
             比如，别的线程已经做了减法，当前值已经被减成了 990
             那么本线程的这次 990 就作废了，进入 while 下次循环重试
             - 一致，以 next 设置为新值，返回 true 表示成功
			 */
            if (atomicInteger.compareAndSet(pre,next)){
                break;
            }
        }
    }
```

其中的关键是 compareAndSet，它的简称就是 CAS （也有 Compare And Swap 的说法），它必须是原子操作。

在IA64，x86 指令集中有 _**cmpxchg**_ 指令完成 CAS 功能，在 sparc-TSO 也有 _**casa**_ 指令实现，而在 ARM 和 PowerPC 架构下，则需要使用一对 _**ldrex/strex**_ 指令来完成 LL/SC 的功能。

通过**缓存锁定**来保证原子性。所谓缓存锁定是指内存区域如果被缓存在处理器的缓存行中，并且在 _Lock_ 操作期间被锁定，那么当他执行锁操作写回到内存时，处理器不在总线上声言 _LOCK#_ 信号，而是修改内部的内存地址，并允许他的缓存一致性机制来保证操作的原子性，因为缓存一致性机制会阻止同时修改两个以上处理器缓存的内存区域数据（这里和 volatile 的可见性原理相同），当其他处理器回写已被锁定的缓存行的数据时，会使缓存行无效。

![1594776811158](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20200715093333-972226.png)

#### volatile

在上面代码中的AtomicInteger，保存值的value属性使用了volatile 。获取共享变量时，为了保证该变量的可见性，需要使用 volatile 修饰。

它可以用来修饰**成员变量和静态成员变量**，他可以避免线程从自己的工作缓存中查找变量的值，必须到主存中获取 它的值，线程操作 volatile 变量都是直接操作主存。即一个线程对 volatile 变量的修改，对另一个线程可见。

> 再提一嘴 volatile 仅仅保证了共享变量的可见性，让其它线程能够看到最新值，但不能解决指令交错问题（不能保证原 子性）

CAS 必须借助 volatile 才能读取到共享变量的最新值来实现【比较并交换】的效果

#### 为什么无锁效率高

1. 无锁情况下，即使重试失败，线程始终在高速运行，没有停歇，而 synchronized 会让线程在没有获得锁的时候，发生上下文切换，进入阻塞。打个比喻：线程就好像高速跑道上的赛车，高速运行时，速度超快，一旦发生上下文切换，就好比赛车要减速、熄火，等被唤醒又得重新打火、启动、加速... 恢复到高速运行，代价比较大
2. 但无锁情况下，因为线程要保持运行，需要额外 CPU 的支持，CPU 在这里就好比高速跑道，没有额外的跑道，线程想高速运行也无从谈起，虽然不会进入阻塞，但由于没有分到时间片，仍然会进入可运行状态，还是会导致上下文切换。

#### CAS 的特点

结合 CAS 和 volatile 可以实现无锁并发，适用于线程数少、多核 CPU 的场景下。

1. CAS 是基于乐观锁的思想：最乐观的估计，不怕别的线程来修改共享变量，就算改了也没关系，我吃亏点再重试呗。
2. synchronized 是基于悲观锁的思想：最悲观的估计，得防着其它线程来修改共享变量，我上了锁你们都别想改，我改完了解开锁，你们才有机会。
3. CAS 体现的是无锁并发、无阻塞并发，请仔细体会这两句话的意思
   1. 因为没有使用 synchronized，所以线程不会陷入阻塞，这是效率提升的因素之一
   2. 但如果竞争激烈(写操作多)，可以想到重试必然频繁发生，反而效率会受影响

### 6.3原子整数

java.util.concurrent.atomic并发包提供了一些并发工具类，这里把它分成五类：

1.  使用原子的方式更新基本类型

    * AtomicInteger：整型原子类
    * AtomicLong：长整型原子类
    * AtomicBoolean ：布尔型原子类

    上面三个类提供的方法几乎相同，所以我们将以 AtomicInteger 为例子来介绍。
2. 原子引用
3. 原子数组
4. 字段更新器
5. 原子累加器

下面先讨论原子整数类，以 AtomicInteger 为例讨论它的api接口：通过观察源码可以发现，AtomicInteger 内部都是通过cas的原理来实现的！！好像发现了新大陆！ Test6.java

```java
    public static void main(String[] args) {
        AtomicInteger i = new AtomicInteger(0);
        // 获取并自增（i = 0, 结果 i = 1, 返回 0），类似于 i++
        System.out.println(i.getAndIncrement());
        // 自增并获取（i = 1, 结果 i = 2, 返回 2），类似于 ++i
        System.out.println(i.incrementAndGet());
        // 自减并获取（i = 2, 结果 i = 1, 返回 1），类似于 --i
        System.out.println(i.decrementAndGet());
        // 获取并自减（i = 1, 结果 i = 0, 返回 1），类似于 i--
        System.out.println(i.getAndDecrement());
        // 获取并加值（i = 0, 结果 i = 5, 返回 0）
        System.out.println(i.getAndAdd(5));
        // 加值并获取（i = 5, 结果 i = 0, 返回 0）
        System.out.println(i.addAndGet(-5));
        // 获取并更新（i = 0, p 为 i 的当前值, 结果 i = -2, 返回 0）
        // 函数式编程接口，其中函数中的操作能保证原子，但函数需要无副作用
        System.out.println(i.getAndUpdate(p -> p - 2));
        // 更新并获取（i = -2, p 为 i 的当前值, 结果 i = 0, 返回 0）
        // 函数式编程接口，其中函数中的操作能保证原子，但函数需要无副作用
        System.out.println(i.updateAndGet(p -> p + 2));
        // 获取并计算（i = 0, p 为 i 的当前值, x 为参数1, 结果 i = 10, 返回 0）
        // 函数式编程接口，其中函数中的操作能保证原子，但函数需要无副作用
        // getAndUpdate 如果在 lambda 中引用了外部的局部变量，要保证该局部变量是 final 的
        // getAndAccumulate 可以通过 参数1 来引用外部的局部变量，但因为其不在 lambda 中因此不必是 final
        System.out.println(i.getAndAccumulate(10, (p, x) -> p + x));
        // 计算并获取（i = 10, p 为 i 的当前值, x 为参数1值, 结果 i = 0, 返回 0）
        // 函数式编程接口，其中函数中的操作能保证原子，但函数需要无副作用
        System.out.println(i.accumulateAndGet(-10, (p, x) -> p + x));
    }
    
```

### 6.4 原子引用  对象地址的原子性

为什么需要原子引用类型？保证引用类型的共享变量是线程安全的（确保这个原子引用没有引用过别人）。

基本类型原子类只能更新一个变量，如果需要原子更新多个变量，需要使用引用类型原子类。

* AtomicReference：引用类型原子类
* AtomicStampedReference：原子更新带有版本号的引用类型。该类将整数值与引用关联起来，可用于解决原子的更新数据和数据的版本号，可以解决使用 CAS 进行原子更新时可能出现的 ABA 问题。
* AtomicMarkableReference ：原子更新带有标记的引用类型。该类将 boolean 标记与引用关联起来，~~也可以解决使用 CAS 进行原子更新时可能出现的 ABA 问题。~~

使用原子引用实现BigDecimal存取款的线程安全：Test7.java

下面这个是不安全的实现过程：

```java
class DecimalAccountUnsafe implements DecimalAccount {
    BigDecimal balance;
    public DecimalAccountUnsafe(BigDecimal balance) {
        this.balance = balance;
    }
    @Override
    public BigDecimal getBalance() {
        return balance;
    }
    @Override
    public void withdraw(BigDecimal amount) {
        BigDecimal balance = this.getBalance();
        this.balance = balance.subtract(amount);
    }
}
```

解决代码如下：在AtomicReference类中，存在一个value类型的变量，保存对BigDecimal对象的引用。

```java
class DecimalAccountCas implements DecimalAccount{

    //private BigDecimal balance;
    private AtomicReference<BigDecimal> balance ;

    public DecimalAccountCas(BigDecimal balance) {
        this.balance = new AtomicReference<>(balance);
    }

    @Override
    public BigDecimal getBalance() {
        return balance.get();
    }

    @Override
    public void withdraw(BigDecimal amount) {
        while(true){
            BigDecimal pre = balance.get();
            // 注意：这里的balance返回的是一个新的对象，即 pre!=next
            BigDecimal next = pre.subtract(amount);
            if (balance.compareAndSet(pre,next)){
                break;
            }
        }
    }
}
```

#### ABA 问题及解决

ABA 问题：Test8.java 如下程序所示，虽然再other方法中存在两个线程对共享变量进行了修改，但是修改之后又变成了原值，main线程中对此是不可见得，这种操作这对业务代码并无影响。

```java
    static AtomicReference<String> ref = new AtomicReference<>("A");
    public static void main(String[] args) throws InterruptedException {
        log.debug("main start...");
        // 获取值 A
        // 这个共享变量被它线程修改
        String prev = ref.get();
        other();
        utils.sleep(1);
        // 尝试改为 C
        log.debug("change A->C {}", ref.compareAndSet(prev, "C"));
    }
    private static void other() {
        new Thread(() -> {
            log.debug("change A->B {}", ref.compareAndSet(ref.get(), "B"));
        }, "t1").start();
        utils.sleep(1);
        new Thread(() -> {
            // 注意：如果这里使用  log.debug("change B->A {}", ref.compareAndSet(ref.get(), new String("A")));
            // 那么此实验中的 log.debug("change A->C {}", ref.compareAndSet(prev, "C"));
            // 打印的就是false， 因为new String("A") 返回的对象的引用和"A"返回的对象的引用时不同的！
            log.debug("change B->A {}", ref.compareAndSet(ref.get(), "A"));
        }, "t2").start();
    }
```

主线程仅能判断出共享变量的值与最初值 A 是否相同，不能感知到这种从 A 改为 B 又改回 A 的情况，如果主线程希望：只要有其它线程【动过了】共享变量，那么自己的 cas 就算失败，这时，仅比较值是不够的，需要再加一个版本号。使用AtomicStampedReference来解决。·

#### AtomicStampedReference

解决ABA问题 Test9.java

```java
static AtomicStampedReference<String> ref = new AtomicStampedReference<>("A",0);
    public static void main(String[] args) throws InterruptedException {
        log.debug("main start...");
        // 获取值 A
        int stamp = ref.getStamp();
        log.info("{}",stamp);
        String prev = ref.getReference();
        other();
        utils.sleep(1);
        // 尝试改为 C
        log.debug("change A->C {}", ref.compareAndSet(prev, "C",stamp,stamp+1));
    }
    
    private static void other() {
        new Thread(() -> {
            int stamp = ref.getStamp();
            log.info("{}",stamp);
            log.debug("change A->B {}", ref.compareAndSet(ref.getReference(), "B",stamp,stamp+1));
        }, "t1").start();
        utils.sleep(1);
        new Thread(() -> {
            int stamp = ref.getStamp();
            log.info("{}",stamp);
            log.debug("change B->A {}", ref.compareAndSet(ref.getReference(), "A",stamp,stamp+1));
        }, "t2").start();
    }
```

#### AtomicMarkableReference

AtomicStampedReference 可以给原子引用加上版本号，追踪原子引用整个的变化过程，如：A -> B -> A ->C，通过AtomicStampedReference，我们可以知道，引用变量中途被更改了几次。但是有时候，并不关心引用变量更改了几次，只是单纯的关心是否更改过，所以就有了AtomicMarkableReference Test10.java

### 6.5 原子数组

使用原子的方式更新数组里的某个元素

* AtomicIntegerArray：整形数组原子类
* AtomicLongArray：长整形数组原子类
* AtomicReferenceArray ：引用类型数组原子类

上面三个类提供的方法几乎相同，所以我们这里以 AtomicIntegerArray 为例子来介绍。实例代码：Test11.java

我们将使用函数式编程来实现，先看看一些函数式编程的接口的javadoc文档

```java
Represents a supplier of results.
表示supplier的结果。
There is no requirement that a new or distinct result be returned each time the supplier is invoked.
不要求每次调用供应商时都返回一个新的或不同的结果。
This is a functional interface whose functional method is get().
这是一个函数接口，其函数方法是get（）。
public interface Supplier<T> {
    /**
     * Gets a result.
     * @return a result
     */
    T get();
}

Represents a function that accepts one argument and produces a result.
表示接受一个参数并生成结果的函数。
This is a functional interface whose functional method is apply(Object).
这是一个函数接口，其函数方法是apply（Object）。
public interface Function<T, R> {
/**
* Applies this function to the given argument.
* @param t the function argument
* @return the function result
*/	
  R apply(T t);
  //....
  }
  
  Represents an operation that accepts two input arguments and returns no result. This is the two-arity specialization of Consumer. Unlike most other functional interfaces, BiConsumer is expected to operate via side-effects.
表示接受两个输入参数且不返回结果的操作。这就是Consumer的二元参数版本。与大多数其他功能性接口不同，BiConsumer期望执行带有副作用的操作。
This is a functional interface whose functional method is accept(Object, Object).
这是一个函数接口，其函数方法是accept（Object，Object）。
public interface BiConsumer<T, U> {
   void accept(T t, U u);
     //....
  }



Represents an operation that accepts a single input argument and returns no result. Unlike most other functional interfaces, Consumer is expected to operate via side-effects.
表示接受单个输入参数但不返回结果的操作。与大多数其他功能接口不同，消费者期望执行带有副作用的操作。 
public interface Consumer<T> {
void accept(T t);
   //....
  }
```

### 6.6 字段更新器 对属性进行更新&#x20;

AtomicReferenceFieldUpdater // 域 字段 ，AtomicIntegerFieldUpdater，AtomicLongFieldUpdater

注意：利用字段更新器，可以针对对象的某个域（Field）进行原子操作，只能配合 volatile 修饰的字段使用，否则会出现异常，因为cas要配合volatile使用,因为cas要确定比较最新值。

Atomic类是内部给属性加了volatile

而updater是对非Atomic类的属性，进行无锁更新。所以需要使用者手动给属性添加volatile

```
Exception in thread "main" java.lang.IllegalArgumentException: Must be volatile type
```

### 6.7 原子累加器

#### 累加器性能比较  原子累加器比Atomic效率高

LongAdder累加器的使用

```java
    public static void main(String[] args) {
        for (int i = 0; i < 5; i++) {
            demo(() -> new LongAdder(), adder -> adder.increment());
        }
        for (int i = 0; i < 5; i++) {
            demo(() -> new AtomicLong(), adder -> adder.getAndIncrement());
        }

    }
    
    private static <T> void demo(Supplier<T> adderSupplier, Consumer<T> action) {
        T adder = adderSupplier.get();
        long start = System.nanoTime();
        List<Thread> ts = new ArrayList<>();
        // 4 个线程，每人累加 50 万
        for (int i = 0; i < 40; i++) {
            ts.add(new Thread(() -> {
                for (int j = 0; j < 500000; j++) {
                    action.accept(adder);
                }
            }));
        }
        ts.forEach(t -> t.start());
        ts.forEach(t -> {
            try {
                t.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        long end = System.nanoTime();
        System.out.println(adder + " cost:" + (end - start)/1000_000);
    }
```

性能提升的原因很简单，就是在有竞争时，设置多个累加单元(但不会超过cpu的核心数)，Therad-0 累加 Cell\[0]，而 Thread-1 累加Cell\[1]... 最后将结果汇总。这样它们在累加时操作的不同的 Cell 变量，因此减少了 CAS 重试失败，从而提高性能。

#### 源码之 LongAdder

LongAdder 类有几个关键域

```java
// 累加单元数组, 懒惰初始化
transient volatile Cell[] cells;
// 基础值, 如果没有竞争, 则用 cas 累加这个域
transient volatile long base;
// 在 cells 创建或扩容时, 置为 1, 表示加锁
transient volatile int cellsBusy;
```

**cas 锁**

使用cas实现一个自旋锁

```java
// 不要用于生产实践！！！
public class LockCas {
    private AtomicInteger state = new AtomicInteger(0);
    public void lock() {
        while (true) {
            if (state.compareAndSet(0, 1)) {
                break;
            }
        }
    }
    public void unlock() {
        log.debug("unlock...");
        state.set(0);
    }
}
```

测试

```java
        LockCas lock = new LockCas();
        new Thread(() -> {
            log.debug("begin...");
            lock.lock();
            try {
                log.debug("lock...");
                sleep(1);
            } finally {
                lock.unlock();
            }
        }).start();
        new Thread(() -> {
            log.debug("begin...");
            lock.lock();
            try {
                log.debug("lock...");
            } finally {
                lock.unlock();
            }
        }).start();
```

**原理之伪共享**

其中 Cell 即为累加单元

```java
// 防止缓存行伪共享
@sun.misc.Contended
static final class Cell {
    volatile long value;
    Cell(long x) { value = x; }
    // 最重要的方法, 用来 cas 方式进行累加, prev 表示旧值, next 表示新值
    final boolean cas(long prev, long next) {
        return UNSAFE.compareAndSwapLong(this, valueOffset, prev, next);
    }
    // 省略不重要代码
}
```

下面讨论@sun.misc.Contended注解的重要意义

得从缓存说起，缓存与内存的速度比较

![1594821128208](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20200715215209-330421.png)

因为 CPU 与 内存的速度差异很大，需要靠预读数据至缓存来提升效率。缓存离cpu越近速度越快。 而缓存以缓存行为单位，每个缓存行对应着一块内存，一般是 64 byte（8 个 long），缓存的加入会造成数据副本的产生，即同一份数据会缓存在不同核心的缓存行中，CPU 要保证数据的一致性，如果某个 CPU 核心更改了数据，其它 CPU 核心对应的整个缓存行必须失效。

![1594821188387](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20200716135941-626948.png)

因为 Cell 是数组形式，在内存中是连续存储的，一个 Cell 为 24 字节（16 字节的对象头和 8 字节的 value），因 此缓存行可以存下 2 个的 Cell 对象。这样问题来了： Core-0 要修改 Cell\[0]，Core-1 要修改 Cell\[1]

无论谁修改成功，都会导致对方 Core 的缓存行失效，比如 Core-0 中 Cell\[0]=6000, Cell\[1]=8000 要累加 Cell\[0]=6001, Cell\[1]=8000 ，这时会让 Core-1 的缓存行失效，@sun.misc.Contended 用来解决这个问题，它的原理是在使用此注解的对象或字段的前后各增加 128 字节大小的padding，从而让 CPU 将对象预读至缓存时占用不同的缓存行，这样，不会造成对方缓存行的失效

![1594821225708](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20200716135939-234417.png)

再来看看LongAdder类的累加increment()方法中又主要调用下面的方法

```java
  public void add(long x) {
        // as 为累加单元数组
        // b 为基础值
        // x 为累加值
        Cell[] as; long b, v; int m; Cell a;
        // 进入 if 的两个条件
        // 1. as 有值, 表示已经发生过竞争, 进入 if
        // 2. cas 给 base 累加时失败了, 表示 base 发生了竞争, 进入 if
        if ((as = cells) != null || !casBase(b = base, b + x)) {
            // uncontended 表示 cell 没有竞争
            boolean uncontended = true;
            if (
                // as 还没有创建
                    as == null || (m = as.length - 1) < 0 ||
                            // 当前线程对应的 cell 还没有被创建，a为当线程的cell
                            (a = as[getProbe() & m]) == null ||
       // 给当前线程的 cell 累加失败 uncontended=false ( a 为当前线程的 cell )
                            !(uncontended = a.cas(v = a.value, v + x))
            ) {
                // 进入 cell 数组创建、cell 创建的流程
                longAccumulate(x, null, uncontended);
            }
        }
    }
```

**add 方法分析**

add 流程图

```java
  final void longAccumulate(long x, LongBinaryOperator fn,
                              boolean wasUncontended) {
        int h;
        // 当前线程还没有对应的 cell, 需要随机生成一个 h 值用来将当前线程绑定到 cell
        if ((h = getProbe()) == 0) {
            // 初始化 probe
            ThreadLocalRandom.current();
            // h 对应新的 probe 值, 用来对应 cell
            h = getProbe();
            wasUncontended = true;
        }
        // collide 为 true 表示需要扩容
        boolean collide = false;
        for (;;) {
            Cell[] as; Cell a; int n; long v;
            // 已经有了 cells
            if ((as = cells) != null && (n = as.length) > 0) {
                // 但是还没有当前线程对应的 cell
                if ((a = as[(n - 1) & h]) == null) {
                    // 为 cellsBusy 加锁, 创建 cell, cell 的初始累加值为 x
                    // 成功则 break, 否则继续 continue 循环
                    if (cellsBusy == 0) {       // Try to attach new Cell
                        Cell r = new Cell(x);   // Optimistically create
                        if (cellsBusy == 0 && casCellsBusy()) {
                            boolean created = false;
                            try {               // Recheck under lock
                                Cell[] rs; int m, j;
                                if ((rs = cells) != null &&
                                    (m = rs.length) > 0 &&
                                    // 判断槽位确实是空的
                                    rs[j = (m - 1) & h] == null) {
                                    rs[j] = r;
                                    created = true;
                                }
                            } finally {
                                cellsBusy = 0;
                            }
                            if (created)
                                break;
                            continue;           // Slot is now non-empty
                        }
                }
                // 有竞争, 改变线程对应的 cell 来重试 cas
                else if (!wasUncontended)
                    wasUncontended = true;
                    // cas 尝试累加, fn 配合 LongAccumulator 不为 null, 配合 LongAdder 为 null
                else if (a.cas(v = a.value, ((fn == null) ? v + x : fn.applyAsLong(v, x))))
                    break;
                    // 如果 cells 长度已经超过了最大长度, 或者已经扩容, 改变线程对应的 cell 来重试 cas
                else if (n >= NCPU || cells != as)
                    collide = false;
                    // 确保 collide 为 false 进入此分支, 就不会进入下面的 else if 进行扩容了
                else if (!collide)
                    collide = true;
                    // 加锁
                else if (cellsBusy == 0 && casCellsBusy()) {
                    // 加锁成功, 扩容
                    continue;
                }
                // 改变线程对应的 cell
                h = advanceProbe(h);
            }
            // 还没有 cells, cells==as是指没有其它线程修改cells，as和cells引用相同的对象，使用casCellsBusy()尝试给 cellsBusy 加锁
            else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
                // 加锁成功, 初始化 cells, 最开始长度为 2, 并填充一个 cell
                // 成功则 break;
                boolean init = false;
                try {                           // Initialize table
                    if (cells == as) {
                        Cell[] rs = new Cell[2];
                        rs[h & 1] = new Cell(x);
                        cells = rs;
                        init = true;
                    }
                } finally {
                    cellsBusy = 0;
                }
                if (init)
                    break;
            }
            // 上两种情况失败, 尝试给 base 使用casBase累加
            else if (casBase(v = base, ((fn == null) ? v + x : fn.applyAsLong(v, x))))
                break;
        }
    }
```

上图中的第一个else if 中代码的逻辑，这是cells未创建时的处理逻辑。

![1594825327345](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20200715230208-829688.png)

上图中的if 中代码的逻辑，里面包含线程对应的cell已经创建好和没创建好的两种情况。

线程对应的cell还没创建好，则执行的是第一个红框里的代码，逻辑如下

![1594826082009](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20200715231442-249345.png)

线程对应的cell已经创建好，则执行的是第二个红框里的代码，逻辑如下

![1594826936970](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20200715232859-416098.png)

**sum 方法分析**

获取最终结果通过 sum 方法，将各个累加单元的值加起来就得到了总的结果。

```java
    public long sum() {
        Cell[] as = cells; Cell a;
        long sum = base;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    sum += a.value;
            }
        }
        return sum;
    }
```

### 6.8 Unsafe

#### 概述

Unsafe 对象提供了非常底层的，操作内存、线程的方法，Unsafe 对象不能直接调用，只能通过反射获得。LockSupport的park方法，cas相关的方法底层都是通过Unsafe类来实现的。Test14.java

```java
    static Unsafe unsafe;
    static {
        try {
            // Unsafe 使用了单例模式，unsafe对象是类中的一个私有的变量
            Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
            theUnsafe.setAccessible(true);
            unsafe = (Unsafe) theUnsafe.get(null);
        } catch (NoSuchFieldException | IllegalAccessException e) {
            throw new Error(e);
        }
    }
    static Unsafe getUnsafe() {
        return unsafe;
    }
```

#### Unsafe CAS 操作

1. 使用Unsafe 进行cas操作：Test15.java
2. 使用自定义的 AtomicData 实现之前线程安全的原子整数 ，并用之前的取款实例来进行验证 Test16.java

### 6.9总结

1. CAS 与 volatile
2. juc包下API
   1. 原子整数
   2. 原子引用
   3. 原子数组
   4. 字段更新器
   5. 原子累加器
3. Unsafe
4. 原理方面
   1. LongAdder 源码
   2. 伪共享
