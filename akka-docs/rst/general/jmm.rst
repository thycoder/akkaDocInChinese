.. _jmm:

Akka和Java内存模型
================================

使用包含Scala和Akka的Typesafe Stack的主要好处是，它简化了编写并发软件的过程。本文将讨论Typesafe Stack，特别是Akka，是如何在并发应用中访问共享内存的。

Java内存模型
---------------------
在Java 5之前，Java内存模型（JMM）的定义是不清晰的。当共享内存被多个线程访问时，可能会得到各种各样的奇怪结果，例如：

* 一个线程看不到其它线程所写入的值：可见性问题
* 由于指令没有按期望的顺序执行，一个线程观察到其它线程的 ‘不可能’ 行为：指令乱序问题
随着Java 5对JSR 133的实现，这些问题很多都被解决了。 JMM是一组基于 “happens-before(在……之前发生)” 关系的规则, 限制了何时一个内存访问行为必须在另一个之前发生，以及相反地，它们何时能够乱序地发生。这些规则的两个示例如下：

* **监视器锁规则：** 对一个锁的释放，发生于对同一个锁的所有后续获取操作之前
* **volatile变量规则:** 对一个volatile变量的写操作，发生于对同一volatile变量的所有后续读取操作之前
尽管JMM看起来很复杂，但是这一规范试图在易用性和编写高性能可伸缩的并发数据结构的能力之间找到一个平衡点。

Actors 与 Java 内存模型
--------------------------------
使用Akka中所实现的Actor，多个线程在共享内存上执行操作有两种方式：

* 当一条消息被发送到一个actor（例如，从另一个actor）。大多数情况下消息是不可变的，但是如果这条消息不是一个被正确创建的不可变对象，假如没有 “发生先于” 规则,则接收方可能会看到部分初始化的数据结构，甚至有可能看到凭空出现的数据（long/double）。
* 当一个actor在处理消息时更改了自己的内部状态，而之后又在后续处理其它消息时访问了这一状态。需要意识到的重要一点是，在actor模型中，你并未得到关于同一个actor处理不同的消息时会在同一个线程中执行的保证。
为了避免actor中的可见性和乱序问题，Akka保证以下两条 “happens before” 规则:

*  **The actor发送规则:** 将一条消息发送到一个actor，发生于同一个actor接收那条消息之前。
*  **The actor后续处理规则:** 处理一条消息，发生于同一个actor处理下一条消息之前。

.. note::

    按照外行能够看懂的术语，这些规则的含义是，actor内部状态的变化在那个actor处理下一条消息之时是可见的。因此你的actor中的字段不需要是volatile的，或者等效的(equivalent)的。

这两条规则都只适用于同一个actor实例，如何使用了不同的actor则无效。

Futures 与 Java内存模型
---------------------------------

一个Future的完成，发生于任何注册到它的回调函数的执行之前。

我们建议不要捕获（close over）非final的字段 (Java中的final，Scala中的val), 如果你 *确实* 选择了捕捉非final的字段, 则它们必须被标记为 *volatile*，从而使其当前值对回调函数可见。

如果你捕获一个引用, 你还必须确保其所指向的实例是线程安全的。 我们强烈建议远离那些使用了锁的对象，因为这会引入性能问题，并且在最坏的情况下会导致死锁。 这些是synchronized的危险之处。

STM 与 Java内存模型
-----------------------------
Akka中的软件事务性内存 (STM) 也提供了一条 “happens before” 规则:

*  **事务性引用规则:**: 在事务提交过程中在一个事务性引用上的一次成功的写操作，发生于对同一事务性引用的所有后续读操作之前。
这条规则根JMM中的 ‘volatile 变量’ 规则看上去非常相似. 目前Akka STM仅支持延迟写操作，所以对共享内存的实际写操作会被延迟到事务提交之时。事务期间的写操作被存放于一个本地缓冲区中 (事务的写操作集合)，对其它的事务不可见。这就是脏读不可能发生的原因。

这些规则在Akka中的实现方式是一个实现细节，它会随时间而变化，确切的细节甚至可能依赖于所使用的配置。但是它们将建立在其它的JMM规则的基础之上，如监视器锁规则和volatile变量规则。这意味着Akka用户不需要关心为了提供这样的“happens before”关系而增加同步，因为这是Akka的职责。这样你就可以腾出手来处理业务逻辑，而Akka框架代表你保证这些规则的满足。

.. _jmm-shared-state:

Actor与共享可变状态
-------------------------------

因为Akka在JVM上运行，所以还有一些其它的规则需要遵守。

* 先捕获Actor内部状态，然后再将其暴露给其它线程

.. code-block:: scala

    class MyActor extends Actor {
     var state = ...
     def receive = {
        case _ =>
          //Wrongs

        // Very bad, shared mutable state,
        // will break your application in weird ways
          Future { state = NewState }
          anotherActor ? message onSuccess { r => state = r }

        // Very bad, "sender" changes for every message,
        // shared mutable state bug
          Future { expensiveCalculation(sender()) }

          //Rights

        // Completely safe, "self" is OK to close over
        // and it's an ActorRef, which is thread-safe
          Future { expensiveCalculation() } onComplete { f => self ! f.value.get }

        // Completely safe, we close over a fixed value
        // and it's an ActorRef, which is thread-safe
          val currentSender = sender()
          Future { expensiveCalculation(currentSender) }
     }
    }

* 消息 **应当**  是不可变的, 这是为了避开共享可变状态的陷阱。
