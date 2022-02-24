# 共享模型之不可变

### 7.1 日期转换的问题

问题提出，下面的代码在运行时，由于 SimpleDateFormat 不是线程安全的，有很大几率出现 `java.lang.NumberFormatException` 或者出现不正确的日期解析结果。

```java
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
        for (int i = 0; i < 10; i++) {
            new Thread(() -> {
                try {
                    log.debug("{}", sdf.parse("1951-04-21"));
                } catch (Exception e) {
                    log.error("{}", e);
                }
            }).start();
        }
```

思路 - 不可变对象

如果一个对象在不能够修改其内部状态（属性），那么它就是线程安全的，因为不存在并发修改啊！这样的对象在 Java 中有很多，例如在 Java 8 后，提供了一个新的日期格式化类DateTimeFormatter：

```java
        DateTimeFormatter dtf = DateTimeFormatter.ofPattern("yyyy-MM-dd");
        for (int i = 0; i < 10; i++) {
            new Thread(() -> {
                LocalDate date = dtf.parse("2018-10-01", LocalDate::from);
                log.debug("{}", date);
            }).start();
        }
```

### 7.2 不可变设计

> 1. 更多不可变类的知识，可参考这[这里](https://www.cnblogs.com/dolphin0520/p/10693891.html)
> 2. final类的知识，参考[这里](https://www.cnblogs.com/xiaoxi/p/6392154.html)

另一个大家更为熟悉的 String 类也是不可变的，以它为例，说明一下不可变类**设计的要素**

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];
    /** Cache the hash code for the string */
    private int hash; // Default to 0
    // ...
}
```

#### final 的使用

发现该类、类中所有属性都是 final 的，属性用 final 修饰保证了该属性是只读的，不能修改，类用 final 修饰保证了该类中的方法不能被覆盖，防止子类无意间破坏不可变性。

#### 保护性拷贝

但有同学会说，使用字符串时，也有一些跟修改相关的方法啊，比如 substring 等，那么下面就看一看这些方法是 如何实现的，就以 substring 为例：

```java
    public String substring(int beginIndex, int endIndex) {
        if (beginIndex < 0) {
            throw new StringIndexOutOfBoundsException(beginIndex);
        }
        if (endIndex > value.length) {
            throw new StringIndexOutOfBoundsException(endIndex);
        }
        int subLen = endIndex - beginIndex;
        if (subLen < 0) {
            throw new StringIndexOutOfBoundsException(subLen);
        }
        // 上面是一些校验，下面才是真正的创建新的String对象
        return ((beginIndex == 0) && (endIndex == value.length)) ? this
                : new String(value, beginIndex, subLen);
    }
```

发现其内部是调用 String 的构造方法创建了一个新字符串，再进入这个构造看看，是否对 final char\[] value 做出 了修改：结果发现也没有，构造新字符串对象时，会生成新的 char\[] value，对内容进行复制。这种通过创建副本对象来避免共享的手段称之为【保护性拷贝（defensive copy）】

```java
    public String(char value[], int offset, int count) {
        if (offset < 0) {
            throw new StringIndexOutOfBoundsException(offset);
        }
        if (count <= 0) {
            if (count < 0) {
                throw new StringIndexOutOfBoundsException(count);
            }
            if (offset <= value.length) {
                this.value = "".value;
                return;
            }
        }
        // Note: offset or count might be near -1>>>1.
        if (offset > value.length - count) {
            throw new StringIndexOutOfBoundsException(offset + count);
        }
        // 上面是一些安全性的校验，下面是给String对象的value赋值，新创建了一个数组来保存String对象的值
        this.value = Arrays.copyOfRange(value, offset, offset+count);
    }
```

#### 模式之享元

1. 简介定义英文名称：Flyweight pattern. 当需要重用数量有限的同一类对象时
2. 体现
   1.  在JDK中 Boolean，Byte，Short，Integer，Long，Character 等包装类提供了 valueOf 方法，例如 Long 的valueOf 会缓存 -128\~127 之间的 Long 对象，在这个范围之间会重用对象，大于这个范围，才会新建 Long 对象：

       ```java
       public static Long valueOf(long l) {
        final int offset = 128;
        if (l >= -128 && l <= 127) { // will cache
        return LongCache.cache[(int)l + offset];
        }
        return new Long(l);
       }
       ```

       > 注意： Byte, Short, Long 缓存的范围都是 -128~~127 Character 缓存的范围是 0~~127 Integer的默认范围是 -128\~127，最小值不能变，但最大值可以通过调整虚拟机参数 `"-Djava.lang.Integer.IntegerCache.high` "来改变 Boolean 缓存了 TRUE 和 FALSE
   2. String 串池
   3. BigDecimal BigInteger
3. diy:例如：一个线上商城应用，QPS 达到数千，如果每次都重新创建和关闭数据库连接，性能会受到极大影响。 这时预先创建好一批连接，放入连接池。一次请求到达后，从连接池获取连接，使用完毕后再还回连接池，这样既节约了连接的创建和关闭时间，也实现了连接的重用，不至于让庞大的连接数压垮数据库。 Test17.java

#### final的原理

1. 设置 final 变量的原理
   1.  理解了 volatile 原理，再对比 final 的实现就比较简单了

       ```java
       public class TestFinal {final int a=20;}
       ```

       字节码

       ```java
       0: aload_0
       1: invokespecial #1 // Method java/lang/Object."<init>":()V
       4: aload_0
       5: bipush 20
       7: putfield #2 // Field a:I
        <-- 写屏障
       10: return
       ```
   2. final变量的赋值操作都必须在定义时或者构造器中进行初始化赋值，并发现 final 变量的赋值也会通过 putfield 指令来完成，同样在这条指令之后也会加入写屏障，保证在其它线程读到它的值时不会出现为 0 的情况。
2. 获取 final 变量的原理：从字节码的层面去理解 [视频](https://www.bilibili.com/video/BV16J411h7Rd?p=197)。

### 7.3 本章小结

1. 不可变类使用
2. 不可变类设计
3. 原理方面：final
4. 模式方面
   1. 享元模式-> 设置线程池

## 问题

1. 什么时候将导致用户态到内核态的转变？在synchroniezed进行加锁的时候。
2. final是怎么优化读取速度的？复习完jvm再看就懂了。[视频](https://www.bilibili.com/video/BV16J411h7Rd?p=197)
