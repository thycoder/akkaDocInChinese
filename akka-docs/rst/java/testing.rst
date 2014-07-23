.. _akka-testkit-java:

##############################
测试actor系统
##############################

如同任何软件一样，自动化测试时开发循环中非常重要的一部分。actor模型体现了对于代码单元如何切分以及它们之间如何交互的一个不同的视角，这影响了如何执行测试。

.. note::

  由于Scala可用的测试DSL的简洁性 (`ScalaTest`_,
  `Specs2`_, `ScalaCheck`_), 即使主工程是用Java编写的，使用Scala来编写单元测试套件也是个好主意。如果那不是你想做的，你也可以通过Java来使用 :class:`TestKit` 及其朋友，尽管它具有更加繁琐的语法，如下所述。 Munish Gupta 曾经 `发布一篇很好的博客
  <http://www.akkaessentials.in/2012/05/using-testkit-with-java.html>`_ 展示了一些你可能有用的模式。

.. _ScalaTest:  http://scalatest.org/
.. _Specs2:     http://specs2.org/
.. _ScalaCheck: http://code.google.com/p/scalacheck/

Akka带有一个专门的模块 :mod:`akka-testkit` 用来在不同级别支持测试，这些级别分为两个清晰区分的类别：

 -测试孤立的代码块而不涉及actor模型， 意味着没有多线程，这暗含了关于事件顺序的完全确定的行为，并且不必关注并发，这在后续内容中会被称作 **单元测试** 。
 - 测试 (多个) 封装的actor，包含多线程调度；这暗含着不确定的事件顺序，但是依靠actor模型避免了对并发的关注，这在后续内容中被称作**集成测试** 。

当然两类测试会有力度的变化，其中单元测试朝白盒测试延伸，而集成测试可以包含对整个actor网络的功能测试。重要的区别在于并发考虑是否为测试的一部分。所提供的工具在后续小节会进行详细的介绍。

.. note::
   确保将 :mod:`akka-testkit` 添加到你的依赖。

使用 :class:`TestActorRef` 进行同步单元测试
===================================================

在 :class:`Actor` 类内测试业务逻辑可以分为两部分：每个原子操作必须隔离地运行，然后是到来的事件顺序必须被正确地处理，甚至当时间顺序可能存在变化的情况下。前者是单线程单元测试的主要使用场景，而后者只能在集成测试中进行验证。

正常情况下， :class:`ActorRef` 将内部的 :class:`Actor` 实例与外界隔离，唯一的通信渠道是actor的信箱。这种限制是单元测试的障碍，这导致了 :class:`TestActorRef` 的诞生。 这种特殊类型的引用转为测试目的而设计，并且允许以两种方式访问actor：要么通过获取一个对底层actor的引用，要么通过调用或查询actor的行为 (:meth:`receive`). 每一种都在下面有单独的小节。

获得对一个 :class:`Actor` 的引用
------------------------------------------

访问实际的 :class:`Actor` 对象允许在所包含的方法上应用所有传统的单元测试技术。获取一个引用的做法如下:

.. includecode:: code/docs/testkit/TestKitDocTest.java#test-actor-ref

由于 :class:`TestActorRef` 在actor类型方面是泛型的，它会以正确的静态类型返回所含的actor。从此往后你可以带着任何单元测试工具，像往常一样测试你的actor。

测试 Actor 的行为
----------------------------
当dispatcher调用actor对一条消息的处理逻辑时，它实际会调用当前actor所注册的当前行为的 :meth:`apply` 。这一开始会得到所声明的 :meth:`receive` 方法的返回值，但是它可以在响应外部消息时通过 :meth:`become` 和 :meth:`unbecome` 进行修改。所有这些都会为整体actor的行为作出贡献，并且它并不让自己在 :class:`Actor` 上易于测试。因此 :class:`TestActorRef` 提供了一种不同模式的操作，来补充 :class:`Actor` 测试: 它支持所有常规 :class:`ActorRef` 所支持的操作。 向这个actor发送的消息同步地在当前线程上进行处理，并且应答可以照常被发回。这个技巧是通过下面所描述的 :class:`CallingThreadDispatcher` (参见 `CallingThreadDispatcher`_)而变得可能的；dispatcher隐式地为任何实例化为一个 :class:`TestActorRef` 的actor创建。

.. includecode:: code/docs/testkit/TestKitDocTest.java#test-behavior

由于 :class:`TestActorRef` 是 :class:`LocalActorRef` 的子类，带有一些特殊的额外功能，而且类似监控和重启等方面能够正确地工作，但是需要当心的是，只有所涉及的所有actor都使用 :class:`CallingThreadDispatcher` 时，代码执行才是严格同步的。你一添加一些包含更高级调度功能的元素，你就离开了单元测试领域，因为那样你就就将再次需要考虑异步。 (在大多数情况下，问题将会是一直等待，直到所需的效果有机会发生。).

另一个对于单线程测试进行了覆盖的特殊方面是 :meth:`receiveTimeout`, 因为包含此功能会要求将 :obj:`ReceiveTimeout` 消息进行异步排队，违反了同步契约。

.. note::

   总结: :class:`TestActorRef` 覆盖了两个字段: 它将 dispatcher 设置为 :obj:`CallingThreadDispatcher.global` 并且将 :obj:`receiveTimeout` 设置为None.

中间道路: 期望异常
----------------------------------------

如果你想测试actor行为，包括热交换，但是不涉及dispatcher，并且不让 :class:`TestActorRef` 吞掉任何抛出的异常，那么你可以使用另一种模式：只需要在 :class:`TestActorRef` 上使用 :meth:`receive` ，这会被转发到内含的actor:

.. includecode:: code/docs/testkit/TestKitDocTest.java#test-expecting-exceptions

使用场景
---------

当然你可以混合并搭配 :class:`TestActorRef` 的两种 modi operandi ，满足你的测试需要:
 - 一种常见的使用场景是在发送测试消息之前将actor设置为一个特定的内部状态
 - 另一种是在发送测试消息后验证内部状态转换的正确性

尽管试验各种可能性，并且如果你找到了有用的模式，请毫不犹豫地让Akka论坛知道它们！谁知道呢，通用的操作甚至有可能被制作成精美的DSL。

使用 :class:`JavaTestKit` 进行异步集成测试
==========================================================

当你有理由地确认actor的业务逻辑正确之后，下一步是验证它在目标环境中运行正确。对环境的定义当然取决于手头的问题，以及你想要的测试级别。最小的配置包含这样的测试过程，它提供所需的激励，被测试的actor，以及一个用于接收回复的actor。更大的系统将待测试的actor替换为一个actor网络，在各种不同的注入点应用激励，并且安排不同的消息发送点将结果发送，但是基本原则是相同的，就是一个单独的过程驱动整个测试。

:class:`JavaTestKit` 类包含一个工具集合，使得通用任务变得简单。

.. includecode:: code/docs/testkit/TestKitSampleTest.java#fullsample

:class:`JavaTestKit` 包含一个actor，名为 :obj:`testActor` ，它是消息被检查的入口，检查方法为各式各样的 ``expectMsg...`` 断言，下面详述。这个testActor的引用要使用上述的 :meth:`getRef()` 方法来获取。 :obj:`testActor` 也可以照常被发送给别的actor，通常将其订阅为一个通知监听器。有一整套完整的检查方法，例如，接收所有连续的满足特定标准的消息，接收一系列固定消息或类，在一定时间内不接受任何消息，等等。

传入到JavaTestKit的构造器中的ActorSystem可通过 :meth:`getSystem()` 访问.

.. note::

  记得在测试完成之后关闭actor系统（失败的情况下也关闭），从而使得所有actor-包括测试actor，都停止。

内建的断言
-------------------

上面提到的 :meth:`expectMsgEquals` 不是用来表述关于接收消息的断言的唯一方法，完整的方法集合如下：

.. includecode:: code/docs/testkit/TestKitDocTest.java#test-expect

在这些例子中，下面提到的最大间隔被漏掉了，在这种情况下会使用配置项 ``akka.test.single-expect-default`` 的值作为默认值，而它自身默认为3秒 (否则会遵循最内层嵌套的 :class:`Within` ，如 :ref:`下
<JavaTestKit.within>` 所述). 完整的签名是：:

  * :meth:`public <T> T expectMsgEquals(Duration max, T msg)`

	给定的消息必须在给定的时间之内被接收；这个对象会被返回。

  * :meth:`public Object expectMsgAnyOf(Duration max, Object... msg)`

	一个对象必须在给定的时间内被接收，并且它必须至少等于(使用  ``equals()`` 进行比较)所传入的引用对象之一; 所接收的对象会被返回。

  * :meth:`public Object[] expectMsgAllOf(Duration max, Object... msg)`

	给定时间内匹配所提供的对象数组尺寸的一些对象必须被返回，并且对于每个给定的对象，接收到的对象中必须至少有一个等于它（使用 ``equals()`` 比较). 接收到的对象的完整序列会按照接收顺序返回。

  * :meth:`public <T> T expectMsgClass(Duration max, Class<T> c)`
	一个给定 :class:`Class` 的对象必须在所分配的时间帧之内被返回；这个对象将作为返回值。注意这会进行一次一致性检查，如果你需要这个类是相等的（译者注：而不是继承关系），你需要后续进行验证。

  * :meth:`public <T> T expectMsgAnyClassOf(Duration max, Class<? extends T>... c)`
	一个对象必须在给定时间内被接收，并且它必须是所提供的 :class:`Class` 中某一个类的对象; 所接收的对象将被返回。注意这会进行一次一致性检查，如果你需要这个类是相等的（译者注：而不是继承关系），你需要后续进行验证。

    .. note::
	  因为Java的类型系统的限制，可能有必要在使用这个方法时添加 ``@SuppressWarnings("unchecked")`` 。

  * :meth:`public void expectNoMsg(Duration max)`
	在给定时间内必须没有任何消息被接收。当一个消息在调用这一方法之前被接收，并且没有使用其他方法之一将其从队列中删除时，这个断言也会失败。

  * :meth:`Object[] receiveN(int n, Duration max)`

    ``n`` 条消息必须在给定时间内被接收；接收到的消息将被返回。

对于要求更多细化条件的情况，可以使用传入代码块的一些结构：

  * **ExpectMsg<T>**

    .. includecode:: code/docs/testkit/TestKitDocTest.java#test-expectmsg

    :meth:`match(Object in)` 方法会在一个消息于给定时间范围（可以作为一个构造器参数而给定）之内返回时立即被计算。如果它抛出 ``noMatch()``（ 在此处本来调用那个方法就已足够（译者：如果使用Scala而不是Java）； ``throw`` 关键字只在如果不用它则编译器会抱怨错误返回值类型时才是有用的-因为Java缺乏Scala中表示“不会正常返回”的类型概念），那么这个预期会失败并抛出一个 :class:`AssertionError`, 否则匹配的而且可能被转换的对象会被存储，用于被 :meth:`get()` 方法获取。

  * **ReceiveWhile<T>**

    .. includecode:: code/docs/testkit/TestKitDocTest.java#test-receivewhile
	
这个结构工作方式类似于 ExpectMsg, 但是它不断收集复合这个标准的消息，并且它在遇到不匹配的消息时不会失败。收集消息在时间到了的情况下也会结束，当消息之间间隔太长时间，或者当已经取到足够消息时。

    .. includecode:: code/docs/testkit/TestKitDocTest.java#test-receivewhile-full
       :exclude: match-elided

之所以需要两次指定 ``String`` 结果类型，是由于需要创建一个类型正确的数组，而Java不能推断（数组）这个类的类型参数。、

  * **AwaitCond**

    .. includecode:: code/docs/testkit/TestKitDocTest.java#test-awaitCond
	
这个通用结构与测试工具的消息接收没有关系，嵌入的条件能够根据作用域之内的任何东西计算一个布尔结果值。

  * **AwaitAssert**

	.. includecode:: code/docs/testkit/TestKitDocTest.java#test-awaitAssert
	
这个通用结构与测试工具的消息接收没有关系，嵌入的断言能够检查作用域之内的任何东西。

还有一些情形，在其中并不是所有发送到测试工具的消息都与测试相关，但是移除它们就意味着在测试时修改actor。出于这种目的，可以忽略特定的消息：

  * **IgnoreMsg**

    .. includecode:: code/docs/testkit/TestKitDocTest.java#test-ignoreMsg

期望日志消息
----------------------

由于集成测试不允许调用参与测试的actor的内部处理逻辑，因此验证期待异常的测试不能直接进行。相反，可将日志系统用于这一目的：将常规的事件处理器替换为 :class:`TestEventListener` 并使用一个 :class:`EventFilter` 就能够断言日志消息，包括那些由异常所产生的日志：

.. includecode:: code/docs/testkit/TestKitDocTest.java#test-event-filter

如果发生次数是制定的-如上所示-那么 ``exec()`` 会一直阻塞，直到匹配数目的消息已经收到，或者 ``akka.test.filter-leeway`` 中配置的超时时间耗尽 (时间在 ``run()`` 方法返回之后开始计算). 在超时的情况下测试会失败。

.. note::
   确保在你的 ``application.conf`` 中将默认的日志处理器替换为 :class:`TestEventListener` 来启用这一功能::

     akka.loggers = [akka.testkit.TestEventListener]

.. _JavaTestKit.within:

时间断言
-----------------

功能测试的一个重要部分涉及时间：特定的事情必须不能立即发生（类似于计数器），另一些需要在截止时间之前发生。 因此，所有检查方法都接收一个时间上限，在此之前必须获取一个正面或负面的结果。时间下限的检查需要在这一检查之外进行，这能够被一个用于管理时间约束的新结构来加快:

.. includecode:: code/docs/testkit/TestKitDocTest.java#test-within

 :meth:`Within.run()` 内的代码块必须在 :obj:`min` 和 :obj:`max` 之间的一个 :ref:`Duration` 内完成, 而前者默认为0.截止时间是通过将 :obj:`max` 参数加到代码块的开始时间上而算出的，它在代码块中对所有的检查方法都隐式可用，如果你不指定它，则它继承自最内层的 :meth:`within` 代码块。

应该注意的一点是，如果这个代码块的最后一条由消息驱动的断言是 :meth:`expectNoMsg` 或 :meth:`receiveWhile`, 则对 :meth:`within` 的最终检查会被跳过，从而避免由于唤醒延迟而导致的错误的测试通过。这意味着虽然其中每一个独立的断言仍然使用时间上限，整个代码块在这种情况下会有长度随机的延迟。

.. note::
   所有时间都使用 ``System.nanoTime`` 来度量, 这意味着它们描述的是挂钟时间(wall time)，而不是CPU时间或系统时间。

考虑缓慢的测试系统
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在你的闪电般迅速的笔记本上测试时配置的紧张的超时时间，总是会导致高负载的Jenkins服务器（或类似产品）上出现伪失败。为了照顾这种情况，所有最大间隔在内部会乘以一个来自 :ref:`configuration` 中的 ``akka.test.timefactor`` 因子, 默认为1 1.

你可以通过使用 :class:`JavaTestKit` 中的 ``dilated`` 方法，将同样的因子应用到其他的时间间隔上。

.. includecode:: code/docs/testkit/TestKitDocTest.java#duration-dilation

使用多个探针Actor
---------------------------

当所测试的actor被要求发送多个消息到不同目的地时，按照此前所示，使用 :class:`JavaTestKit` 时可能难以区分抵达 :obj:`testActor` 的消息。另一种方式是使用它创建简单的探针actor，用于插入到消息流中。这一功能使用一个小例子来解释是最好的：

.. includecode:: code/docs/testkit/TestKitDocTest.java#test-probe

这个简单的测试验证了一个同样简单的Forwarder actor，通过将一个探针插入为forwarder的目标。另一个例子是两个actor A和B 通过A往B发送消息来进行协作。为了验证这个消息流，一个 :class:`TestProbe` 可被插入为A的目标，使用转发功能或下面所述的自动操控，在测试配置中包含一个真实的B。

探针还可以配备自定义断言，从而让你的测试代码更加简洁和清楚：

.. includecode:: code/docs/testkit/TestKitDocTest.java
   :include: test-special-probe

你在此处具有完全的灵活性，能够混合和匹配 :class:`JavaTestKit` 工具与你自己的检查，并且选择一个直观的名字。现实中，你的代码可能比上面所给的例子复杂得多；尽管使用这种能力！

.. warning::
  
  任何从 ``TestProbe`` 发送到另一个运行于CallingThreadDispatcher纸上的actor的消息都有死锁的风险，如果另一个actor也可以发送消息到探针的话。 :meth:`TestProbe.watch` 和 :meth:`TestProbe.unwatch` 的实现也会将一个消息发送到watchee，这意味着试图从一个 :meth:`TestProbe` 中监控一个  :class:`TestActorRef` 是危险的。

从探针监控其他Actor
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

一个 :class:`JavaTestKit` 可以将自己注册为任意其他actor的DeathWatch:

.. includecode:: code/docs/testkit/TestKitDocTest.java
   :include: test-probe-watch

回复探针所接收的消息
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

探针存储了最后一条出队消息的发送者（也就是，在接收到 ``expectMsg*`` 之后),它可使用 :meth:`getLastSender()` 方法来获取。这一信息还可以隐式地被用于让探针回复最后一条收到的消息：

.. includecode:: code/docs/testkit/TestKitDocTest.java#test-probe-reply

转发探针所接收的消息
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

探针还可以转发所接收的消息（也就是，在它接收 ``expectMsg*`` 之后), 保留原始的发送者:

.. includecode:: code/docs/testkit/TestKitDocTest.java#test-probe-forward

自动操控
^^^^^^^^^^
将消息接收到一个队列中并稍后检查是很好的，但是为了保持一个测试运行，并且稍后验证其痕迹，你还可以安装一个 :class:`AutoPilot` 到参加测试的探针里(实际上是在任何 :class:`TestKit` 中) ，它会在所观察的队列中有消息入队时被调用。这个代码可用于转发消息，例如，按照一个链条 ``A --> Probe -->
B`` , 只要遵循一种特定的协议。

.. includecode:: code/docs/testkit/TestKitDocTest.java#test-auto-pilot

:meth:`run` 方法必须返回这个 auto-pilot 作为下一条消息，包装在一个 :class:`Option` 中; 将其设置为 :obj:`None` 会结束自动操控.

关于时间断言的警告
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

:meth:`within` 代码块在使用测试探针时，可能会被认为是反直觉的：你需要记住， :ref:`上面 <JavaTestKit.within>` 所描述这个作用域定义良好的截止时间是局部于每个探针的。因此，探针不能不会响应彼此的截止时间，或者外围 :class:`JavaTestKit` 实例的截止时间:

.. includecode:: code/docs/testkit/TestKitDocTest.java#test-within-probe

这里 ``expectMsgEquals`` 调用会使用默认的超时时间。

.. _Java-CallingThreadDispatcher:

CallingThreadDispatcher
=======================

:class:`CallingThreadDispatcher` 在单元测试中具有很好的用途，如上所述，但是最初构思它，是为了支持在错误的时候生成连续的stacktrace。由于这个特殊的dispatcher在当前线程上运行所有通常会直接入队的东光县，消息处理链的完整历史就可以被记录在调用堆栈上，只要所有中间actor都运行在这个dispatcher之上。

如何使用它
-------------

只需按照正常的方式设置dispatcher

.. includecode:: code/docs/testkit/TestKitDocTest.java#calling-thread-dispatcher

它是如何工作的
------------

当接受到一次调用时， :class:`CallingThreadDispatcher` 检查接收actor是否已经活跃在当前线程上。这种情况的最简单例子是，一个actor向自己发送消息。在这种情况下，处理不能立即继续，因为那会违反actor模型，因此这个调用会被排入队列，并且将在那个actor的活跃调用被执行完之后再进行处理；因此，它会在调用线程上处理，但是仅在actor完成之前工作后。在另一种情况下，调用简单地在当前线程上立即处理。通过这个dispatcher所调度的Future也会立即执行。

这种方案使得 :class:`CallingThreadDispatcher` 的工作方式类似于一个用于任意actor的通用dispatcher，它从来不会在外部事件上阻塞。

当存在多条线程时，可能会发生运行于这个dispatcher上的一个actor的两次调用同时发生于不同线程的情况。在这种情况下，两者都在相应的线程上被直接处理，其中两者会竞争actor的锁，并且输者必须等待。这样，actor模型保持完好，但是代价是由于受限的调度而损失并发性。在某种意义上，这等价于传统的互斥风格的并发。

另一个余下的难点，在于对挂起和继续的正确处理：当actor挂起时，后续的调用会被排队到线程安全的队列中（对于常规的情况，也使用相同的队列）。然而，对 :meth:`resume` 的调用，是由一个特定的线程完成的，并且系统中所有其他的线程很可能都不在执行指定的actor, 这会导致线程安全的队列不能被它们的本地线程清空。因此，调用 :meth:`resume`  的线程会将所有当前已入队的来自所有线程的调用收集到自己的队列，并且处理它们。

局限
-----------

.. warning::
   在CallingThreadDispatcher 被用于顶层actor但不检查TestActorRef的情况下,就存在一个时间窗口，在其中actor等待被user守卫actor创建。 在这一时间向actor发送消息会导致它们进入队列，然后在守卫actor的线程而不是调用者线程上执行。为了避免这种情况，请使用TestActorRef.

如果actor的行为在某件事情上阻塞，而这件事情通常会在调用者actor发送这条消息后才会被影响，使用这个dispatcher时，这很明显会导致死锁。在基于 :class:`CountDownLatch` 进行同步的actor测试中，这是很常见的场景:

.. code-block:: scala

   val latch = new CountDownLatch(1)
   actor ! startWorkAfter(latch)   // actor will call latch.await() before proceeding
   doSomeSetupStuff()
   latch.countDown()

这个例子会在第二行所发起的消息处理中无限地阻塞，永远不会到达第四行，而第四行在常规dispatcher中会解锁。

因此，要牢记， :class:`CallingThreadDispatcher` 不是常规dispatcher的通用替代品. 另一方面，它非常有用的情况是，将你的actor网络运行在其上而进行测试，因为如果它能够不带死锁地运行，那么它在生产环境下不会死锁的概率很高。

.. warning::

   上述的命题可惜不是个坚固的保证，因为当你的代码运行于不同的dispatcher上时，可能会直接或间接地修改它的行为。如果你在寻找一个工具来帮助你调试死锁，  :class:`CallingThreadDispatcher` 可能有助于特定的错误场景，但是要牢记，它可能会给出错误的测试不通过结果，也会给出错误的测试通过结果。
   false positives.

线程中断
--------------------

如果 CallingThreadDispatcher 看到当前线程在消息处理返回时自己的 ``isInterrupted()`` 标记被设置，它会在完成所有的处理之后抛出一个 :class:`InterruptedException` (也就是，如上所述，所有需要处理的消息在这之前将会被处理). 因为 :meth:`tell` 由于它的方法契约而不能抛出异常，这个异常会被捕获并记录，但是线程的interrupted状态会再次被设置。

如果在消息处理过程中一个 :class:`InterruptedException` 被抛出，那么它会在 CallingThreadDispatcher  的消息处理循环中被捕获，线程的interrupted 标记将会被设置，并且处理会继续照常进行。

.. note::
  
  这两段的总结是，如果当前线程在 CallingThreadDispatcher 下工作时被中断，那么着会导致消息发送返回时 ``isInterrupted`` 标记被设置为 ``true`` ，并且 :class:`InterruptedException` 不会被抛出。

好处
--------

作为总结, 下面是 :class:`CallingThreadDispatcher` 必须提供的特性：

 - 单线程测试的确定性执行，同时几乎保持完整的actor语义
 - 在异常对战踪迹中包含完整的消息处理历史，直到错误发送的点。
 - 排除特定类型的思索场景

.. _actor.logging-java:

跟踪Actor调用
=========================

到此为止所描述的测试工具的目的是清晰表达关于系统行为的断言。如果测试失败，找到原因，修复错误，并重新验证测试，通常是你的责任。这个过程由调试器和日志来支持，其中Akka工具箱提供了如下选项：

* *在Actor实例内部记录所抛出的异常s*
  
  这总是打开的；与其他日志机制不同，这种日志在 ``ERROR`` 级别.

* *记录特殊消息*
  
  Actor自动处理特定的特殊消息，例如 :obj:`Kill`, :obj:`PoisonPill`, 等等. 启用跟踪这些消息调用的功能，是通过设置项 ``akka.actor.debug.autoreceive`` 来控制的, 这会在所有actor上启用跟踪。

* *记录actor生命周期*
  
  Actor创建，启动，重启，监控启动，监控停止，以及停止，可以通过启用配置项 ``akka.actor.debug.lifecycle`` 来进行跟踪; 同样地，这也会在所有actor上一致地启用。

所有这些消息都以 ``DEBUG`` 级别记录. 作为概括, 你可以使用这个配置片段来启用actor活动的完全日志::

  akka {
    loglevel = "DEBUG"
    actor {
      debug {
        autoreceive = on
        lifecycle = on
      }
    }
  }

Configuration
=============

TestKit模块有一些配置属性，请参考： :ref:`reference configuration <config-akka-testkit>`.

