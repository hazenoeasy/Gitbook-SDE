# Debug

Java Lamda 只能访问 显性 或者隐性的final 变量

[https://blog.csdn.net/qq\_42013590/article/details/106865079](https://blog.csdn.net/qq\_42013590/article/details/106865079)

[https://blog.csdn.net/single\_wolf\_wolf/article/details/105615471](https://blog.csdn.net/single\_wolf\_wolf/article/details/105615471)

[https://blog.csdn.net/qq564425/article/details/81369284](https://blog.csdn.net/qq564425/article/details/81369284)

1. &#x20;实例变量都存储在堆中，而局部变量则保存在栈上。&#x20;
2. 如果Lambda 可以直接访问局部变量，而且Lambda 是在一个线程中使用的，则使用 Lambda 的线程，可能会在分配该变量的线程将这个变量收回之后，去访问该变量。
3. 因此，Java 在访问自由局部变量时，实际上是在访问它的副本，而不是访问原始变量。
4. 如果局部变量仅仅赋值一次那就没有什么区别了——因此就有了这个限制。 实例变量可以的原因是，因为它们保存在堆中，而堆是在线程之间共享的。&#x20;
5. 第二，这一限制不鼓励你使用改变外部变量的典型命令式编程模式。&#x20;

