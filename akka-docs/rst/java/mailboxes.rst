.. _mailboxes-java:

信箱
#########

一个Akka ``Mailbox`` 持有目标为一个``Actor`` 的消息。通常每个 ``Actor`` 具有自己的信箱， 但是在 ``BalancingPool`` 的例子下，所有被路由的actor将共享一个单独的信箱实例。

信箱选择
=================

为一个Actor指定一个消息队列类型。
-------------------------------------------

一一通过让actor实现参数化的接口 :class:`RequiresMessageQueue` 来为特定类型的actor指定一种特定类型的消息队列。这是一个例子：

.. includecode:: code/docs/actor/MyBoundedUntypedActor.java#my-bounded-untyped-actor

:class:`RequiresMessageQueue` 接口的类型参数需要在类似如下的配置中映射到一个信箱：

.. includecode:: ../scala/code/docs/dispatcher/DispatcherDocSpec.scala
   :include: bounded-mailbox-config,required-mailbox-config

闲置每次当你创建一个类型为 :class:`MyBoundedUntypedActor` 的actor时，它将会试图获取一个有界信箱。如果这个actor在部署时配置了不同类型的信箱，无论直接配置还是通过一个指定信箱类型的dispatcher来间接配置，那么这种映射会被覆盖。

.. note::

  为actor所创建的信箱的队列类型将同接口所要求的类型进行核对，如果这个队列并没有实现所需的类型，那么actor的创建将会失败。

为一个Dispatcher指定一个消息队列类型
-----------------------------------------------

一个dispatcher也可以指定运行于其上的actor所使用的信箱类型。一个例子是BalancingDispatcher，它要求一个对于多个并发消费者为类型安全的消息队列。这样的要求在dispacher配置小节中如此被阐明：

  my-dispatcher {
    mailbox-requirement = org.example.MyInterface
  }

所给出的要求指定了一个类或接口的名字，然后消息队列实现的超类型会确保此要求被满足。当发生冲突时-例如，如果这个actor需要一个不满足此要求的信箱类型-那么actor的创建将会失败。

如何选择信箱类型
--------------------------------

当一个actor被创建时， :class:`ActorRefProvider` 实现要决定执行actor所用的dispatcher。然后信箱将按照如下顺序被决定:

1. 如果actor的部署配置小节中包含一个``mailbox`` 键，那么这将指定描述所使用的信箱类型的配置段的名字。

2. 如果actor的 ``Props`` 包含一个信箱选择调用 - 也即 ``withMailbox`` - 那么这将指定描述所使用的信箱类型的配置段的名字。

3. 如果dispatcher的配置段包含一个 ``mailbox-type`` 键，那么这一段将被用于配置信箱类型。

4. 如果actor要求一个如上所描述的信箱类型，那么这一要求之中的映射将被用于确定所使用的信箱类型；如果失败，那么dispatcher的要求-如果存在-则会被尝试。

5. 如果dispatcher要求一个如上所描述的信箱类型，那么这一要求之中的映射将被用于确定所使用的信箱类型。

6. 默认的 ``akka.actor.default-mailbox`` 将被使用。

默认信箱
---------------

如上所述，如果信箱未被指定，则默认的信箱将会被使用。默认情形下这将是一个无界信箱，它由 ``java.util.concurrent.ConcurrentLinkedQueue`` 支撑。

``SingleConsumerOnlyUnboundedMailbox`` 是一种更加高效的信箱, 并且它可用作默认的信箱，但是它不能同 BalancingDispatcher 一同使用。

将 ``SingleConsumerOnlyUnboundedMailbox`` 配置为默认的信箱 ::

  akka.actor.default-mailbox {
    mailbox-type = "akka.dispatch.SingleConsumerOnlyUnboundedMailbox"
  }

哪个配置将被传入到信箱配置？
-------------------------------------------------

每个信箱类型由一个继承自 :class:`MailboxType` 的类实现，并且需要两个构造器参数: 一个 :class:`ActorSystem.Settings` 对象以及一个 :class:`Config` 段. 后者的计算方式为：从actor系统的配置中获取所指定的配置段，将其 ``id`` 键覆盖为信箱类型的配置路径，并且向默认信箱配置段中添加一个备用信箱类型。

内建的信箱实现
===============================

Akka 自带了大量的信箱实现：

* UnboundedMailbox
  - 默认信箱

  - 由 ``java.util.concurrent.ConcurrentLinkedQueue`` 支撑

  - 阻塞: 否

  - 有界: 否

  - 配置名称: "unbounded" 或 "akka.dispatch.UnboundedMailbox"

* SingleConsumerOnlyUnboundedMailbox

  - 由一个非常高效的多生产者单消费者队列所支撑, 不能同 BalancingDispatcher 一起使用

  - 阻塞: 否

  - 有界: 否

  - 配置名称: "akka.dispatch.SingleConsumerOnlyUnboundedMailbox"

* BoundedMailbox

  - 由一个 ``java.util.concurrent.LinkedBlockingQueue`` 支撑

  - 阻塞: 是

  - 有界: 是

  - 配置名称: "bounded" 或 "akka.dispatch.BoundedMailbox"

* UnboundedPriorityMailbox

  - 由一个 ``java.util.concurrent.PriorityBlockingQueue`` 支撑

  - 阻塞: 是

  - 有界: 否

  - 配置名称: "akka.dispatch.UnboundedPriorityMailbox"

* BoundedPriorityMailbox

  - 由一个 ``java.util.PriorityBlockingQueue`` 包装于 ``akka.util.BoundedBlockingQueue`` 之中的 ``java.util.PriorityBlockingQueue`` 所支撑

  - 阻塞: 是

  - 有界: 是

  - 配置名称: "akka.dispatch.BoundedPriorityMailbox"

* UnboundedControlAwareMailbox

  - 以更高的优先级投递继承了 ``akka.dispatch.ControlMessage`` 的消息

  - 由两个 ``java.util.concurrent.ConcurrentLinkedQueue`` 所支撑

  - 阻塞: 否

  - 有界: 否

  - 配置名称: "akka.dispatch.UnboundedControlAwareMailbox"

* BoundedControlAwareMailbox

  - 以更高的优先级投递继承了 ``akka.dispatch.ControlMessage``的消息

  - 由两个 ``java.util.concurrent.ConcurrentLinkedQueue`` 所支撑，并且在容量耗尽的情况下入队会阻塞。

  - 阻塞: 是

  - 有界: 是

  - 配置名称: "akka.dispatch.BoundedControlAwareMailbox"

信箱配置示例
==============================

PriorityMailbox
---------------

如何创建一个 PriorityMailbox:

.. includecode:: ../java/code/docs/dispatcher/DispatcherDocTest.java#prio-mailbox

然后将其添加到配置:

.. includecode:: ../scala/code/docs/dispatcher/DispatcherDocSpec.scala#prio-dispatcher-config

然后是一个如何使用它的例子:

.. includecode:: ../java/code/docs/dispatcher/DispatcherDocTest.java#prio-dispatcher

还可以像这样直接配置一个信箱类型:

.. includecode:: ../scala/code/docs/dispatcher/DispatcherDocSpec.scala
   :include: prio-mailbox-config-java,mailbox-deployment-config

然后直接在配置中使用它:

.. includecode:: code/docs/dispatcher/DispatcherDocTest.java#defining-mailbox-in-config

或者在代码中使用它:

.. includecode:: code/docs/dispatcher/DispatcherDocTest.java#defining-mailbox-in-code

ControlAwareMailbox
-------------------

一个 ``ControlAwareMailbox`` 非常有用的情况是：当一个actor需要能够立即接收控制消息，不论它的信箱中已经有多少其他消息。

它可以像这样进行配置:

.. includecode:: ../scala/code/docs/dispatcher/DispatcherDocSpec.scala#control-aware-mailbox-config

控制消息需要继承 ``ControlMessage`` trait:

.. includecode:: ../java/code/docs/dispatcher/DispatcherDocTest.java#control-aware-mailbox-messages

And then an example on how you would use it:

.. includecode:: ../java/code/docs/dispatcher/DispatcherDocTest.java#control-aware-dispatcher

创建你自己的信箱类型
==============================

一例胜千言

.. includecode:: code/docs/dispatcher/MyUnboundedJMailbox.java#mailbox-implementation-example

.. includecode:: code/docs/dispatcher/MyUnboundedJMessageQueueSemantics.java#mailbox-implementation-example

然后你只需在dispatcher配置或信箱配置中将"mailbox-type"指定为你的MailboxType全类名。

.. note::

  确保包含一个具有 ``akka.actor.ActorSystem.Settings`` 和 ``com.typesafe.config.Config`` 参数的构造器, 因为创建你自己的信箱类型时，这个构造器会通过反射而被调用。作为第二个参数传入的config是描述了使用这个信箱类型的dispatcher或信箱设的配置; 信箱类型会为每个使用它的dispatcher或信箱各自创建一次。

你还可以将信箱用作dispatcher上的一个要求，像这样：

.. includecode:: ../scala/code/docs/dispatcher/DispatcherDocSpec.scala#custom-mailbox-config-java

或者通过在你的actor类上定义要求，像这样：

.. includecode:: code/docs/dispatcher/DispatcherDocTest.java#require-mailbox-on-actor


``system.actorOf`` 的特殊语义
=======================================

为了在保持返回值类型为 :class:`ActorRef` （以及所返回的引用功能齐全的语义）的同时，使 ``system.actorOf`` 为同步且非阻塞，在这种情况下进行了特殊处理。在幕后创建了一个空类型的actor引用，它被发送给系统的守卫actor，它实际上会创建actor及其context并将其放入到引用当中。直到这件事发生之前，所有发送到这个 :class:`ActorRef` 的消息将在本地排队，并且仅在换入真正的填充物(actor)时，这些消息才会被传送到真实的信箱。因此，

.. code-block:: scala

   final Props props = ...
   // this actor uses MyCustomMailbox, which is assumed to be a singleton
   system.actorOf(props.withDispatcher("myCustomMailbox").tell("bang", sender);
   assert(MyCustomMailbox.getInstance().getLastEnqueued().equals("bang"));

实际上会失败；你必须允许一些时间来传递并重试这次检查，通过 :meth:`TestKit.awaitCond` 的方式。

